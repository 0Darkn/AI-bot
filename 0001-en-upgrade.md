---

# 🔐 Real Per-User Authentication (Python)

## 🎯 Goals

* User registration
* Secure login
* Password **hash + salt**
* Active session
* Integration with:

  * per-user memory
  * FAISS
  * XML / JSON logs
  * multilingual support
  * menus

---

## 📁 Recommended Structure

```
auth/
 ├─ auth.py
 ├─ users.json
 ├─ session.py
 └─ permissions.py
```

---

## 🔑 1. User Storage (JSON)

### 📄 `users.json`

```json
{
  "users": []
}
```

Passwords are **never** stored in plain text. **Never.**

---

## 🔐 2. Secure Password Hashing

We will use `hashlib + os.urandom`.

### 🔹 `auth.py`

```python
# auth.py
# User management: registration and login

import json
import os
import hashlib
import hmac

USERS_FILE = "auth/users.json"

def load_users():
    with open(USERS_FILE, "r", encoding="utf-8") as f:
        return json.load(f)

def save_users(data):
    with open(USERS_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=2, ensure_ascii=False)

def hash_password(password, salt=None):
    """
    Creates a secure password hash
    """
    if not salt:
        salt = os.urandom(16)

    pwd_hash = hashlib.pbkdf2_hmac(
        "sha256",
        password.encode(),
        salt,
        100_000
    )
    return salt.hex(), pwd_hash.hex()

def register_user(username, password):
    data = load_users()

    for u in data["users"]:
        if u["username"] == username:
            return False, "User already exists"

    salt, pwd_hash = hash_password(password)

    data["users"].append({
        "username": username,
        "salt": salt,
        "password": pwd_hash,
        "role": "user"
    })

    save_users(data)
    return True, "User successfully registered"
```

---

## 🔓 3. Secure Login

```python
def authenticate_user(username, password):
    data = load_users()

    for u in data["users"]:
        if u["username"] == username:
            salt = bytes.fromhex(u["salt"])
            _, pwd_hash = hash_password(password, salt)

            if hmac.compare_digest(pwd_hash, u["password"]):
                return True, u

    return False, None
```

➡️ `hmac.compare_digest` prevents timing attacks.

---

## 🧾 4. Active Session (persistent login)

### 🔹 `session.py`

```python
# session.py
# Active user session

SESSION = {
    "logged_in": False,
    "username": None,
    "role": None
}

def start_session(user):
    SESSION["logged_in"] = True
    SESSION["username"] = user["username"]
    SESSION["role"] = user["role"]

def end_session():
    SESSION["logged_in"] = False
    SESSION["username"] = None
    SESSION["role"] = None
```

---

## 🛂 5. Permissions (admin / user)

### 🔹 `permissions.py`

```python
# permissions.py

from session import SESSION

def is_admin():
    return SESSION["role"] == "admin"

def login_required():
    return SESSION["logged_in"]
```

---

## 📋 6. Integration with Command MENU

### 🔹 `commands.py` (example)

```python
from auth.auth import authenticate_user, register_user
from auth.session import start_session, end_session
from auth.permissions import login_required

def process_command(text):
    if text.startswith("/register"):
        _, user, pwd = text.split()
        ok, msg = register_user(user, pwd)
        return msg

    if text.startswith("/login"):
        _, user, pwd = text.split()
        ok, data = authenticate_user(user, pwd)
        if ok:
            start_session(data)
            return f"Welcome {user} 👋"
        return "Invalid login ❌"

    if text == "/logout":
        end_session()
        return "Session ended"

    if not login_required():
        return "⚠️ Login required"

    return "Command recognized"
```

---

## 🧠 7. Linking to **Per-User Memory**

Now everything becomes simple 👇

```python
from auth.session import SESSION
from memory.user_memory import save_user_memory

def save_context(text):
    user = SESSION["username"]
    save_user_memory(user, text)
```

➡️ Each user has:

```
memory/user_john/
memory/user_anna/
```

---

## 🧾 8. Logs with User Information

### JSON

```python
log_json("message", {
    "user": SESSION["username"],
    "text": text
})
```

### XML

```python
log_xml("login", SESSION["username"])
```

---

## 🔒 9. Security (Best Practices)

✔ Passwords never stored in plain text
✔ Hashing with salt
✔ Secure comparison
✔ Clear module separation
✔ Easy to migrate to SQLite later

---

## 🚀 Possible Next Upgrades

1. 🔐 **SQLite instead of JSON**
2. 🌐 Authentication via **Flask (cookies / tokens)**
3. 🧠 **Persistent FAISS per user**
4. 🖥️ Graphical login with **Qt**
5. 🔑 JWT / API tokens
6. 👥 Groups and advanced permissions

---
