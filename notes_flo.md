# Hilfreiche Commands & Vorlesungsnotizen

## Docker

image identifiziert über `tag`, zB `postgres:16.2-bookworm`
- ``docker inspect tag``
- ``docker sbom tag`` (software bill of materials)
- ``dive.exe tag``
    - ermöglicht das tiefe "eintauchen" in die Struktur des image
    - Darstellung über Schichten & Baumstruktur
    - drei Sichten, wechseln mit Tab
    - Befehle in Leiste unten mit Strg
- ``C:\Zimmermann\dive\dive.exe tag``
    - drei schichten
    - punkt da wo man gerade focus hat (tab zum wechseln)
    - Layers sieht man zuerst unterste Schicht -> höhere Schichten nach unten lesen
    - Leertaste um ordner einzuklappen in rechter Ansicht
- ``docker compose up`` (in ordner mit compose.yml) um Container zu starten
  - ``-d`` oder ``--detach`` um Container von Shell zu lösen und im Hintergrund laufen zu lassen
- ``docker compose exec db bash`` (db ist der servicename der in der compose.yml angegeben wurde)
    - führt z.b. bash in postgres aus

# Vorlesung 1

### image starten
- `docker compose`
- ``compose.yml``/``docker-compose.yaml`` zur Spezifikation von Services
    - gültige JSON-Datei -> gültige YAML-Datei
    - YAML ist Obermenge von JSON
    - Einrückung wichtig (immer um 2)
    - Baumstruktur
    - ports-attribut -> Verbindung ports Windows und Linux-Image sozusagen Portweiterleitung von Container
- ``docker compose exec 'servicename' 'befehl'``
    - führe Befehl in image aus
    - zb ``docker compose exec db bash``


# Vorlesung 2

- Primärschlüssel müssen eindeutig sein und sollten sich nicht ändern -> zb email ungeeignet
- komplexere Datemsätze als Attribut abbilden:
    - JSON Objekt zur Simulation von String-Arrays
        - varchar
        - zweite Tabelle mit 1:n oder m:n Beziehung via JOINs
- ``.\.venv\Scripts\Activate.ps1`` ausführen für virtual environment
- siehe ReadMe.md ab Zeile 774
- backend starten python -m flaskapp im flaskapp ordner
- Startreihenfolge ``.\.venv\Scripts\Activate.ps1``, ``python -m flaskapp``, ``docker compose up``
- Controller für patient in ``src/flaskapp/rest/patient_get_controller.py durchgegangen``
    - Annotations bzw. Dekorators:
        - ``@ctrl.get`` definiert die Route
        - ``@roles_allowed`` definiert erlaubte Rollen -> sonst Unauthorized oder Forbidden

