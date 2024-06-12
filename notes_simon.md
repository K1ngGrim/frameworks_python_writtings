# Vorlesung 1
## docker
### informationen zu image anzeigen
image identifiziert über `tag`, zB `postgres:16.2-bookworm`
- ``docker inspect tag``
- ``docker sbom tag`` (software bill of materials)
- ``dive.exe tag``
    - ermöglicht das tiefe "eintauchen" in die Struktur des image
    - Darstellung über Schichten & Baumstruktur
    - drei Sichten, wechseln mit Tab 
    - Befehle in Leiste unten mit Strg
    
### image starten
- `docker compose`
- ``compose.yml``/``docker-compose.yaml`` zur Spezifikation von Services
    - gültige JSON-Datei -> gültige yaml-Datei
    - yaml ist Obermenge von JSON
    - Einrückung wichtig (immer um 2)
    - Baumstruktur
    - ports-attribut -> Verbindung ports Windows und Linux-Image
- docker compose exec 'servicename' 'befehl'
    - führe Befehl in image aus
    - zb docker compose exec db bash 

# Vorlesung 2
- Primärschlüssel müssen eindeutig sein und sollten sich nicht ändern -> zb email ungeeignet
- komplexere Datemsätze als Attribut abbilden:
    - json Objekt zur Simulation von String-Arrays
    - varchar
    - zweite Tabelle mit 1:n oder m:n Beziehung via JOINs
- Startreihenfolge `.\.venv\Scripts\Activate.ps1`, ` python -m flaskapp`, `docker compose up`
- ``patient_get_controller.py``
    - @ctrl.get definiert die Route
    - @roles_allowed definiert erlaubte Rollen -> sonst Unauthorized oder Forbidden

# Vorlesung 10.05.
## Datenbankupdate
- postgres-version in `.extras\compose\db\postgres\compose.yml` auf **16.3** aktualisiert, anschließend `docker pull postgres:16.3-bookworm`
- in diesem Fall funktioniert anschließendes `docker compose up` ohne Probleme, in der Regel ist nach Update aber oft ist manuelle Migration notwendig. Dann müsste man
    - server herunterfahren
    - in ``C:\Zimmermann\volumes\postgres\data`` löschen
    - neu starten

    (für uns dieses Mal nicht notwendig)

## Dokumentation
- mögliche Tools: mkdocs, sphinx,...
### mkdocs
- python venv öffnen `.\.venv\Scripts\Activate.ps1`
- `mkdocs serve`
    - verwendet standardmäßig Port 3000
- generiert automatisch UML-Diagramme mit PlantUML
    - ER-Diagramm
    - Use-Case-Diagramm
    - Komponentendiagramm
    - Klassendiagramm
- Konfiguration
    - in `mkdocs.yml`
        - ``build_plantuml`` Attribut für plantuml Konfig  
        - output-Dateityp, Eingabe- und Ausgabepfad usw
    - in ``pyproject.toml``
        - Attribut ``doc``
        - Versionen usw
---
### Einschub: Indizes und balancierte Bäume
- **Lücke -> Notizen Flo**
- Baumstruktur ermöglicht lesen mit logarithmischem Aufwand
- erneutes ausbalancieren nach Schreiboperationen erhöht den Schreibaufwand
    - Bäume liegen auf der Platte -> ineffizient
    - <= 5 Indexe sinnvoll
- Fremdschlüssel berücksichtigen, weil die für JOINs relevant sind
---
- Klassendiagramm zeigt, dass Entity-Klasse vor anderen entwickelt/erstellt werden muss
- Anpassung der Diagramme in ``docs\diagramme\src``
- sowohl funktionale Konfiguration als auch optische Anpassungen möglich
### Use-Case-Diagramm
- `docs\diagramme\src\use-cases.plantuml`
- Preview in VSCode mit ``Alt+D``
### ER-Diagramm
- ``docs\diagramme\src\er-diagramm.plantuml``
- ``*`` = not null -> wird als schwarzer Punkt im Diagramm angezeigt 
### Klassendiagramm (Architektur)
- nur packages, keine Klassen -> für fastAPI nicht notwendig
- Nutze Klassendiagramm-Typ als Architekturdiagramm

## Datenbankzugriff / OR-Mapping
``src\flaskapp\entity\patient.py``
- doku-Kommentare mit drei `"""` werden in mkdocs-Doku übernommen 
- unnötige JOINs gefährlich
    - Daten werden unnötigerweise mit JOIN gelesen
    - Applikationsserver wird mit unnötigen Daten versorgt
    - unnötig von Festplatte gelesen, von DB zu Transport konvertiert, transportiert, von Applikation passend konvertiert, über REST-Schnittstelle angeboten, von App gelesen ... -> großer Overhead
