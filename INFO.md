# Moodle lokal mit Docker und WSL starten

Diese Anleitung beschreibt die lokale Entwicklungsumgebung unter Windows mit
Docker Desktop, WSL und dem Moodle-Checkout im Nachbarordner `../moodle`.

## Voraussetzungen

- Docker Desktop ist gestartet.
- Die WSL-Integration für die verwendete Linux-Distribution ist in Docker
  Desktop aktiviert.
- Die Befehle in dieser Anleitung werden in einem WSL-Terminal (zum Beispiel
  Ubuntu) ausgeführt, nicht in PowerShell.

## 1. WSL öffnen und ins Repository wechseln

WSL lässt sich über das Startmenü oder in PowerShell mit `wsl` öffnen. Die
Windows-Dateien sind in WSL unter `/mnt/c/` verfügbar.

```bash
cd /mnt/c/Users/user/MoodleTest/moodle-docker
```

Der Moodle-Checkout befindet sich daneben unter
`/mnt/c/Users/user/MoodleTest/moodle` und verwendet den Branch
`MOODLE_503_STABLE`.

Falls Moodle noch nicht geklont wurde, kann es aus dem `moodle-docker`-Ordner
so angelegt werden:

```bash
git clone -b MOODLE_503_STABLE https://github.com/moodle/moodle.git ../moodle
```

## 2. Docker-Umgebung konfigurieren

Diese Umgebungsvariablen müssen in jeder neuen WSL-Terminalsitzung gesetzt
werden:

```bash
export MOODLE_DOCKER_WWWROOT="$PWD/../moodle"
export MOODLE_DOCKER_DB=pgsql
```

Die Docker-Konfiguration für Moodle muss einmalig in den Checkout kopiert
werden:

```bash
cp config.docker-template.php "$MOODLE_DOCKER_WWWROOT/config.php"
```

Wenn `config.php` dort bereits vorhanden ist, muss der Kopiervorgang nicht
wiederholt werden.

## 3. Container starten

`bin/moodle-docker-compose` ist der Projekt-Wrapper, der die benötigten Compose-
Dateien zusammenstellt. Aus dem `moodle-docker`-Verzeichnis starten:

```bash
bin/moodle-docker-compose up -d
bin/moodle-docker-wait-for-db
```

Falls beim Aufruf der Fehler
`/usr/bin/env: ‘bash\r’: No such file or directory` erscheint, haben die
Skripte Windows-Zeilenenden. Einmalig in WSL konvertieren und den Start
wiederholen:

```bash
sed -i 's/\r$//' bin/moodle-docker-compose bin/moodle-docker-wait-for-db
bin/moodle-docker-compose up -d
```

## 4. Moodle-Datenbank installieren

Die Datenbank muss nur bei einer frischen Moodle-Installation initialisiert
werden. Der lokale Admin-Benutzer wird dabei als `admin` mit dem Passwort
`test` eingerichtet:

```bash
bin/moodle-docker-compose exec webserver php admin/cli/install_database.php \
  --agree-license \
  --fullname="Docker Moodle" \
  --shortname="docker_moodle" \
  --adminpass="test" \
  --adminemail="admin@example.com"
```

Danach ist Moodle unter <http://localhost:8000> erreichbar. Anmeldung:

- Benutzername: `admin`
- Passwort: `test`

## 5. Hello-World-Plugin installieren und öffnen

Das Beispielplugin liegt im Moodle-Checkout unter
`public/local/helloworld`. Moodle 5.3 verwendet `public` als Webroot.

Als Admin die Seite <http://localhost:8000/admin/index.php> öffnen und das
angezeigte Plugin-Upgrade ausführen. Anschließend ist das Plugin unter
<http://localhost:8000/local/helloworld/> erreichbar. Ein Neustart der
Container ist dafür nicht erforderlich.

## Moodle-Cron ausführen

Moodle empfiehlt, den Cronjob jede Minute auszuführen. Wenn im Admin-Bereich
die Warnung erscheint, dass `admin/cron.php` noch nie gelaufen ist, kann Cron
einmalig in WSL aus dem `moodle-docker`-Verzeichnis gestartet werden:

```bash
cd /mnt/c/Users/bewor/MoodleTest/moodle-docker
export MOODLE_DOCKER_WWWROOT="$PWD/../moodle"
export MOODLE_DOCKER_DB=pgsql

bin/moodle-docker-compose exec -T webserver php admin/cli/cron.php
```

Für wiederholte Ausführung alle 60 Sekunden kann derselbe Aufruf in einer
Schleife laufen. Dazu ein zweites WSL-Terminal öffnen und dort Folgendes
ausführen; das Terminal muss dafür geöffnet bleiben:

```bash
cd /mnt/c/Users/bewor/MoodleTest/moodle-docker
export MOODLE_DOCKER_WWWROOT="$PWD/../moodle"
export MOODLE_DOCKER_DB=pgsql

while true; do
  bin/moodle-docker-compose exec -T webserver php admin/cli/cron.php
  sleep 60
done
```

Die Schleife endet, wenn das Terminal geschlossen oder mit `Ctrl+C` abgebrochen
wird. Für eine lokale Testumgebung kann Cron alternativ bei Bedarf manuell
gestartet werden.

## Container stoppen und wieder starten

Zum Stoppen, ohne die Container zu entfernen:

```bash
bin/moodle-docker-compose stop
```

Später wieder starten:

```bash
bin/moodle-docker-compose start
```

Zum Stoppen und Entfernen der Container:

```bash
bin/moodle-docker-compose down
```

Beim erneuten Öffnen eines Terminals zuerst wieder ins `moodle-docker`-
Verzeichnis wechseln und die beiden Umgebungsvariablen aus Schritt 2 setzen.

## Hinweis zur Geschwindigkeit

Derzeit liegen beide Repositories auf dem Windows-Dateisystem unter `/mnt/c`.
Dateizugriffe über diesen Pfad können Moodle in WSL und Docker deutlich
verlangsamen. Wenn das später stört, können die Repositories ins Linux-
Dateisystem von WSL verschoben oder dort neu geklont werden, zum Beispiel nach
`~/projects/`.

Die Datei `.gitpod.yml` betrifft Gitpod und ist für den lokalen WSL-Ablauf nicht
erforderlich.
