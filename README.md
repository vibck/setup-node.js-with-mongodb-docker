# Teil 1: Setup für Node.js Backend mit MongoDB und Testdaten

Diese Anleitung beschreibt, wie du ein Node.js-Backend mit MongoDB einrichtest und Testdaten beim ersten Start hinzufügst. Wir nutzen Docker-Container für MongoDB und das Backend.

---

### 1. **Installiere die notwendigen Abhängigkeiten**
Erstelle einen neuen Ordner namens `api-backend`
Zuerst müssen die Pakete für Node.js und MongoDB installiert werden:

```bash
npm init -y
npm install express mongoose dotenv cors 
```

### 2. **Erstelle eine `.env`-Datei**

Erstelle eine `.env`-Datei im Stammverzeichnis des Projekts, um sensible Umgebungsvariablen zu speichern, z. B. die Verbindungsdaten zur MongoDB-Datenbank:

```env
MONGO_URI=mongodb://root:deinPasswort@mongo-container:27017/todos_db?authSource=admin
PORT=5000
```

### 3. **Erstelle die MongoDB-Verbindung und Testdatenlogik**

Erstelle eine Datei `database.js`, um die Verbindung zu MongoDB herzustellen und Testdaten einzufügen:

```javascript
require('dotenv').config();
const mongoose = require('mongoose');

// Datenbankschema für "todos"
const todoSchema = new mongoose.Schema({
    text: { type: String, required: true },
    isComplete: { type: Boolean, default: false },
});

const Todo = mongoose.model('Todo', todoSchema);

// MongoDB-Verbindung herstellen
async function connectToDatabase() {
    try {
        await mongoose.connect(process.env.MONGO_URI);
        console.log('Erfolgreich mit MongoDB verbunden');

        // Testdaten einfügen, falls noch keine Todos vorhanden sind
        const count = await Todo.countDocuments();
        if (count === 0) {
            console.log('Keine Todos gefunden. Testdaten werden hinzugefügt.');

            // Testdaten
            const todos = [
                { text: 'Python auffrischen', isComplete: false },
                { text: 'JavaScript üben', isComplete: false },
                { text: 'React lernen', isComplete: false },
            ];

            try {
                await Todo.insertMany(todos);
                console.log('Testdaten erfolgreich hinzugefügt');
            } catch (insertError) {
                console.error('Fehler beim Einfügen der Testdaten:', insertError.message);
            }
        }
    } catch (err) {
        console.error('Fehler beim Verbinden mit MongoDB:', err.message);
        process.exit(1);
    }
}

// Verbindung schließen, wenn der Prozess beendet wird
process.on('SIGINT', async () => {
    try {
        await mongoose.connection.close();
        console.log('MongoDB-Verbindung geschlossen');
        process.exit(0);
    } catch (err) {
        console.error('Fehler beim Schließen der MongoDB-Verbindung:', err.message);
        process.exit(1);
    }
});

module.exports = { connectToDatabase, Todo };
```

### 4. **Backend-Server erstellen (Express API)**

Erstelle die Datei `app.js` für dein Node.js-Backend mit Express. Hier werden die Routen definiert, um Todos zu erstellen, abzurufen, zu aktualisieren und zu löschen.

