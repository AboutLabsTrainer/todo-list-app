# Codeänderung und Deployment mit Docker

Diese Anleitung zeigt zwei Wege, eine Änderung an der Todo-App sichtbar zu machen:

1. **Deployment über ein Image.** Der geänderte Code wird in ein neues Image gebaut und als neuer Container gestartet.
2. **Hot Folder fürs Frontend.** Der Ordner `src/static` liegt auf dem Rechner und wird in den laufenden Container gemountet. Eine Änderung erscheint nach dem Neuladen im Browser, ohne neues Image.

Die App läuft unter [http://localhost:3000](http://localhost:3000). Aufgaben liegen in MySQL, im Volume `todo-mysql-data`. Ein neues App-Image oder ein neuer App-Container behält diese Daten.

Voraussetzung ist Docker Desktop. Alle Befehle im Projektordner `todo-list-app` ausführen.

## Gemeinsamer Ausgangszustand

MySQL starten und den Compose-App-Container anhalten, damit Port 3000 frei ist. Der Service `app` in `compose.yaml` mountet den ganzen Projektordner und startet `nodemon`. Für die beiden Wege unten bleibt nur MySQL aus Compose übrig.

```powershell
docker compose up -d
docker compose stop app
```

Prüfen:

```powershell
docker compose ps
```

`mysql` steht auf `running`, `app` auf `exited`.

Die App nutzt MySQL, sobald `MYSQL_HOST` gesetzt ist (`src/persistence/index.js`). Die folgenden Container bekommen deshalb dieselben Werte wie in `compose.yaml`:

| Variable | Wert |
| --- | --- |
| `MYSQL_HOST` | `mysql` |
| `MYSQL_USER` | `root` |
| `MYSQL_PASSWORD` | `secret` |
| `MYSQL_DB` | `todos` |

Compose legt das Netzwerk `todo-list-app_default` an. Der Name lässt sich mit `docker network ls` prüfen. Weicht er ab, in den `docker run`-Befehlen den tatsächlichen Namen einsetzen.

## Weg 1: Änderung ins Image deployen

Das `Dockerfile` kopiert den Quellcode ins Image und startet `node src/index.js`. Eine Änderung liegt erst im Container, nachdem das Image neu gebaut und der Container ersetzt wurde.

### 1. Sichtbare Version einbauen

In `src/static/index.html` den Seitentitel setzen:

```html
<title>Todo App v1</title>
```

### 2. Image bauen und Container starten

```powershell
docker build -t todo-list-app:v1 .
docker run -d --name todo-v1 --network todo-list-app_default -p 127.0.0.1:3000:3000 -e MYSQL_HOST=mysql -e MYSQL_USER=root -e MYSQL_PASSWORD=secret -e MYSQL_DB=todos todo-list-app:v1
```

Der erste Build kann länger dauern, weil der Build-Kontext auch `node_modules` enthält.

[http://localhost:3000](http://localhost:3000) öffnen. Der Browsertab heißt **Todo App v1**. Eine Aufgabe anlegen, damit später sichtbar ist, dass die Daten den Containerwechsel überstehen.

### 3. Code ändern

In `src/static/index.html`:

```html
<title>Todo App v2</title>
```

Die Seite neu laden. Der Tab heißt weiter **Todo App v1**, weil der laufende Container den Stand aus dem Image `todo-list-app:v1` ausliefert.

### 4. Neue Version deployen

```powershell
docker build -t todo-list-app:v2 .
docker stop todo-v1
docker rm todo-v1
docker run -d --name todo-v2 --network todo-list-app_default -p 127.0.0.1:3000:3000 -e MYSQL_HOST=mysql -e MYSQL_USER=root -e MYSQL_PASSWORD=secret -e MYSQL_DB=todos todo-list-app:v2
```

Neu laden. Der Tab heißt **Todo App v2**. Die vorher angelegte Aufgabe ist noch da.

```powershell
docker images todo-list-app
```

zeigt `todo-list-app:v1` und `todo-list-app:v2`.

Vor Weg 2 den Container aus Weg 1 entfernen, damit Port 3000 wieder frei ist:

```powershell
docker stop todo-v2
docker rm todo-v2
```

## Weg 2: Hot Folder fürs Frontend

Express liefert `src/static` bei jedem Request von der Platte aus (`src/index.js`). Liegt dieser Ordner als Volume im Container, ersetzt er die Frontend-Dateien aus dem Image. Speichern und die Seite neu laden reichen. Der Node-Prozess bleibt derselbe Container.

Backend-Dateien wie `src/index.js` und die Routen unter `src/routes` stammen weiter aus dem Image. Dafür gilt Weg 1.

### 1. Image mit Hot Folder starten

Hier das Image aus Weg 1. Der Titel im Image kann `Todo App v2` sein. Der Mount legt die Dateien vom Rechner darüber.

```powershell
docker run -d --name todo-hot --network todo-list-app_default -p 127.0.0.1:3000:3000 -e MYSQL_HOST=mysql -e MYSQL_USER=root -e MYSQL_PASSWORD=secret -e MYSQL_DB=todos -v "${PWD}/src/static:/app/src/static" todo-list-app:v1
```

`${PWD}/src/static` auf dem Host ist `/app/src/static` im Container.

### 2. Frontend ändern und neu laden

In `src/static/js/app.js` den Platzhalter des Eingabefelds ändern (etwa Zeile 99):

```jsx
placeholder="Neue Aufgabe"
```

[http://localhost:3000](http://localhost:3000) neu laden. Das Feld zeigt **Neue Aufgabe**. Es ist kein `docker build` und kein neuer Container nötig.

Optional denselben Effekt mit dem Seitentitel prüfen. In `src/static/index.html`:

```html
<title>Todo App Hot Folder</title>
```

Neu laden. Der Browsertab übernimmt den neuen Titel sofort.

Zeigt der Browser noch die alte Datei, die Seite hart neu laden (Strg+F5).

### 3. Grenze des Hot Folders zeigen

In `src/index.js` eine Zeile in der Log-Ausgabe ändern, zum Beispiel `Listening on port 3000` zu `Listening on port 3000 (hot)`. Die Logs ändern sich nicht:

```powershell
docker logs todo-hot
```

Diese Datei liegt im Image und ist nicht gemountet. Sie wird erst mit Weg 1 wirksam: Image neu bauen, Container `todo-hot` durch einen neuen Container ersetzen.

## Aufräumen

```powershell
docker stop todo-hot
docker rm todo-hot
docker compose down
```

`docker compose down -v` löscht zusätzlich das MySQL-Volume und damit die Aufgaben.
