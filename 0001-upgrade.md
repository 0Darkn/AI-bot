---

# 🔐 Autenticação real por utilizador (Python)

## 🎯 Objectivos

* Registo de utilizadores
* Login seguro
* Password **hash + salt**
* Sessão activa
* Integração com:

  * memória por utilizador
  * FAISS
  * logs XML / JSON
  * multilinguagem
  * menus

---

## 📁 Estrutura recomendada

```
auth/
 ├─ auth.py
 ├─ users.json
 ├─ session.py
 └─ permissions.py
```

---

## 🔑 1. Armazenamento de utilizadores (JSON)

### 📄 `users.json`

```json
{
  "users": []
}
```

Nunca guardamos passwords em texto simples. **Nunca.**

---

## 🔐 2. Hash de passwords (seguro)

Vamos usar `hashlib + os.urandom`.

### 🔹 `auth.py`

```python
# auth.py
# Gestão de utilizadores: registo e login

import json
import os
import hashlib
import hmac

USERS_FILE = "auth/users.json"

def carregar_users():
    with open(USERS_FILE, "r", encoding="utf-8") as f:
        return json.load(f)

def guardar_users(dados):
    with open(USERS_FILE, "w", encoding="utf-8") as f:
        json.dump(dados, f, indent=2, ensure_ascii=False)

def hash_password(password, salt=None):
    """
    Cria hash seguro da password
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

def registar_user(username, password):
    dados = carregar_users()

    for u in dados["users"]:
        if u["username"] == username:
            return False, "Utilizador já existe"

    salt, pwd_hash = hash_password(password)

    dados["users"].append({
        "username": username,
        "salt": salt,
        "password": pwd_hash,
        "role": "user"
    })

    guardar_users(dados)
    return True, "Utilizador registado com sucesso"
```

---

## 🔓 3. Login seguro

```python
def autenticar_user(username, password):
    dados = carregar_users()

    for u in dados["users"]:
        if u["username"] == username:
            salt = bytes.fromhex(u["salt"])
            _, pwd_hash = hash_password(password, salt)

            if hmac.compare_digest(pwd_hash, u["password"]):
                return True, u

    return False, None
```

➡️ `hmac.compare_digest` evita ataques de timing.

---

## 🧾 4. Sessão activa (login persistente)

### 🔹 `session.py`

```python
# session.py
# Sessão activa do utilizador

SESSAO = {
    "logado": False,
    "username": None,
    "role": None
}

def iniciar_sessao(user):
    SESSAO["logado"] = True
    SESSAO["username"] = user["username"]
    SESSAO["role"] = user["role"]

def terminar_sessao():
    SESSAO["logado"] = False
    SESSAO["username"] = None
    SESSAO["role"] = None
```

---

## 🛂 5. Permissões (admin / user)

### 🔹 `permissions.py`

```python
# permissions.py

from session import SESSAO

def is_admin():
    return SESSAO["role"] == "admin"

def precisa_login():
    return SESSAO["logado"]
```

---

## 📋 6. Integração com MENU de comandos

### 🔹 `comandos.py` (exemplo)

```python
from auth.auth import autenticar_user, registar_user
from auth.session import iniciar_sessao, terminar_sessao
from auth.permissions import precisa_login

def processar_comando(texto):
    if texto.startswith("/registar"):
        _, user, pwd = texto.split()
        ok, msg = registar_user(user, pwd)
        return msg

    if texto.startswith("/login"):
        _, user, pwd = texto.split()
        ok, dados = autenticar_user(user, pwd)
        if ok:
            iniciar_sessao(dados)
            return f"Bem-vindo {user} 👋"
        return "Login inválido ❌"

    if texto == "/logout":
        terminar_sessao()
        return "Sessão terminada"

    if not precisa_login():
        return "⚠️ Necessita de login"

    return "Comando reconhecido"
```

---

## 🧠 7. Ligação à **memória por utilizador**

Agora tudo fica simples 👇

```python
from auth.session import SESSAO
from memory.user_memory import guardar_memoria_user

def guardar_contexto(texto):
    user = SESSAO["username"]
    guardar_memoria_user(user, texto)
```

➡️ Cada utilizador tem:

```
memory/user_joao/
memory/user_ana/
```

---

## 🧾 8. Logs com utilizador

### JSON

```python
log_json("mensagem", {
    "user": SESSAO["username"],
    "texto": texto
})
```

### XML

```python
log_xml("login", SESSAO["username"])
```

---

## 🔒 9. Segurança (boas práticas)

✔ Password nunca em texto simples
✔ Hash com salt
✔ Comparação segura
✔ Separação de módulos
✔ Fácil de migrar para SQLite depois

---

## 🚀 Próximos upgrades possíveis



1. 🔐 **SQLite em vez de JSON**
2. 🌐 Autenticação via **Flask (cookies / tokens)**
3. 🧠 **FAISS persistente por utilizador**
4. 🖥️ Login gráfico com **Qt**
5. 🔑 JWT / API tokens
6. 👥 Grupos e permissões avançadas