```javascript
const express = require('express');
const cors = require('cors');
const { connectToDatabase, Todo } = require('./database');
require('dotenv').config();

const app = express();
const port = process.env.PORT || 5000;

app.use(express.json());
app.use(cors());

// Verbinde zur MongoDB-Datenbank und füge Testdaten hinzu, falls nötig
connectToDatabase();

// API-Routen
app.get('/todos', async (req, res) => {
    try {
        const todos = await Todo.find();
        res.json(todos);
    } catch (err) {
        res.status(500).send('Fehler beim Abrufen der Todos');
    }
});

app.post('/todos', async (req, res) => {
    const { text, isComplete } = req.body;
    if (!text) {
        return res.status(400).send("Todo text cannot be empty");
    }

    try {
        const newTodo = new Todo({ text, isComplete });
        await newTodo.save();
        res.status(201).send('Todo created');
    } catch (err) {
        res.status(500).send('Fehler beim Erstellen eines Todos');
    }
});

app.put('/todos/:id', async (req, res) => {
    const { id } = req.params;
    const { text, isComplete } = req.body;

    if (!text) {
        return res.status(400).send("Todo text cannot be empty");
    }

    try {
        const updatedTodo = await Todo.findByIdAndUpdate(id, { text, isComplete }, { new: true });
        if (!updatedTodo) {
            return res.status(404).send('Todo not found');
        }
        res.status(200).send('Todo updated');
    } catch (err) {
        res.status(500).send('Fehler beim Aktualisieren des Todos');
    }
});

app.delete('/todos/:id', async (req, res) => {
    const { id } = req.params;

    try {
        const deletedTodo = await Todo.findByIdAndDelete(id);
        if (!deletedTodo) {
            return res.status(404).send('Todo not found');
        }
        res.status(200).send('Todo deleted');
    } catch (err) {
        res.status(500).send('Fehler beim Löschen des Todos');
    }
});

// Server starten
app.listen(port, () => {
    console.log(`Server läuft auf http://localhost:${port}`);
});

module.exports = { app };
```

### 5. **MongoDB-Container starten**

Starte einen MongoDB-Container mit folgendem Befehl:

```bash
docker run --name mongo-container \
  --network app-network \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=deinPasswort \
  -e MONGO_INITDB_DATABASE=todos_db \
  -p 27017:27017 \
  -d mongo:latest
```

### 6. **Backend-Container bauen und starten**

Baue und starte das Backend in einem Docker-Container:

```bash
docker buildx build -t node-backend .
docker run --name node-backend \
  --env-file .env \
  -p 5000:5000 \
  --network app-network \
  -d node-backend
```

### 7. **API testen**

Jetzt kannst du die API testen, indem du zu `http://localhost:5000/todos` gehst (z. B. über Postman oder den Browser).

- **GET `/todos`**: Zeigt alle Todos an.
- **POST `/todos`**: Erstelle ein neues Todo.
- **PUT `/todos/:id`**: Aktualisiere ein bestehendes Todo.
- **DELETE `/todos/:id`**: Lösche ein Todo.

---

### Zusammenfassung:

1. **Installiere die notwendigen Node.js-Abhängigkeiten**: Express, Mongoose, CORS, dotenv, etc.
2. **Erstelle die `.env`-Datei** mit der MongoDB-Verbindungs-URL und Port.
3. **Erstelle eine `database.js`**-Datei für die MongoDB-Verbindung und das Hinzufügen von Testdaten.
4. **Erstelle die Express API** in `app.js` für CRUD-Operationen.
5. **Starte den MongoDB-Container** über Docker.
6. **Starte den Backend-Container**, verbinde ihn mit MongoDB und stelle sicher, dass die Testdaten hinzugefügt werden.
7. **Teste die API** lokal über `http://localhost:5000/todos`.

Jetzt hast du ein Node.js-Backend, das mit MongoDB verbunden ist und beim ersten Start automatisch Testdaten hinzufügt.

----

# Teil 2: Environment Variablen nutzen
Erreiche, dass der folgende Befehl 

```bash
docker run --name mongo-container \
 --network app-network
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=deinPasswort \
  -e MONGO_INITDB_DATABASE=todos_db \
  -p 27017:27017 \
  -d mongo:latest
```

ohne 

```bash
-e MONGO_INITDB_ROOT_USERNAME=root \
-e MONGO_INITDB_ROOT_PASSWORD=deinPasswort \
-e MONGO_INITDB_DATABASE=todos_db \
```

korrekt ausgeführt werden kann. Dokumentiere deine Schritte dazu.
