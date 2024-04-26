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