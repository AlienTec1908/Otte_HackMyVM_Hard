# Otte - HackMyVM (Hard)

![Otte.png](Otte.png)

## Übersicht

*   **VM:** Otte
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Otte)
*   **Schwierigkeit:** Hard
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 17. Oktober 2022
*   **Original-Writeup:** https://alientec1908.github.io/Otte_HackMyVM_Hard/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Otte" zu erlangen. Der Weg dorthin führte über mehrere Eskalationsstufen: Zuerst wurde eine Local File Inclusion (LFI)-Schwachstelle in einem PHP-Skript (`thinkgeek.php`) entdeckt, das über HTTP Basic Authentication (`root:zP2wxY4uE`) zugänglich war. Diese LFI ermöglichte das Auslesen einer `shell.php` im Home-Verzeichnis des Benutzers `thomas`, welche eine RCE über einen `command`-Parameter bot. Dies führte zu einer Reverse Shell als `www-data`. Die nächste Eskalation zu `thomas` gelang durch das Auslesen eines Passworts (`youareonthegoodwaybro`) aus einem QR-Code, der in einer Bilddatei versteckt war (Steganographie). Als `thomas` konnte mittels einer unsicheren `sudo`-Regel (`sudo -u laetitia python3 /home/laetitia/simpler.py -p`) durch Command Injection eine Shell als `laetitia` erlangt werden. Eine weitere `sudo`-Regel erlaubte `laetitia`, `w3m` als `cedric` auszuführen, was über einen Shell-Escape innerhalb von `w3m` zu einer Shell als `cedric` führte. Schließlich wurde (implizit) als `cedric` der private SSH-Schlüssel des `root`-Benutzers gefunden, was den direkten Root-Login per SSH ermöglichte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `lftp`
*   `hydra` (versucht)
*   `wfuzz`
*   `nc` (netcat)
*   `stegseek` (impliziert für QR-Code)
*   `steghide` (impliziert für QR-Code)
*   `ssh`
*   `curl`
*   `wget`
*   Python3
*   `grep`
*   `cp`
*   `echo`
*   `w3m`
*   CyberChef (extern, für QR-Code)
*   Standard Linux-Befehle (`vi`/`nano`, `ls`, `cat`, `cd`, `sudo`, `bash`, `find`, `id`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Otte" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web/FTP Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.107) mit `arp-scan` identifiziert.
    *   `nmap`-Scan offenbarte Port 21 (FTP, ProFTPD, anonymer Login erfolgreich), 22 (SSH, OpenSSH 7.9p1) und 80 (HTTP, Apache 2.4.38).
    *   Anonymer FTP-Login fand `note.txt` mit Hinweis auf Benutzer `thomas` und PHP-Code in dessen Home-Verzeichnis.
    *   `gobuster` auf Port 80 fand u.a. `thinkgeek.php`. Der Webserver erforderte HTTP Basic Authentication (`root:zP2wxY4uE` - Herkunft dieser Credentials unklar im Log, aber für Zugriff genutzt).

2.  **Initial Access (LFI zu RCE als `www-data`):**
    *   Mittels `wfuzz` wurde der Parameter `file` für `thinkgeek.php` gefunden, der anfällig für Local File Inclusion (LFI) war (`thinkgeek.php?file=../../../../../etc/passwd`).
    *   Über LFI wurde die Existenz von `/home/thomas/shell.php` bestätigt.
    *   Mittels `wfuzz` wurde der Parameter `command` für die (via LFI eingebundene) `shell.php` gefunden.
    *   Eine RCE wurde durch Senden eines Bash-Reverse-Shell-Payloads an `thinkgeek.php?file=../../../../home/thomas/shell.php&command=[PAYLOAD]` erreicht.
    *   Eine Reverse Shell als `www-data` wurde auf einem Netcat-Listener (Port 9001) empfangen.

3.  **Privilege Escalation (von `www-data` zu `thomas` via QR/Stego & SSH):**
    *   Analyse einer (nicht spezifizierten) PNG-Datei mittels Hexdump und CyberChef enthüllte einen QR-Code.
    *   Der QR-Code enthielt die Information `thomas:youareonthegoodwaybro`.
    *   Erfolgreicher SSH-Login als `thomas` mit dem Passwort `youareonthegoodwaybro`.

4.  **Privilege Escalation (von `thomas` zu `laetitia` via `sudo python`):**
    *   `sudo -l` als `thomas` zeigte, dass `/usr/bin/python3 /home/laetitia/simpler.py -p` als `laetitia` ohne Passwort ausgeführt werden durfte.
    *   Das Python-Skript `simpler.py` forderte zur Eingabe einer IP auf und war anfällig für Command Injection.
    *   Durch Eingabe von `$('/bin/bash')` als "IP" wurde eine Shell als `laetitia` erlangt.

5.  **Privilege Escalation (von `laetitia` zu `cedric` via `sudo w3m`):**
    *   `sudo -l` als `laetitia` zeigte, dass `/usr/bin/w3m *` als `cedric` ohne Passwort ausgeführt werden durfte.
    *   `sudo -u cedric /usr/bin/w3m https://google.de` wurde ausgeführt. Innerhalb von `w3m` wurde durch Eingabe von `!/bin/bash` eine Shell als `cedric` gestartet.

6.  **Privilege Escalation (von `cedric` zu `root` via SSH Key Leak):**
    *   Als `cedric` wurde (implizit im Log) der private SSH-Schlüssel des `root`-Benutzers gefunden.
    *   Der Schlüssel wurde gespeichert (`idtt`), die Berechtigungen gesetzt (`chmod 600`).
    *   Erfolgreicher SSH-Login als `root` mit dem privaten Schlüssel (`ssh root@otte.hmv -i idtt`).
    *   Die User-Flag (`HMViwazhere`, Pfad impliziert `/home/gabriel/user.txt` - *Hinweis: Benutzer Gabriel taucht hier erstmals auf*) und Root-Flag (`HMVohmygod` in `/root/root.txt`) wurden gefunden.

## Wichtige Schwachstellen und Konzepte

*   **HTTP Basic Authentication:** Zugriff auf Web-Ressourcen war durch Basic Auth geschützt (Credentials `root:zP2wxY4uE` im Bericht verwendet).
*   **Local File Inclusion (LFI):** `thinkgeek.php` war anfällig für LFI, was das Einbinden und Ausführen einer PHP-Shell aus einem Benutzerverzeichnis ermöglichte.
*   **Remote Code Execution (RCE):** Die via LFI eingebundene `shell.php` erlaubte RCE über einen GET-Parameter.
*   **Steganographie (QR-Code in Bild):** Ein Passwort für den Benutzer `thomas` war in einem QR-Code versteckt, der aus einem Bild extrahiert wurde.
*   **Unsichere `sudo`-Regeln:**
    *   `thomas` durfte ein Python-Skript (`simpler.py`) als `laetitia` ausführen, welches anfällig für Command Injection war.
    *   `laetitia` durfte `w3m` als `cedric` ausführen, was Shell-Escapes ermöglichte.
*   **Preisgabe eines privaten Root-SSH-Schlüssels:** Der private SSH-Schlüssel von `root` war für den Benutzer `cedric` zugänglich.
*   **Anonymer FTP-Zugriff:** Ermöglichte das Herunterladen einer Hinweisdatei.

## Flags

*   **User Flag (`/home/gabriel/user.txt` - *Benutzer Gabriel unklar aus Flow*):** `HMViwazhere`
*   **Root Flag (`/root/root.txt`):** `HMVohmygod`

## Tags

`HackMyVM`, `Otte`, `Hard`, `LFI`, `RCE`, `Steganography`, `QR Code`, `sudo Exploit`, `Python Command Injection`, `w3m Shell Escape`, `SSH Key Leak`, `Linux`, `Web`, `Privilege Escalation`, `Apache`, `ProFTPD`
