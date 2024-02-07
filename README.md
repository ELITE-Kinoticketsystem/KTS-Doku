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

# Tech Stack

## Backend

### GoLang
- **Performance:** Entwickelt von Google, bietet Go hohe Leistung durch kompilierte Sprache. Ist auf hohe Leistung Optimiert
- **Einfache Syntax:** Go's klare und einfache Syntax fördert schnelle Entwicklung und Wartbarkeit
- **Effiziente Ressourcennutzung:** Die integrierte Garbage Collection entlastet Entwickler von manuellem Speichermanagement.
- **Integriertes Testing:** Die Standardbibliothek unterstützt das Schreiben von Tests direkt, was die Robustheit der Anwendung sicherstellt.

### Gin
- **Schnell:** Gin ist für seine hohe Leistung bekannt, was schnelle Antwortzeiten für Endnutzer bedeutet.
- **Middleware-Unterstützung:** Ermöglicht die einfache Integration von Funktionalitäten wie Authentifizierung, Logging etc.
- **JSON-Validierung:** Vereinfacht die Verarbeitung und Validierung eingehender Daten.
- **Gruppierung von Routen:** Verbessert die Organisation und Übersichtlichkeit der Endpunkte.
- **Fehler-Management:** Hilft bei der effektiven Handhabung von Anwendungsfehlern.

### Go-Jet
- **Automatisch generierte Datenmodelltypen:** Erleichtert die Arbeit mit Datenbanktypen und Ergebnissen von Datenbankabfragen.
- **Typsicherer SQL Builder:** Minimiert Fehler bei der Erstellung von SQL-Queries und unterstützt eine Vielzahl von Anweisungen wie Select, Insert, Update und Delete.

### MySQL
- **Open-Source und Kostenfreiheit:** Ideal für Projekte mit begrenztem Budget.
- **Skalierbarkeit und Leistung:** Erfüllt unterschiedliche Anforderungen mit optimierter Leistung.
- **Umfangreiche Community und Unterstützung:** Zugang zu einem breiten Erfahrungsschatz und kontinuierlicher Entwicklung.
- **Standards und Interoperabilität:** Ermöglicht einfache Integration in verschiedene Plattformen und Technologien.

### Aufbau
Die Bearbeitung einer Anfrage an das Backend kann in 3 Abschnitte unterteilt werden. Dementsprechend ist der Großteil der Funktionalität des Backends in 3 packages strukturiert.

0. Zunächst wird durch den Router die Route der einkommenden Anfrage überprüft, um dann den jeweiligen spezifischen Handler aufzurufen, der für diese Anfrage zuständig ist. Gegebenenfalls werden vor dem Handler noch middlewares durchlaufen, wie im Abschnitt Authentifizierung erklärt.

1. Der Handler übernimmt die Validierung der Payload.
Diese kann einerseits im Body der Request im json-Format oder aber als dynamischer Routenparameter, meist in Form einer UUID vorliegen.
Handelt es sich um syntaktisch inkorrekte oder unerwartete Daten, wird der HTTP-Fehlercode 400 (BAD REQUEST) zurückgegeben.
Ansonsten werden die Daten an die nächste Stelle, den Controller, weitergegeben.
Das Ergebnis dieser Operation erhält der Handler zurück und gibt dieses bei Erfolg entweder mit dem Statuscode 200 (OK) oder 201 (CREATED) zurück.
Die Handler sind im package _handlers_ zu finden.

2. Der Controller erhält vom Handler die syntaktisch validen Daten.
Er dient zur Bündelung der Datenbankabfragen, da manche Operationen mehrere Datenbankzugriffe erfordern.
Bei der Erstellung eines Kinosaals muss nämlich zum einen die _cinema_halls_ Tabelle aber auch die _seats_ Tabelle angesprochen werden.
Die Controller befinden sich im package _controllers_.

4. Die Datenbankanfragen werden über sogenannte Repositories realisiert.
Dabei wird einem Repository meist eine konkrete Datenbanktabelle zugeordnet, für welche diese zuständig ist.
Der Zugriff auf diese Tabelle erfolgt daher ausschließlich über dieses Repository. Die Repositories sind im package _repositories_ zu finden.

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


# Buchungsprozess

