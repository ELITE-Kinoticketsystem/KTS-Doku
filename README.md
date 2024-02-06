# KTS-Doku
# Setup-Anleitung für das Kinoticket-System
Diese Anleitung beschreibt zwei Methoden zur Einrichtung des Kinoticket-Systems: die Verwendung von Docker und Docker Compose sowie die lokale Einrichtung ohne Docker. Die Docker-Methode wird aufgrund ihrer Einfachheit und Nähe zum Produktionssystem bevorzugt.

## Lokales Aufsetzen mit Docker

### Voraussetzungen
* Docker und Docker Compose müssen auf Ihrem System installiert sein. Die Installationsanleitung variiert je nach Betriebssystem. Anleitungen finden Sie auf den offiziellen Docker-Webseiten für [Docker](https://docs.docker.com/desktop/install/mac-install/) und [Docker Compose](https://docs.docker.com/compose/install/).
* Im Rootverzeichnis des Projekts müssen die Dateien docker-compose.yml und init-db.sql vorhanden sein. Der Inhalt der init-db.sql-Datei sollte der hier bereitgestellten [Datei](https://github.com/ELITE-Kinoticketsystem/Backend-KTS/blob/main/database_scripts/create_database.sql) entsprechen.

### Docker Compose Konfiguration
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
      MYSQL_PASSWORD: password
    ports:
      - 3306:3306
    volumes:
      - db_data:/var/lib/mysql
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-u", "user", "--password=password"]
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

#### Services im Detail
* MySQL-Container: Hostet die Datenbank. Initialisiert sich mit einem SQL-Skript für die erforderlichen Tabellen und Daten.
* Adminer-Container: Ermöglicht die Verwaltung der Datenbank über einen Webbrowser.
* Backend-Service: Wird in einem eigenen Container gehostet und startet erst, wenn die Datenbank verfügbar ist. Regelmäßige Gesundheitsprüfungen werden durchgeführt.
* Frontend-Service: Wird in einem separaten Container bereitgestellt und startet erst, wenn das Backend funktionsfähig ist.

### Starten der Anwendung
In dem Rootverzeichnis des Projekts den Befehl `docker-compose up -d` ausführen. Nach dem Start sind die Services über folgende Ports erreichbar:
* Adminer: http://localhost:8089
* Frontend: http://localhost
* Backend: http://localhost:8080

## Lokales Aufsetzen ohne Docker
### Datenbank
* MySQL lokal installieren und das init-db.sql-Skript ausführen, um die erforderlichen Tabellen zu erstellen.
### Backend
* Das Git-Repository Klonen und sicher stellen, dass Go installiert ist. `go mod download` ausführen, um die Abhängigkeiten herunterzuladen. Die `.env.example` in `.env` umbennen und die Umgebungsvariablen setzen. Um das Backend zu starten, das `set-and-run.sh` ausführen.
### Frontend
* Das Git-Repository Klonen und sicher stellen, dass Node.js und pnpm installiert sind. Die Variable apiUrl in der Datei authService.ts auf http://localhost:8080 setzen. Die Abhängigkeiten mit `pnpm install` und den Entwicklerserver mit `pnpm run dev` starten.


# Pipelines
In diesem Projekt wird Github Actions verwendet, um die Pipelines zu automatisieren. Diese Pipelines gewährleisten, dass der Code bei jedem Push oder Pull Request in den Hauptbranch automatisch getestet, gebaut und bereitgestellt wird.

## Continuous Integration (CI)

Die CI-Pipeline wird bei jedem `push` oder `pull_request` auf den `main`-Branch ausgelöst. Die Pipeline führt verschiedene Jobs aus, um die Qualität und Funktionalität des Codes sicherzustellen.

### Jobs und Schritte

1. **Build**
   - **Umgebung:** Läuft auf einem Ubuntu-Server mit der neuesten Version.
   - **Schritte:**
     - **Checkout Code:** Der aktuelle Code wird ausgecheckt, um mit der Pipeline arbeiten zu können.
     - **Set up Go:** Die Go-Umgebung wird mit einer spezifischen Version (1.21.5) eingerichtet.
     - **Create `go.mod`:** Eine `go.mod`-Datei wird erstellt, und anschließend die Abhängigkeiten installiert.
     - **Install Staticcheck:** Staticcheck, ein Werkzeug für statische Code-Analyse in Go, wird installiert.
     - **Build project and verify dependencies:** Das Projekt wird gebaut, und die Abhängigkeiten werden verifiziert.
     - **Verify Code Quality:** Die Code-Qualität wird durch `go vet` und Staticcheck überprüft, um gängige Fehler und Stilprobleme zu identifizieren.
     - **Test:** Die Tests werden ausgeführt.
     - **Update coverage report:** Ein Testabdeckungsbericht wird erstellt und aktualisiert.


## Continuous Deployment (CD)

Die CD-Pipeline wird bei jedem `push` auf den `main`-Branch ausgelöst. Sie baut das Docker-Image des Projekts und pusht es in ein Docker-Repository. Anschließend wird der Dienst auf einem entfernten Server neu gestartet.

### Jobs und Schritte

1. **docker-build**
   - **Umgebung:** Läuft auf einem Ubuntu-Server mit der neuesten Version.
   - **Schritte:**
     - **Checkout Code:** Der aktuelle Code wird ausgecheckt, um mit der Pipeline arbeiten zu können.
     - **Set up Docker Buildx:** Docker Buildx wird konfiguriert, um das Bauen und Pushen von Docker-Images zu unterstützen.
     - **Login to Docker Hub:** Es wird eine Anmeldung bei Docker Hub durchgeführt, um die gebauten Images dort ablegen zu können.
     - **Build and push:** Das Docker-Image wird gebaut und in das Docker Hub Repository gepusht.

2. **start-service**
   - **Abhängigkeit:** Benötigt den erfolgreichen Abschluss des `docker-build`-Jobs.
   - **Umgebung:** Läuft auf einem Ubuntu-Server mit der neuesten Version.
   - **Schritte:**
     - **Execute remote ssh commands:** Über SSH werden Befehle auf einem entfernten Server ausgeführt. Dabei wird das Docker-Compose-Projekt aktualisiert, der alte Dienst gestoppt, entfernt und der neue Dienst gestartet.

### Sicherheitsaspekte

- Sensitive Informationen wie Docker Hub-Anmeldedaten und SSH-Schlüssel für den entfernten Server werden als GitHub Secrets gespeichert und in den Workflows verwendet.


# Datenmodell
![Datenmodell](https://github.com/ELITE-Kinoticketsystem/KTS-Doku/blob/main/Datenstruktur.png)
Unser Datenmodell ist auf Events ausgerichtet, welches die einmalige Vorstellung eines Films oder mehrerer Filme repräsentiert. Ein Event wird in unserer Datenbank mit einem Start- und Enddatum definiert, um die Dauer des Ereignisses festzulegen. Jedes Event ist einzigartig und wird durch eine ID identifiziert.

Ein Event kann mehrere Filme beinhalten, was durch die Beziehungstabelle event_movies dargestellt wird. Diese verbindet die events-Tabelle mit der movies-Tabelle über die jeweiligen IDs. Die movies-Tabelle enthält wichtige Informationen über die Filme, wie Titel, Beschreibung, Banner, Trailer-URL und Veröffentlichungsdatum.
Es gibt auch eine genres-Tabelle, die die verschiedenen Filmgenres beinhaltet. Filme können einem oder mehreren Genres zugeordnet werden, was durch die Beziehungstabelle movie_genres dargestellt wird. Schließlich gibt es noch die actors-Tabelle, die Informationen über die Schauspieler enthält, und die movie_actors-Beziehungstabelle, die Schauspieler mit den jeweiligen Filmen verknüpft.

Um die Interaktion mit Kunden zu erleichtern, umfasst unser Modell eine users-Tabelle, die Details wie Benutzername, E-Mail und Passwort für den Login-Prozess speichert. Die Kunden können Bewertungen zu den Filmen abgeben, was in der reviews-Tabelle erfasst wird, die eine Verbindung zur movies-Tabelle und zur users-Tabelle herstellt, um zu verfolgen, welcher Benutzer welche Bewertung abgegeben hat.

Die Events finden in Kinosälen statt, die in der Tabelle cinema_halls mit Eigenschaften wie Kapazität und Abmessungen aufgeführt sind. Diese Säle befinden sich an bestimmten Standorten, die in der addresses-Tabelle definiert sind und über eine Beziehung zur theatres-Tabelle verfügen.

Die Sitzplatzverwaltung wird durch die Tabellen seats, seat_categories, event_seats und event_seat_categories gehandhabt. seats beschreibt die physischen Sitze in einem Kino, während seat_categories die verschiedenen Sitzkategorien (Regular, Vip, Loge) definiert. event_seats verknüpft Sitze mit Events und speichert Informationen darüber, ob und wann ein Sitz reserviert oder ausgewählt wurde. event_seat_categories verbindet die Sitzkategorien mit den spezifischen Events und legt die Preise für die Sitzplätze fest.

Um Transaktionen zu verwalten, gibt es die tickets- und orders-Tabellen, welche die informationen über Käufe von Tickets für Events speichern.


# KTS-Anmeldung

## JWT (JSON Web Token)

Wir haben uns dafür entschieden, JWT (JSON Web Tokens) für die Implementierung der Authentifizierung und Autorisierung in unserer Webanwendung aus mehreren Gründen:

**Zustandslosigkeit**: JWTs sind zustandslos, was bedeutet, dass der Server keinen Sitzungsstatus speichern muss. Dies verbessert die Skalierbarkeit der Anwendung und verringert die Last auf dem Server, was besonders wichtig ist, wenn wir mit einem großen Benutzerstrom rechnen.

**Sicherheit**: JWTs bieten eine robuste Sicherheitsschicht, da sie signiert werden können, um sicherzustellen, dass sie nicht manipuliert wurden. Dies erhöht das Vertrauen in die Integrität der übertragenen Daten und schützt vor potenziellen Angriffen wie CSRF (Cross-Site Request Forgery).

**Flexibilität**: JWTs können verschiedene Arten von Nutzlasten (Payloads) enthalten, die für unsere Anwendung spezifische Informationen enthalten können. Dadurch können wir benutzerdefinierte Daten über den Benutzer transportieren und eine personalisierte Benutzererfahrung bieten.

**Einfache Integration**: JWTs basieren auf offenen Standards und können leicht in verschiedene Plattformen und Sprachen integriert werden. Dies ermöglicht eine reibungslose Zusammenarbeit zwischen Frontend- und Backend-Entwicklern und vereinfacht die Wartung und Skalierung unserer Anwendung.

**Token**: Ein JWT-Token besteht aus drei Teilen, die durch Punkte getrennt sind: Header, Payload und Signatur. Diese Teile werden Base64-kodiert und durch Punkte getrennt, um ein JWT-Token zu bilden.

**Refresh-Token**: Ein Refresh-Token ist ein spezielles JWT-Token, das verwendet wird, um ein neues JWT-Token zu erhalten, wenn das alte abgelaufen ist. Dies ermöglicht es uns, die Gültigkeitsdauer unserer JWT-Tokens zu begrenzen und die Sicherheit unserer Anwendung zu erhöhen.

## Registrierung

Die Registrierung erfolgt über die [KTS-Register-Page](https://cinemika.germanywestcentral.cloudapp.azure.com/auth/register).
Der User ist verpflichtet, folgende Daten anzugeben:

- Vorname
- Nachname
- Username
- E-Mail
- Passwort, das mindestens 8 Zeichen lang sein muss, mindestens einen Großbuchstaben, einen Kleinbuchstaben, eine Ziffer und ein Sonderzeichen enthalten muss.

Das Fronted schickt die Daten im folgenden Format an das Backend über einen POST-Request an die Route `/auth/register`:

```json
{
  "firstName": "Michael",
  "lastName": "Jordan",
  "username": "MJ23",
  "email": "jordan.micheal@nba.com",
  "password": "Passwort123!"
}
```

Nachdem der User die Registrierung abgeschlossen hat, wird eine E-Mail an die angegebene E-Mail-Adresse gesendet, um die Registrierung zu bestätigen. Außerdem wird der User automatisch eingeloggt und erhält einen JWT-Token und einen refreshToken, der über Cookies vom Backend mitgeschickt wird. Er muss sich also nicht nochmal einloggen.

## Anmeldung

Die Anmeldung erfolgt über die [KTS-Login-Page](https://cinemika.germanywestcentral.cloudapp.azure.com/auth/login).
Der User ist verpflichtet, folgende Daten anzugeben:

- Username
- Passwort

Das Fronted schickt die Daten im folgenden Format an das Backend über einen POST-Request an die Route `/auth/login`:

```json
{
  "username": "MJ23",
  "password": "Passwort123!"
}
```

Sollte das Passwort oder der Username falsch sein, wird ein generischer Fehler zurückgegeben und der User muss die Anmeldung erneut versuchen. Wir haben uns für einen generischen Fehler entschieden, um zu verhindern, dass ein potenzieller Angreifer Informationen über die Gültigkeit von Benutzernamen und Passwörtern erhält.
Ist die Anmeldung erfolgreich, erhält der User einen JWT-Token und einen refreshToken, der über Cookies vom Backend mitgeschickt wird.

## Authentifizierung
Die bereitgestellten Endpunkte des Backends lassen sich in 3 Kategorien einteilen:
- Öffentliche Endpunkte: Diese Endpunkte sind für alle Benutzer zugänglich und erfordern keine Authentifizierung. Beispiele hierfür sind die Registrierung und die Anmeldung.
- Geschützte Endpunkte: Diese Endpunkte erfordern einen gültigen JWT-Token, um aufgerufen zu werden. Beispiele hierfür sind die Buchungen und das Erstellen von Reviews. Hierfür wird eine middleware verwendet, die vor der eigentlichen Abarbeitung der Anfrage zwischengeschaltet ist. Sie überprüft den JWT-Token und authentifiziert den Benutzer. Außerdem enthält der JWT-Token die UserId des Benutzers, die in diesem Schritt extrahiert wird und an die Anfrage weitergegeben wird. Wenn kein gültiger JWT-Token mitgeschickt wurde, wird der HTTP-Statuscode 401 (Unauthorized) zurückgegeben.
- Admin-Endpunkte: Diese Endpunkte erfordern eine spezielle Rolle, um aufgerufen zu werden. Beispiele hierfür sind die Endpunkte für das Dashboard sowie das Erstellen von Kinosälen. Auch hier wird eine middleware verwendet, die vor der eigentlichen Abarbeitung der Anfrage zwischengeschaltet ist. Sie überprüft den JWT-Token und die Rolle des Benutzers. Wenn der Benutzer nicht die erforderliche Rolle hat, wird der HTTP-Statuscode 403 (Forbidden) zurückgegeben.