# Vorlesung 3 (26.04)
- Asynchron programmiert mit asgi3 (asgi Version 3)
- In der Main wird das Module flask_app mit asgi3 benötigt
- main -> asgi_app -> flask_app
- `FLASK_APP: Final = Flask(__name__, static_folder=None)` Baut das
finale Flask mit dem Namen `FLASK_APP`
- `rest: Final = Blueprint(name="rest", import_name=__name__, url_prefix="/rest")` Baut eine Blaupause mit dem Namen
Rest und dem URL_prefix `/rest`
- Es werden noch unter Blueprints registriert:
```python
rest.register_blueprint(blueprint=get_ctrl)
rest.register_blueprint(blueprint=write_ctrl)
FLASK_APP.register_blueprint(rest)
```
- `ctrl: Final = Blueprint("PatientGetController", __name__)` ist eine "art" Dependency Injection (Er findet den Begriff gut aber nicht ganz passend)
- `@ctrl.get("/<int:id>")` kann anschließend als Decorator/Annotation verwendet werden, 
um Methoden zu sagen, das sie nun auf GET Requests hören.
- `patient: Final = await find_by_id(id=id, user_dto=user_dto)` asynchroner abstieg/Aufruf einer Asynchronen Funktion. 
Die Funktion mit diesem Aufruf muss ebenfalls asynchron sein. Abstieg von Controller zum Service.
- `email: Mapped[str] = mapped_column(unique=True)` definition im Entity für die hier eindeutige
Mail-Adresse (`unique=True`)
- SQLAlchemy ist der Python ORM-Mapper, wir müssen uns also keine Gedanken über die Datentypen in der SQL Datenbank machen,
egal ob Postgres, MySQL, MSSQL oder SQLite (**Nice to Know**: SQLite wird mit jeder Python Installation mitgeliefert)
- `id: Mapped[int | None] = mapped_column(Identity(start=1000), primary_key=True)` ist Definition für Primary Key. 
Dieser Startet bei 1000 und mit `Identity` wird der SQL Befehl `GENERATED AS IDENTITY`
```sql
CREATE TABLE IF NOT EXISTS patient (
    id            integer GENERATED ALWAYS AS IDENTITY(START WITH 1000) PRIMARY KEY USING INDEX TABLESPACE patientspace,
    .
    .
    .
```
- ENUM für Geschlecht, aber ohne ENUM in sql? -> Varchar mit Check constrain
- Ausschreiben vom Geschlecht in der SQL-Tabelle:
  - Bessere Lesbarkeit
  - Sonst Verwechslungsgefahr mit beispielsweise Wochentagen
  - Durch Indexing wird die Performance verbessert
- 1:N Beziehungen:
```python
kosten: Mapped[list["Kosten"]] = relationship(
      back_populates="patient", cascade="save-update, delete"
  )
"""Die in einer 1:N-Beziehung referenzierten Kosten."""
```
- Relational würde bedeuten, das es eine Reihenfolge gibt.
- Joins und Kaskaden:
```
adresse: Mapped["Adresse"] = relationship(
  back_populates="patient", innerjoin=True, cascade="save-update, delete"
  )
```
- `innerjoin=True`: Beim Verknüpfen wird nur ein Inner Join verwendet, ansonsten ein Outer Join
- `cascade=save-update, delete`: Patient hat Adresse (transitive abhängigkeit). Beim CREATE wird die Adresse
durch das cascade mit dem Patienten angelegt.
- HTTP Header `If-None-Match="0"` kann zu performance Verbesserungen genutzt werden → Falls Version in der DB
nicht höher als `If-None-Match`, wird Status `304: Not Modified` zurückgegeben.
- Beispiel für List Comprehensions:
  ```python
    self.fachaerzte_json = (
            [facharzt_enum.name for facharzt_enum in fachaerzte]
            if fachaerzte is not None
            else None
        )
    ```
- Beispiel für Walrus-Operator:
  ```python
  >>> walrus = False
  >>> walrus
  
  False
  
  >>> (walrus := True)
  
  True
  
  >>> walrus
  
  True
  ```
- Case Insensitive `.filter(Patient.nachname.ilike(f"%{teil}%"))`

# Vorlesung 4 (10.05)

### Docker Postgres Image Update
- Neues Postgres Update mit der Version `16.3`
  - In der Postgres compose.yml die Version abändern `image: postgres:16.3-bookworm`
  - Docker pull ausführen: 
  ```shell
    docker pull postgres:16.3-bookworm
  ```
  - Danach Docker Container starten:
  ```shell
    docker compose up
  ```
  - Je nach Postgres Version muss eventuell Migration ausgeführt werden → Online nachschlagen, 
  wie die Migration aussieht
  - ``View a summary of image vulnerabilities and recommendations → docker scout quickview postgres:16.3-bookworm``
  - ``postgres/data`` löschen falls man kein Bock hat auf Migration

### MKDocs für Anwendungsdokumentation
- ``mkdocs serve`` in Virtual-Environment für Webserver für Dokumentation mit mkdocs
  - ``INFO    -  [11:55:43] Watching paths for changes: 'docs', 'mkdocs.yml', 'src\flaskapp'``
    - ``mkdocs.yml`` ist Zentrale Config Datei für mkdocs
    - ``docs`` ist unterverzeichnis
    - Webserver erreichbar über ``http://127.0.0.1:8000/``
    - ``mkdocs`` generiert unter anderem ER-Diagramm der Datenbank