Der Buchungsprozess beginnt mit der Auswahl einer Veranstaltung. Eine Veranstaltung ist ein Special events oder eine einfache Vorstellungen und findet in einem Kinosaal statt. Deshalb wird man nach Auswahl der Veranstaltung zur Sitzauswahl im Saal weitergeleitet. Hier kann der Benutzer Sitze auswählen, die er buchen oder reservieren möchte. Dabei sind die Sitze, welche bereits von anderen Benutzern belegt wurden ausgegraut und nicht auswählbar. Der Belegt-Zustand ist der Zustand, indem sich der Sitz befindet, wenn er entweder erfolgreich gebucht, reserviert oder blockiert wurde. Und blockiert ist der Sitz, wenn ein Nutzer den Sitz ausgewählt hat und kein anderer Nutzer diesen bereits reserviert, gebucht oder ausgewählt hat. Der Zustand der Blockierung ist zeitgebunden und beträgt initial 15min, jedoch kann durch erfolgreiche Blockierungen weiterer Sitze der Timer neu gestartet werden, was im weiteren Verlauf noch erklärt wird. Des Weiteren ist die Auswahl von Sitzen in verschiedenen Sitzreihen nicht erlaubt, sowie die Auswahl von nicht-benachbarten Sitzen. Das wird verhindert, um Fragmentierung zu vermeiden und somit möglichst große Sitzgruppen von nicht belegten Sitzen zu ermöglichen. Ferner ist hier das grundlegende Problem der Nebenläufigkeit(Doppelbuchungen) derart gelöst, dass bei Auswahl eines Sitzes geprüft wird, ob dieser nicht bereits von anderen Benutzern belegt wurde. Da die Datenbankzugriffe atomar ablaufen und somit sequenzielle Buchungen erzwingen, bleiben die Belegungen konsistent. Bei jedem Seiten-Neustart und jeder versuchten Sitzauswahl, werden die Sitze neu nachgeladen und es wird aktualisiert angezeigt, ob Sitze bereits belegt sind. Es wurde sich für diese Lösung entschieden, um den Mittelweg zwischen Ressourcenverbrauch für das Refreshen minimal zu halten, aber dennoch die User-experience durch das Anzeigen von veralteten Daten möglichst wenig einzuschränken. Ist der Versuch einen Sitz zu buchen erfolgreich, wird der Nutzer über die selection-overview und die Sitzfarbe darauf hingewiesen, dass die Blockierung erfolgreich war. Bei nicht-erfolgreicher Blockierung erscheint ein Warnhinweis mit Erklärung in der Mitte des Bildschirms. Gelingt es dem Nutzer mindestens einen Sitz erfolgreich zu blockieren startet ein 15-min Timer. Dieser Timer gibt an, wie lange der Sitz noch von dem user belegt ist, bevor dieser wieder frei gegeben wird. Bei jeder erfolgreichen neuen Blockierung, startet dieser Timer erneut(15min) und gibt die Zeit an, wie lange alle blockierten Sitze des Benutzers noch belegt sind und somit für andere Nutzer nicht auswählbar sind. Gelingt dem Benutzer mindestens eine erfolgreiche Blockierung, kann er zur Buchung/Reservierung weitergehen. Hier wird nochmals in einer Übersicht angezeigt, welche Sitze er im Moment blockiert. Außerdem wird die Gesamtsumme, sowie die Kosten der einzelnen Tickets angezeigt. Abschließend kann der Benutzer auswählen, ob er die Tickets direkt bezahlen will oder die Sitze lediglich reservieren will. Hierbei ist anzumerken, dass der Timer für die Sitzbelegung nicht neu startet, um einen fairen Buchungsprozess für alle Benutzer zu gewährleisten. In beiden Fällen führt der erfolgreiche Abschluss einer Reservierung oder Bezahlung zu einer permanenten Belegung des Sitzes. Im Anschluss an die Bestätigung durch Bezahlung oder Reservierungsabschluss werden dem Benutzer die Tickets angezeigt, wobei jedes Ticket mit einem QR-Code kommt, welches bei Veranstaltungsbeginn gescannt werden kann, um das Ticket zu authentifizieren.


## User Stories

**Anforderung:**
Als Maria Mutter möchte ich Filme filtern können, weil ich meine Kinder nur Filme gucken lassen, die **FSK sicher** sind.

Wurde umgesetzt durch
eine Filterfunktion bei der Filmsuche

<img width="137" alt="image" src="https://github.com/ELITE-Kinoticketsystem/KTS-Doku/assets/104858641/9e51ba53-2eb8-44b9-949e-834184883b3b">

**Anforderung:**
Als Maria Mutter möchte ich **mehrere Sitze auswählen** können, weil ich mit meiner Familie zusammen sitzen möchte

Wird erlaubt mit der Einschränkung, dass
Die Sitze benachbart sein müssen und
In einer Reihe, um die Sitzauslastung zu
Optimieren.

<img width="164" alt="image" src="https://github.com/ELITE-Kinoticketsystem/KTS-Doku/assets/104858641/a73b9719-7ed6-4413-9869-84c0f644f683">

**Anforderung:**
Als Alex
Administrator möchte ich **Vorstellungen konfigurieren** können, weil Veranstaltungen sich individuell voneinander unterscheiden.

Durch Einstellungsmöglichkeiten der Sitzpreise,
Filmauswahl, Titel, Beschreibung, Ticketpreise,
Kino + Saal, Datum und Uhrzeit, können
Veranstaltungen individualisiert werden.

<img width="154" alt="image" src="https://github.com/ELITE-Kinoticketsystem/KTS-Doku/assets/104858641/eac63cd5-0782-412e-a983-58aaaf072cfc">

**Anforderung:**
Als Alex
Administrator möchte ich meine **Kinosäle grafisch abbilden** lassen, weil ich meinen Kunden eine realistische Vorstellung von ihrem Kinoerlebnis im Voraus geben will.

Durch Auswahl von Sitztyp:
- Regular
- Double
- Disabled
und Sitzkategorie können einzelne
Sitze in einem Grid platziert werden.

<img width="208" alt="image" src="https://github.com/ELITE-Kinoticketsystem/KTS-Doku/assets/104858641/d2131622-c0c8-4cfd-8542-795d3144a052">

**Anforderung:**
Als Finja Filmfanatikerin möchte ich meine **Buchungsverläufe einsehen** können, weil ich dadurch mit meinen Freunden austauschen kann, wie oft ich im Kino war und welche Filme ich gesehen habe.

Im Benutzer-Dashboard unter dem
Ticket-Reiter ist die Ticket-Historie
einsehbar.

<img width="223" alt="image" src="https://github.com/ELITE-Kinoticketsystem/KTS-Doku/assets/104858641/9728972a-e76f-4b0c-8b99-369dc3b03a7c">