- Hibernate macht JOINs bei 1:1-Beziehungen, aber keine automatischen 1:n JOINs, weil die risikobehaftet sind
    - heute andere Perspektive -> u.a. viel mehr Endgeräte und größere Serverbelastungen
    - andere Tools machen **alles** mit lazy-loading & lazy-fetching
        - bei reads wird nur die explizit im ``FROM`` benannte Tabelle gelesen -> keine automatischen JOINS o.ä.
        - Faustregel: **kein** eager fetching und lazy-fetching vermeiden
            - nicht eager -> keine automatischen JOINs
            - lazy kann zu vielen wiederholten Requests/Nachladen von einzelnen Objekte und Properties führen
            - 1 JOIN kann effizienter sein als 1+n Queries (*"Das n+1 select-Problem"*) -> als Entwickler strategisch in die Zukunft schauen
    - kein automatisches (eager) fetching in der entity klasse, sondern explizit definiertes eager fetching mit joinedLoad im patient_repository (Z. 64ff)
        ```python
            statement: Final = (
                select(Patient).options(joinedload(Patient.adresse)).where(Patient.id == id) # joinedLoad ist inner-JOIN
            )
            
        ```
## REST-Schnittstelle
`src\flaskapp\rest\patient_write_controller.py`
- Blueprint erstellen
- response methode mit docstring für doku versehen
- POST-Request von Clientseite -> Daten stehen im Request-Body
- Client muss über Veränderungen in der Datenbankstruktur informiert werden *(??)*
- GraphQL macht das anders -> später noch Anpassungen nötig
- ``POST``: ``location`` attribut im response-header übermittelt URI für neu angelegten Datensatz. Wert: `https://localhost:8000/rest/1000` mit statuscode 201:Created
- ``PUT``: zB vorhandenen Patienten ändern
    - Antwort bei Erfolg 204:No Content
- etags (Versionsnummer) beginnen und enden mit ``"`` -> kein String!
---
### Einschub Logging
- für zusammengesetzte Strings kein python fString (oder JS string-literale) (Bsp.: Zeile 73) -> fstring wird zur compile-time ausgewertet
- Lazy  Logging mit Loguru
    - ```python
        logger.debug("{} ({})", json_data, type(json_data))
        ```
- String wird erst dann gebaut, wenn es notwendig wird
---
- Z. 78 umwandeln json -> Patient-Objekt, danach mit create in businesslogik hinzufügen (Z80)
- return Response-Objekt "im Sinne von Flask, nicht ganz aber das klären wir noch" (z. 83ff)
- in ``\rest`` liegt ``validation_schemas.py``
    - für third-party lib Marshmallow
    - zB Schemas für Adresse, Patient, Kosten, etc.
    - Struktur gegeben durch Basisklasse Schema von Marshmallow
    - Definiert Struktur und zb notwendige Parameter, reguläre Ausdrücke usw
    - zB für plz: `plz = String(required=True, validate=Regexp("^[0-9]{5}$"))` (Z. 45)
    - Validierung mit ``load(json_data)`` (zB in Z. 76)
### Absicherung paralleler Zugriffe 
- Ziel: Verhinderung von Lost Updates
- oracle hat keine Lesesperren, Postgres ist vorkonfiguriert keine Lesesperren zu verwenden
- in der Entity-Klasse von Patient (`\entity\patient.py`)ist Attribut ``version``
- SQLAlchemy zählt Versionen hoch
- `update`- Funktion in `src\flaskapp\service\patient_write_service.py` prüft auf richtige Version (Z. 101ff)
- Request schickt Header mit: ``If-Match`` nach http-Spezifikation. Sinn: "Mach das Update, aber nur, wenn die Versionsnummer passt"
- im patient-controller ``src\flaskapp\rest\patient_write_controller.py`` wird ``If-Match``-Header aufgegriffen -> Z. 107ff.
    - genau eine Versionsnummer muss angegeben sein -> einelementige Menge
- Errors ggf im Controller catchen und dort durch passende HTTP-Fehlerantwort ersetzen
- `src\flaskapp\service\patient_write_service.py` Z. 101   -  wenn die Datenbankversion größer ist als die, die der Client mitgeschickt hat, besteht das Risiko für ein Lost-Update, deswegen wird die Aktion abgebrochen
- SQLAlchemy erhöht Version, aber ``patient_dto`` muss manuell erhöht werden -> service, Z. 119
- ermöglicht bedingte GET-Requests (liefere nur, wenn neuere Version verfügbar) -> sonst Antwort ``304:Not Modified`` ohne response body



bedingtes get -> if-none-match ; put -> if-match ; \+ etag im response header ; \+ location im response header 

