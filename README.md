# KTS-Doku
## Setup-Anleitung für das Kinoticket-System
Diese Anleitung beschreibt zwei Methoden zur Einrichtung des Kinoticket-Systems: die Verwendung von Docker und Docker Compose sowie die lokale Einrichtung ohne Docker. Die Docker-Methode wird aufgrund ihrer Einfachheit und Nähe zum Produktionssystem bevorzugt.

### Lokales Aufsetzen mit Docker

#### Voraussetzungen
* Docker und Docker Compose müssen auf Ihrem System installiert sein. Die Installationsanleitung variiert je nach Betriebssystem. Anleitungen finden Sie auf den offiziellen Docker-Webseiten für [Docker](https://docs.docker.com/desktop/install/mac-install/) und [Docker Compose](https://docs.docker.com/compose/install/).
* Im Rootverzeichnis des Projekts müssen die Dateien docker-compose.yml und init-db.sql vorhanden sein. Der Inhalt der init-db.sql-Datei sollte der hier bereitgestellten [Datei](https://github.com/ELITE-Kinoticketsystem/Backend-KTS/blob/main/database_scripts/create_database.sql) entsprechen.

#### Docker Compose Konfiguration
Die `docker-compose.yml` definiert die Konfiguration und die Orchestrierung der Container. Die Datei sollte wie folgt aussehen.

```
version: '3.8'

services:
  db:
    image: mysql
    command: --default-authentication-plugin=caching_sha2_password
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: KinoTicketSystem
      MYSQL_USER: user
      MYSQL_PASSWORD: 123456789
    ports:
      - 3306:3306
    volumes:
      - db_data:/var/lib/mysql
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-u", "user", "--password=123456789"]
      start_period: 5s
      interval: 5s
      timeout: 5s
      retries: 55

  adminer:
    image: adminer
    restart: always
    ports:
      - 8089:8080

  kts-backend:
    image: cll1n/kts-backend
    container_name: kts-backend-container
    depends_on:
      db:
        condition: service_healthy
    ports:
      - 8080:8080
    healthcheck:
      test:
        [
          "CMD",
          "curl",
          "-f",
          "kts-backend-container:8080/lifecheck"
        ]
      interval: 15s
      start_period: 10s
      timeout: 10s
      retries: 4
    environment:
      - DB_HOST=
      - DB_PORT=
      - DB_NAME=
      - DB_USER=
      - DB_PASSWORD=
      - JWT_SECRET= 
      - MAILGUN_API_KEY=
      - MAILGUN_DOMAIN=

  kts-frontend:
    image: cll1n/kts-frontend
    container_name: kts-frontend-container
    depends_on:
      kts-backend:
        condition: service_healthy
    ports:
      - 80:3000

volumes:
  db_data:
```

##### Services im Detail
* MySQL-Container: Hostet die Datenbank. Initialisiert sich mit einem SQL-Skript für die erforderlichen Tabellen und Daten.
* Adminer-Container: Ermöglicht die Verwaltung der Datenbank über einen Webbrowser.
* Backend-Service: Wird in einem eigenen Container gehostet und startet erst, wenn die Datenbank verfügbar ist. Regelmäßige Gesundheitsprüfungen werden durchgeführt.
* Frontend-Service: Wird in einem separaten Container bereitgestellt und startet erst, wenn das Backend funktionsfähig ist.

#### Starten der Anwendung
In dem Rootverzeichnis des Projekts den Befehl `docker-compose up -d` ausführen. Nach dem Start sind die Services über folgende Ports erreichbar:
* Adminer: http://localhost:8089
* Frontend: http://localhost
* Backend: http://localhost:8080

### Lokales Aufsetzen ohne Docker
#### Datenbank
* MySQL lokal installieren und das init-db.sql-Skript ausführen, um die erforderlichen Tabellen zu erstellen.
#### Backend
* Das Git-Repository Klonen und sicher stellen, dass Go installiert ist. `go mod download` ausführen, um die Abhängigkeiten herunterzuladen. Die `.env.example` in `.env` umbennen und die Umgebungsvariablen setzen. Um das Backend zu starten, das `set-and-run.sh` ausführen.
#### Frontend
* Das Git-Repository Klonen und sicher stellen, dass Node.js und pnpm installiert sind. Die Variable apiUrl in der Datei authService.ts auf http://localhost:8080 setzen. Die Abhängigkeiten mit `pnpm install` und den Entwicklerserver mit `pnpm run dev` starten.
