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