### Index in einer DB
- Ist eine Spalte ``unique`` so wird ein Unique-Index eingefügt
  - Aufwand geht von ``O(n)`` zu ``O(log n)``
  - Standard ist B+-Baum (Balancierter Baum)
  - Index wird auf der Festplatte gespeichert
  - B-Bäume sind Flacher im Vergleich zu Binärbäumen
    - Weg von der Wurzel zu den Blättern ist kürzer
    - Fördert schnelleres Lesen
  - Mehr als 5 Indizes machen keinen Sinn (Faustregel nach Zimmermann)
    - Index-Dateien müssen bei SQL-Insert/Update/Delete aktualisiert werden → Das ist teuer
    - Baum erneut ausbalancieren kann viel Rechenaufwand und Zeit kosten
    - Joins sind teuer → Fremdschlüsselspalten häufig als Indexe definieren
  - Dokumentation benutzt PlantUML, aber hatte kein Bock das mitzuschreiben

### Datenbank in Python
- Unnötige Joins sind blöd (Für die Aussage hat er bisschen länger gebraucht)
- Um das zu verhindern, benutzen die meisten ORM (Bis auf Hibernate) Lazy-Fetching
  - Lesen nur aus der anfänglichen Tabelle, für Object mit Fremdeintrag wird leeres Object als
  Deckplatte eingesetzt
  - Faustregel ala Zimmermann: kein eager fetching (also keine automatischen JOINs) und lazy-fetching vermeiden
  - ``joinedload(Patient.adresse)`` macht Eager-Fetching in ``patient_repository``, da es für diesen
  Use-Case am besten ist.
- Bei ``POST`` für Patient wird im Header unter ``location`` eine URI zum Eintrag des Patienten zurückgegeben.
- Body bleibt somit leer, sinnvoll dann Statuscode ``201 - Created`` zurückzugeben
- Wird bei einem ``PUT`` überschrieben, so wird der Statuscode ``204 - No Contend`` zurück gegeben,
weil der Client die ID des Patienten schon kennt.
- Im Header ``etag`` wird Versionsnummer mitgegeben, für Lost-Update

### Logging im Projekt

- Es wird Loguru als Logging-Framework verwendet
- Allgemein wird hier auch wieder auf Lazy-Logging zurückgegriffen. Es gibt auch hier verschiedene
Loglevel

### Lost-Update
- Oracle hat keine Lese-Sperren und Postgres ist Standardmäßig nur mit Schreibsperren konfiguriert.
- SQLAlchemy übernimmt Update-Funktion für die Versions-Spalte
- Bei Schreibvorgang wird überprüft, ob Versionsnummer höher als momentan vorhandener Wert in der DB ist
- Versionsnummer steht im Entity-Object
- Im Header wird Feld ``if-Match`` gesetzt, mit der gelesenen Versionsnummer mitgeschickt. Stimmt
diese Version mit der Version in der DB überein, wird Update ausgeführt.
- Jeder Header-Schlüssel kann mehrfach vorkommen, deshalb ``if-Match`` als Set auslesen, überprüfen
ob Länge > 1 sollte das der Fall sein, Exception werfen
  ```python
    if_match_set: Final = request.if_match.as_set()
        if len(if_match_set) != 1:
            return Response(
                status=HTTPStatus.PRECONDITION_REQUIRED, content_type=APPLICATION_JSON
            )
  ```
  Sollte die Version gleich der DB Version sein, so kann der Eintrag mit ``async def update(...)`` geändert werden.
  Beim Result muss die Versionsnummer auch um eins erhöht werden, da die aktualisierte Version in der DB erst nach commit verfügbar ist.
- Status-Code bei nicht Änderung ist ``304 - Not Modified``



