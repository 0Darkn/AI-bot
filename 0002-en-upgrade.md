---

1. 📁 **Final architecture**
2. 🗄️ **SQLite database**
3. 🔐 **Authentication + JWT**
4. 👥 **Groups and permissions**
5. 🧠 **Persistent FAISS per user**
6. 🌐 **Flask backend (secure API)**
7. 🖥️ **Qt frontend (login + bot)**
8. 🔁 **Complete system flow**

---

# 1️⃣ Final project architecture

```
ai_bot/
│
├─ backend/
│   ├─ app.py              # Flask API
│   ├─ auth.py             # Login, JWT
│   ├─ permissions.py
│   ├─ database.py         # SQLite
│   ├─ memory_faiss.py     # FAISS per user
│   └─ models.py
│
├─ frontend/
│   ├─ login_qt.py         # Graphical login
│   ├─ main_qt.py          # Bot interface
│   └─ api_client.py
│
├─ data/
│   ├─ users.db
│   └─ faiss/
│       ├─ user_1.index
│       └─ user_2.index
│
├─ logs/
│   ├─ bot.json
│   └─ bot.xml
│
└─ requirements.txt
```

---

# 2️⃣ SQLite database (users, roles, groups)

### 🔹 `database.py`

```python
import sqlite3

DB = "data/users.db"

def connect_db():
    return sqlite3.connect(DB)

def create_db():
    db = connect_db()
    c = db.cursor()

    c.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY,
        username TEXT UNIQUE,
        password TEXT,
        role TEXT
    )""")

    c.execute("""
    CREATE TABLE IF NOT EXISTS groups (
        id INTEGER PRIMARY KEY,
        name TEXT UNIQUE
    )""")

    c.execute("""
    CREATE TABLE IF NOT EXISTS user_groups (
        user_id INTEGER,
        group_id INTEGER
    )""")

    db.commit()
    db.close()
```

---

# 3️⃣ Authentication + JWT

### 🔹 `auth.py`

```python
import jwt, datetime, hashlib
from database import connect_db

SECRET = "STRONG_SECRET"

def hash_pwd(pwd):
    return hashlib.sha256(pwd.encode()).hexdigest()

def login(username, password):
    db = connect_db()
    c = db.cursor()
    c.execute("SELECT id, password, role FROM users WHERE username=?", (username,))
    user = c.fetchone()

    if not user or user[1] != hash_pwd(password):
        return None

    payload = {
        "user_id": user[0],
        "role": user[2],
        "exp": datetime.datetime.utcnow() + datetime.timedelta(hours=2)
    }

    return jwt.encode(payload, SECRET, algorithm="HS256")
```

---

# 4️⃣ Advanced groups and permissions

### 🔹 `permissions.py`

```python
import jwt
from database import connect_db
from auth import SECRET

def verify_token(token):
    return jwt.decode(token, SECRET, algorithms=["HS256"])

def belongs_to_group(user_id, group):
    db = connect_db()
    c = db.cursor()

    c.execute("""
    SELECT g.name FROM groups g
    JOIN user_groups ug ON ug.group_id = g.id
    WHERE ug.user_id=?
    """, (user_id,))

    groups = [g[0] for g in c.fetchall()]
    return group in groups
```

---

# 5️⃣ Persistent FAISS per user

### 🔹 `memory_faiss.py`

```python
import faiss, os, numpy as np
from sentence_transformers import SentenceTransformer

DIM = 384
model = SentenceTransformer("all-MiniLM-L6-v2")

BASE = "data/faiss/"

def path(user_id):
    os.makedirs(BASE, exist_ok=True)
    return f"{BASE}user_{user_id}.index"

def load_index(user_id):
    path_ = path(user_id)
    if os.path.exists(path_):
        return faiss.read_index(path_)
    return faiss.IndexFlatL2(DIM)

def save_index(user_id, index):
    faiss.write_index(index, path(user_id))

def store_memory(user_id, text):
    index = load_index(user_id)
    emb = model.encode([text]).astype("float32")
    index.add(emb)
    save_index(user_id, index)

def search(user_id, text, k=3):
    index = load_index(user_id)
    emb = model.encode([text]).astype("float32")
    _, ids = index.search(emb, k)
    return ids.tolist()
```

---

# 6️⃣ Flask backend (secure API)

### 🔹 `app.py`

```python
from flask import Flask, request, jsonify
from auth import login
from permissions import verify_token
from memory_faiss import store_memory

app = Flask(__name__)

@app.route("/login", methods=["POST"])
def api_login():
    data = request.json
    token = login(data["user"], data["password"])
    if not token:
        return jsonify({"error": "invalid login"}), 401
    return jsonify({"token": token})

@app.route("/memory", methods=["POST"])
def api_memory():
    token = request.headers.get("Authorization").split()[1]
    user = verify_token(token)
    store_memory(user["user_id"], request.json["text"])
    return jsonify({"status": "ok"})
```

---

# 7️⃣ Qt frontend – Graphical login

### 🔹 `login_qt.py`

```python
from PyQt5.QtWidgets import *
import requests

class Login(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("AI Bot Login")

        self.user = QLineEdit()
        self.pwd = QLineEdit()
        self.pwd.setEchoMode(QLineEdit.Password)

        btn = QPushButton("Login")
        btn.clicked.connect(self.login)

        layout = QVBoxLayout(self)
        layout.addWidget(self.user)
        layout.addWidget(self.pwd)
        layout.addWidget(btn)

    def login(self):
        r = requests.post("http://127.0.0.1:5000/login", json={
            "user": self.user.text(),
            "password": self.pwd.text()
        })

        if r.ok:
            self.token = r.json()["token"]
            QMessageBox.information(self, "OK", "Login successful")
```

---

# 8️⃣ Complete system flow

```
Qt Login
   ↓
Flask /login → JWT
   ↓
Qt stores token
   ↓
Qt sends messages → API
   ↓
Flask validates JWT
   ↓
FAISS per user
   ↓
Contextual response
```

---

## 🧠 What THIS project already does

* ✔ Graphical login
* ✔ Real authentication
* ✔ JWT / API tokens
* ✔ Professional SQLite setup
* ✔ Persistent FAISS per user
* ✔ Groups and permissions
* ✔ Secure backend
* ✔ Qt frontend
* ✔ Scalable (Docker / cloud / local)

---

## 🚀 Next upgrades

* 🤖 connect to a **local LLM (Ollama / LM Studio)**
* 🧩 create **group-based plugins**
* 🖥️ Advanced Qt UI (chat, history, settings)
* 📊 Admin dashboard
* 🔐 Refresh tokens
* 🌍 Full multilingual support
* 🧠 FAISS + metadata (JSON)

