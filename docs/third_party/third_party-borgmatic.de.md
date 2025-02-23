# Borgmatic Backup

## Einführung

Borgmatic ist ein großartiger Weg, um Backups auf Ihrem Mailcow-Setup durchzuführen, da es Ihre Daten sicher verschlüsselt und extrem einfach zu
einzurichten.

Aufgrund seiner Deduplizierungsfähigkeiten können Sie eine große Anzahl von Backups speichern, ohne große Mengen an Speicherplatz zu verschwenden.
Speicherplatz zu verschwenden. So können Sie Backups in sehr kurzen Abständen durchführen, um einen minimalen Datenverlust zu gewährleisten, wenn die Notwendigkeit besteht
Daten aus einer Sicherung wiederherzustellen.

Dieses Dokument führt Sie durch den Prozess zur Aktivierung kontinuierlicher Backups für mailcow mit borgmatic. Die borgmatic
Funktionalität wird durch das [borgmatic Docker image von b3vis](https://github.com/b3vis/docker-borgmatic) bereitgestellt. Schauen Sie sich
die `README` in diesem Repository, um mehr über die anderen Optionen (wie z.B. Push-Benachrichtigungen) zu erfahren, die verfügbar sind.
Diese Anleitung behandelt nur die Grundlagen.

## Einrichten von borgmatic

### Erstellen oder ändern Sie `docker-compose.override.yml`

Im mailcow-dockerized Stammverzeichnis erstellen oder bearbeiten Sie `docker-compose.override.yml` und fügen Sie die folgende
Konfiguration ein:
```yaml
version: '2.1'

services:
  borgmatic-mailcow:
    image: b3vis/borgmatic
    hostname: mailcow
    restart: always
    dns: ${IPV4_NETWORK:-172.22.1}.254
    volumes:
      - vmail-vol-1:/mnt/source/vmail:ro
      - crypt-vol-1:/mnt/source/crypt:ro
      - redis-vol-1:/mnt/source/redis:ro,z
      - rspamd-vol-1:/mnt/source/rspamd:ro,z
      - postfix-vol-1:/mnt/source/postfix:ro,z
      - mysql-socket-vol-1:/var/run/mysqld/:z
      - borg-config-vol-1:/root/.config/borg:Z
      - borg-cache-vol-1:/root/.cache/borg:Z
      - ./data/conf/borgmatic/etc:/etc/borgmatic.d:Z
      - ./data/conf/borgmatic/ssh:/root/.ssh:Z
    environment:
      - TZ=${TZ}
      - BORG_PASSPHRASE=YouBetterPutSomethingRealGoodHere
    networks:
      mailcow-network:
        aliases:
          - borgmatic

volumes:
  borg-cache-vol-1:
  borg-config-vol-1:
```

Stellen Sie sicher, dass Sie die `BORG_PASSPHRASE` in eine sichere Passphrase Ihrer Wahl ändern.

Aus Sicherheitsgründen mounten wir das maildir als schreibgeschützt. Wenn Sie später Daten wiederherstellen wollen, müssen Sie das
müssen Sie das `ro`-Flag entfernen, bevor Sie die Daten wiederherstellen. Dies wird im Abschnitt über die Wiederherstellung von Backups beschrieben.

### Erstellen Sie `data/conf/borgmatic/etc/config.yaml`

Als nächstes müssen wir die borgmatic-Konfiguration erstellen.

```shell
source mailcow.conf
cat <<EOF > data/conf/borgmatic/etc/config.yaml
location:
    source_directories:
        - /mnt/source
    repositories:
        - user@rsync.net:mailcow
    exclude_patterns:
        - '/mnt/source/postfix/public/'
        - '/mnt/source/postfix/private/'
        - '/mnt/source/rspamd/rspamd.sock'

retention:
    keep_hourly: 24
    keep_daily: 7
    keep_weekly: 4
    keep_monthly: 6
    prefix: ""

hooks:
    mysql_databases:
        - name: ${DBNAME}
          username: ${DBUSER}
          password: ${DBPASS}
          options: --default-character-set=utf8mb4
EOF
```

Das Erstellen der Datei auf diese Weise stellt sicher, dass die korrekten MySQL-Zugangsdaten aus `mailcow.conf` übernommen werden.

Diese Datei ist ein minimales Beispiel für die Verwendung von borgmatic mit einem Konto `user` beim Cloud-Speicheranbieter `rsync.net` für
ein Repository namens `mailcow` (siehe `repositories` Einstellung). Es wird sowohl das maildir als auch die MySQL-Datenbank sichern, was alles ist
was alles ist, was Sie brauchen, um Ihr mailcow Setup nach einem Vorfall wiederherzustellen. Die Aufbewahrungseinstellungen werden ein Archiv für
jede Stunde der letzten 24 Stunden, eines pro Tag der Woche, eines pro Woche des Monats und eines pro Monat des letzten halben
Jahr.

Schauen Sie in der [borgmatic Dokumentation](https://torsion.org/borgmatic/) nach, wie Sie andere Arten von Repositories oder
Konfigurationsoptionen. Wenn Sie ein lokales Dateisystem als Backup-Ziel verwenden, stellen Sie sicher, dass Sie es in den
Container einbinden. Der Container definiert zu diesem Zweck ein Volume namens `/mnt/borg-repository`.

!!! note
    Wenn Sie rsync.net nicht verwenden, können Sie wahrscheinlich das Element `remote_path` aus Ihrer Konfiguration streichen.
	
### Erstellen Sie einen crontab

Erstellen Sie eine neue Textdatei in `data/conf/borgmatic/etc/crontab.txt` mit folgendem Inhalt:

```
14 * * * * PATH=$PATH:/usr/bin /usr/bin/borgmatic --stats -v 0 2>&1
```

Diese Datei erwartet eine crontab-Syntax. Das hier gezeigte Beispiel veranlasst das Backup, jede Stunde um 14 Minuten nach
nach der vollen Stunde auszuführen und am Ende einige nette Statistiken zu protokollieren.

### SSH-Schlüssel in Ordner ablegen

Legen Sie die SSH-Schlüssel, die Sie für entfernte Repository-Verbindungen verwenden wollen, in `data/conf/borgmatic/ssh` ab. OpenSSH erwartet die
übliche `id_rsa`, `id_ed25519` oder ähnliches in diesem Verzeichnis zu finden. Stellen Sie sicher, dass die Datei `chmod 600` ist und nicht von der Welt gelesen werden kann
oder OpenSSH wird sich weigern, den SSH-Schlüssel zu benutzen.

### Den Container hochfahren

Für den nächsten Schritt müssen wir den Container in einem konfigurierten Zustand hochfahren und laufen lassen. Um das zu tun, führen Sie aus:

```shell
docker-compose up -d
```

## Wiederherstellung von einem Backup

Das Wiederherstellen eines Backups setzt voraus, dass Sie mit einer neuen Installation von mailcow beginnen, und dass Sie derzeit keine
keine benutzerdefinierten Daten in ihrem maildir oder ihrer mailcow Datenbank.

### Wiederherstellen von maildir

!!! warning
    Dies wird Dateien in Ihrem maildir überschreiben! Führen Sie dies nicht aus, es sei denn, Sie beabsichtigen tatsächlich, Mail
    Dateien von einem Backup wiederherzustellen.

!!! note "Wenn Sie SELinux im Erzwingungsmodus verwenden"
    Wenn Sie mailcow auf einem Host mit SELinux im Enforcing-Modus verwenden, müssen Sie es vorübergehend deaktivieren während
    während der Extraktion des Archivs vorübergehend deaktivieren, da das Mailcow-Setup das vmail-Volumen als privat kennzeichnet, das ausschließlich dem Dovecot-Container
    ausschließlich. SELinux wird (berechtigterweise) jeden anderen Container, wie z.B. den borgmatic Container, daran hindern, auf
    dieses Volume zu schreiben.

Bevor Sie eine Wiederherstellung durchführen, müssen Sie das vmail-Volume in `docker-compose.override.yml` beschreibbar machen, indem Sie das
das `ro`-Flag aus dem Volume entfernen.
Dann können Sie den folgenden Befehl verwenden, um das Maildir aus einem Backup wiederherzustellen:

```shell
docker-compose exec borgmatic-mailcow borgmatic extract --path mnt/source --archive latest
```

Alternativ können Sie auch einen beliebigen Archivnamen aus der Liste der Archive angeben (siehe
[Auflistung aller verfügbaren Archive](#auflistung-aller-verfugbaren-archive))

### MySQL wiederherstellen

!!! warning
    Die Ausführung dieses Befehls löscht und erstellt die mailcow-Datenbank neu! Führen sie diesen Befehl nicht aus, es sei denn sie beabsichtigen, die mailcow-Datenbank von einem Backup wiederherzustellen.

Um die MySQL-Datenbank aus dem letzten Archiv wiederherzustellen, verwenden Sie diesen Befehl:

```shell
docker-compose exec borgmatic-mailcow borgmatic restore --archive latest
```

Alternativ können Sie auch einen beliebigen Archivnamen aus der Liste der Archive angeben (siehe
[Auflistung aller verfügbaren Archive](#auflistung-aller-verfugbaren-archive))

### Nach der Wiederherstellung

Nach der Wiederherstellung müssen Sie mailcow neu starten. Wenn Sie den SELinux-Erzwingungsmodus deaktiviert haben, wäre jetzt ein guter Zeitpunkt, um
ihn wieder zu aktivieren.

Um mailcow neu zu starten, verwenden Sie den folgenden Befehl:

```shell
docker-compose down && docker-compose up -d
```

Wenn Sie SELinux verwenden, werden dadurch auch alle Dateien in Ihrem vmail-Volume neu benannt. Seien Sie geduldig, denn dies kann
eine Weile dauern kann, wenn Sie viele Dateien haben.

## Nützliche Befehle

### Manueller Archivierungslauf (mit Debugging-Ausgabe)

```shell
docker-compose exec borgmatic-mailcow borgmatic -v 2
```

### Auflistung aller verfügbaren Archive

```shell
docker-compose exec borgmatic-mailcow borgmatic list
```

### Sperre aufheben

Wenn borg während eines Archivierungslaufs unterbrochen wird, hinterlässt es eine veraltete Sperre, die gelöscht werden muss, bevor
neue Operationen durchgeführt werden können:

```shell
docker-compose exec borgmatic-mailcow borg break-lock user@rsync.net:mailcow
```

Wobei `user@rsync.net:mailcow` die URI zu Ihrem Repository ist.

Jetzt wäre ein guter Zeitpunkt, einen manuellen Archivierungslauf durchzuführen, um sicherzustellen, dass er erfolgreich durchgeführt werden kann.

### Exportieren von Schlüsseln

Wenn Sie eine der `keyfile`-Methoden zur Verschlüsselung verwenden, **MÜSSEN** Sie sich selbst um die Sicherung der Schlüsseldateien kümmern. Die
Schlüsseldateien werden erzeugt, wenn Sie das Repository initialisieren. Die `repokey`-Methoden speichern die Schlüsseldatei innerhalb des
Repository, so dass eine manuelle Sicherung nicht so wichtig ist.

Beachten Sie, dass Sie in beiden Fällen auch die Passphrase haben müssen, um die Archive zu entschlüsseln.

Um die `keyfile` zu holen, führen Sie aus:

```shell
docker-compose exec borgmatic-mailcow borg key export --paper user@rsync.net:mailcow
```

Wobei `user@rsync.net:mailcow` die URI zu Ihrem Repository ist.