# Vorlesung 31.05.
## Klausurinfos:
- kein Code, nur Verständnis
- Checkliste im Ilias
### REST
- Flask Blueprints
- was ist bei Schreiboperationen zu berücksichtigen
### HTTP-Methoden & Statuscodes
- insbes ein bestimtter angesprochener 3er code
### Schichtenarchitektur
- mittlere Schicht nicht im Detail behandelt
- zuerst validideren, erst dann bearbeiten
- Datenbankzugriff ausführlich!
    - wie weit liest man
    - inwieweit/wann führt man JOINs aus
        - keine "vorauseilenden" JOINs
        - SQLAlchemy default -> keine automatischen Joins
        - kein eager fetching und lazy fetching vermeiden (im Sinne von minimieren)
        - n+1 Problem -> 1 Datensatz wird gelesen und dann schritt für Schritt daten nachgeladen
        - Versionsspalte verwenden
        => REST-Schnittstelle kann Versionszähler verwenden -> ifNotMatch -> wenn Server sieht, dass es keine Änderung gab, wird NotModified zurückgegeben 
... -> Ben, Ioannis
## Vorlesung
- bei sehr **großen** Systemen und **vielen** Clients kann REST zu Problemen führen
    1. Clients fordern z.T. Daten an, die sie gar nicht unbedingt alle brauchen -> "Overfetching"
    2. Clients fordern Daten, bekommen zu wenig Daten und müssen dann weiteren Request abschicken, ggf an weiteren Endpoint -> "Underfetching", "Waterfall-Requests" weil Kettenreaktion, Requests plätschern beim Server ein
- Problem: REST ist "server-driven", Server gibt System vor und Clients müssen sich danach richten "take it, or leave it"
- viele verschiedene Endgerätetypen verschiedener (Bildschirm-)Größe -> unterschiedlich viel Inhalt anzeigbar
### GraphQL
- client-driven -> löst/minimiert Over- und Underfetching
- alternativ auch als Proxy vor einer REST-Schnittstelle nutzbar (normalerweise eigenständig)
- graphQL hat nur einen Endpoint, egal ob wir lesen oder schreiben
    - wird **immer** mit POST angesprochen
    - statuscode **immer** 200
#### Strawberry
- schema first
- code first
#### Schema
- neutrale Beschreibungssprache für Schnittstelle
- schema.graphql wird von strawberry generiert
- losgelöst von http
    - queries und mutations statt expliziter Erwähnung von http-statuscodes und rest-methoden
    - statische Endpoints nicht mehr notwendig
    - Statuscodes, Header,... uninteressant
- Queries
    - ``!`` hinter Objekt => nicht nullable
    - Trennung zwischen Kunde und KundeInput -> zwei verschiedene Datentypen
        - Typen die zurückzuliefernde Daten definieren
        - Typen die Eingabedaten definieren
- ``schema.py``
    - definiere Funktionen hinter queries

# Vorlesung 07.06.2024
## Erstellung Docker-Image -> Dockerfile
- cloud native buildpacks nur bei java + spring-boot sinnvoll => deswegen hier mit dockerfile
- ``flaskapp\Dockerfile``
- Z.36 ändern zu `ARG PYTHON_VERSION=3.12.3`
- ``FROM ...`` definiert ausgangs-image (parent image)
    - baue auf bestehendem image auf
    - in diesem Fall: ``python:${PYTHON_VERSION}-slim-bookworm``
        - Debian (Bookworm) + Python
        - slim = nur das notwendigste -> kleineres image
    - Quelle für images: https://hub.docker.com
- Docker nutzt wsl
- Dockerfile baut **zwei images** -> siehe Z. 41, Z.107
    - jedes ``FROM`` initiiert neue "Stage"
    - erste Stage = Builder
        - baue Image mit Software die für die Konfiguration benötigt wird
    - zweite Stage = endgültiger Zusammenbau 
        - entferne Software/Libraries, die zur Laufzeit nicht mehr notwendig sind
        - erstelle und wechsle zu non-root user
        - kopiere notwendige verzeichnisse aus dem ersten zum zweiten image
        - letzter Schritt: **führe bei containerstart flaskapp aus**
- `WORKDIR` = working directory für docker commands
- `RUN`-Befehl
    - ``RUN <<EOF`` = führe alle Linux Befehle bis EOF aus
    - Konfiguration der Linux-Installation möglich 
        - aktualisiere Abhängigkeiten (apt update und upgrade)
        - Python venv

## nach der Pause
- in `C:\Users\spams\Documents\Studium\h-ka\10_(SS24)\Python-Frameworks\flaskapp\.extras\compose\compose.yml` Zeile 41 image mit dem neuen Image ersetzen -> name wie vor der Pause von uns definiert
- ``cd \flaskapp\.extras\compose``;``docker compose up``
- in der `compose.yml` stehen Pfade 
    - zu Zertifikaten -> damit Zertifikat nicht direkt im (ggf öffentlichen) Image liegt
    - zu Konfigdateien -> ``flaskapp.toml``
