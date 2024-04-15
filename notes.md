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