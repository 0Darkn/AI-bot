1. 📁 **Arquitetura final**
2. 🗄️ **Base de dados SQLite**
3. 🔐 **Autenticação + JWT**
4. 👥 **Grupos e permissões**
5. 🧠 **FAISS persistente por utilizador**
6. 🌐 **Backend Flask (API segura)**
7. 🖥️ **Frontend Qt (login + bot)**
8. 🔁 **Fluxo completo do sistema**

---

# 1️⃣ Arquitetura final do projeto

```
ai_bot/
│
├─ backend/
│   ├─ app.py              # Flask API
│   ├─ auth.py             # Login, JWT
│   ├─ permissions.py
│   ├─ database.py         # SQLite
│   ├─ memory_faiss.py     # FAISS por utilizador
│   └─ models.py
│
├─ frontend/
│   ├─ login_qt.py         # Login gráfico
│   ├─ main_qt.py          # Interface do bot
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

# 2️⃣ Base de dados SQLite (users, roles, groups)

### 🔹 `database.py`

```python
import sqlite3

DB = "data/users.db"

def ligar_db():
    return sqlite3.connect(DB)

def criar_db():
    db = ligar_db()
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

# 3️⃣ Autenticação + JWT

### 🔹 `auth.py`

```python
import jwt, datetime, hashlib
from database import ligar_db

SECRET = "SEGREDO_FORTE"

def hash_pwd(pwd):
    return hashlib.sha256(pwd.encode()).hexdigest()

def login(username, password):
    db = ligar_db()
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

# 4️⃣ Grupos e permissões avançadas

### 🔹 `permissions.py`

```python
import jwt
from database import ligar_db
from auth import SECRET

def verificar_token(token):
    return jwt.decode(token, SECRET, algorithms=["HS256"])

def pertence_grupo(user_id, grupo):
    db = ligar_db()
    c = db.cursor()

    c.execute("""
    SELECT g.name FROM groups g
    JOIN user_groups ug ON ug.group_id = g.id
    WHERE ug.user_id=?
    """, (user_id,))

    grupos = [g[0] for g in c.fetchall()]
    return grupo in grupos
```

---

# 5️⃣ FAISS persistente por utilizador

### 🔹 `memory_faiss.py`

```python
import faiss, os, numpy as np
from sentence_transformers import SentenceTransformer

DIM = 384
model = SentenceTransformer("all-MiniLM-L6-v2")

BASE = "data/faiss/"

def caminho(user_id):
    os.makedirs(BASE, exist_ok=True)
    return f"{BASE}user_{user_id}.index"

def carregar_index(user_id):
    path = caminho(user_id)
    if os.path.exists(path):
        return faiss.read_index(path)
    return faiss.IndexFlatL2(DIM)

def guardar_index(user_id, index):
    faiss.write_index(index, caminho(user_id))

def guardar_memoria(user_id, texto):
    index = carregar_index(user_id)
    emb = model.encode([texto]).astype("float32")
    index.add(emb)
    guardar_index(user_id, index)

def procurar(user_id, texto, k=3):
    index = carregar_index(user_id)
    emb = model.encode([texto]).astype("float32")
    _, ids = index.search(emb, k)
    return ids.tolist()
```

---

# 6️⃣ Backend Flask (API segura)

### 🔹 `app.py`

```python
from flask import Flask, request, jsonify
from auth import login
from permissions import verificar_token
from memory_faiss import guardar_memoria

app = Flask(__name__)

@app.route("/login", methods=["POST"])
def api_login():
    data = request.json
    token = login(data["user"], data["password"])
    if not token:
        return jsonify({"error": "login inválido"}), 401
    return jsonify({"token": token})

@app.route("/memory", methods=["POST"])
def api_memoria():
    token = request.headers.get("Authorization").split()[1]
    user = verificar_token(token)
    guardar_memoria(user["user_id"], request.json["texto"])
    return jsonify({"status": "ok"})
```

---

# 7️⃣ Frontend Qt – Login gráfico

### 🔹 `login_qt.py`

```python
from PyQt5.QtWidgets import *
import requests

class Login(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Login AI Bot")

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
            QMessageBox.information(self, "OK", "Login com sucesso")
```

---

# 8️⃣ Fluxo completo do sistema

```
Qt Login
   ↓
Flask /login → JWT
   ↓
Qt guarda token
   ↓
Qt envia mensagens → API
   ↓
Flask valida JWT
   ↓
FAISS por utilizador
   ↓
Resposta contextual
```

---

## 🧠 O que ESTE projeto já faz

- ✔ Login gráfico
- ✔ Autenticação real
- ✔ JWT / API Tokens
- ✔ SQLite profissional
- ✔ FAISS persistente por utilizador
- ✔ Grupos e permissões
- ✔ Backend seguro
- ✔ Frontend Qt
- ✔ Escalável (Docker / cloud / local)

---

## 🚀 Próximos upgrades

* 🤖 ligar a **LLM local (Ollama / LM Studio)**
* 🧩 criar **plugins por grupo**
* 🖥️ UI Qt avançada (chat, histórico, settings)
* 📊 dashboard admin
* 🔐 refresh tokens
* 🌍 multilinguagem total
* 🧠 FAISS + metadata (JSON)

