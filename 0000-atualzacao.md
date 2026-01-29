## 🧠 1. AI Bot com **FAISS + Embeddings**

### 🎯 Objectivo

Dar **memória semântica** ao bot:

* recordar conversas
* procurar contexto por significado (não por texto exacto)
* responder melhor a longo prazo

### 📦 Dependências

```bash
pip install faiss-cpu sentence-transformers
```

### 📁 Estrutura

```
ai/
 ├─ embeddings.py
 ├─ memory_faiss.py
 └─ model.py
```

### 🔹 `embeddings.py`

```python
# embeddings.py
# Converte texto em vectores numéricos (embeddings)

from sentence_transformers import SentenceTransformer

# Modelo leve e eficaz
model = SentenceTransformer("all-MiniLM-L6-v2")

def texto_para_embedding(texto: str):
    """
    Recebe texto e devolve um vector numérico
    """
    return model.encode([texto])[0]
```

### 🔹 `memory_faiss.py`

```python
# memory_faiss.py
# Memória semântica com FAISS

import faiss
import numpy as np
from embeddings import texto_para_embedding

DIM = 384  # dimensão do modelo
index = faiss.IndexFlatL2(DIM)

memoria_texto = []  # guarda o texto associado ao vector

def guardar_memoria(texto):
    """
    Guarda texto + embedding no índice FAISS
    """
    emb = texto_para_embedding(texto)
    index.add(np.array([emb]).astype("float32"))
    memoria_texto.append(texto)

def procurar_memoria(pergunta, k=3):
    """
    Procura memórias relacionadas com a pergunta
    """
    emb = texto_para_embedding(pergunta)
    _, ids = index.search(np.array([emb]).astype("float32"), k)

    return [memoria_texto[i] for i in ids[0] if i < len(memoria_texto)]
```

---

## 👤 2. **Memória por Utilizador**

### 🎯 Objectivo

Cada utilizador tem:

* histórico próprio
* embeddings separados
* contexto independente

### 📁 Estrutura

```
memory/
 ├─ user_123/
 │   ├─ faiss.index
 │   └─ memory.json
```

### 🔹 Exemplo simples

```python
# user_memory.py

import json
import os

def caminho_user(user_id):
    pasta = f"memory/user_{user_id}"
    os.makedirs(pasta, exist_ok=True)
    return pasta

def guardar_memoria_user(user_id, texto):
    path = caminho_user(user_id) + "/memory.json"

    dados = []
    if os.path.exists(path):
        with open(path, "r", encoding="utf-8") as f:
            dados = json.load(f)

    dados.append(texto)

    with open(path, "w", encoding="utf-8") as f:
        json.dump(dados, f, ensure_ascii=False, indent=2)
```

➡️ **FAISS também pode ser um índice por utilizador** (ideal para bots grandes).

---

## 📋 3. **Comandos por Menu**

### 🎯 Objectivo

Criar comandos tipo:

```
/menu
/memoria
/idioma
/limpar
/logs
```

### 🔹 Dispatcher simples

```python
# comandos.py

def processar_comando(texto, user_id):
    if texto == "/menu":
        return mostrar_menu()

    elif texto == "/memoria":
        return "Memória activa ✔"

    elif texto.startswith("/idioma"):
        return "Idioma alterado"

    else:
        return None

def mostrar_menu():
    return (
        "📋 MENU\n"
        "/menu - mostrar menu\n"
        "/memoria - estado da memória\n"
        "/idioma pt|en\n"
        "/logs - ver logs\n"
    )
```

➡️ Isto funciona tanto em **CLI**, **GUI Qt**, **Telegram**, **Web**.

---

## 🌍 4. **Multilinguagem (i18n)**

### 🎯 Objectivo

Trocar idioma **sem alterar o código**.

### 📁 Estrutura

```
lang/
 ├─ pt.json
 ├─ en.json
```

### 🔹 `pt.json`

```json
{
  "welcome": "Bem-vindo!",
  "menu": "Menu principal",
  "memory_on": "Memória activa"
}
```

### 🔹 Loader

```python
# i18n.py

import json

IDIOMA_ATUAL = "pt"

def texto(chave):
    with open(f"lang/{IDIOMA_ATUAL}.json", "r", encoding="utf-8") as f:
        lang = json.load(f)
    return lang.get(chave, chave)
```

Uso:

```python
print(texto("welcome"))
```

➡️ Podes ligar isto a `/idioma en`.

---

## 🧾 5. **Logs em XML / JSON**

### 🎯 Objectivo

* auditoria
* debug
* histórico
* compatível com outras apps

---

### 🔹 Log em JSON

```python
# log_json.py

import json
from datetime import datetime

def log_json(evento, dados):
    log = {
        "timestamp": datetime.now().isoformat(),
        "evento": evento,
        "dados": dados
    }

    with open("logs/bot.json", "a", encoding="utf-8") as f:
        f.write(json.dumps(log, ensure_ascii=False) + "\n")
```

---

### 🔹 Log em XML

```python
# log_xml.py

import xml.etree.ElementTree as ET
from datetime import datetime

def log_xml(evento, mensagem):
    root = ET.Element("log")
    ET.SubElement(root, "time").text = datetime.now().isoformat()
    ET.SubElement(root, "evento").text = evento
    ET.SubElement(root, "mensagem").text = mensagem

    tree = ET.ElementTree(root)
    tree.write("logs/bot.xml", encoding="utf-8", xml_declaration=True)
```

---

## 🧩 Arquitetura Final Recomendada

```
bot/
 ├─ ai/
 ├─ memory/
 ├─ lang/
 ├─ logs/
 ├─ comandos.py
 ├─ i18n.py
 ├─ main.py
```

---

## 🚀 Próximos upgrades possíveis


* 🔗 integrar isto com **Qt GUI**
* 🌐 ligar a **Flask / Web**
* 🤖 ligar a **LLM local (Ollama, LM Studio)**
* 🧠 persistir FAISS em disco
* 🔐 adicionar autenticação real por utilizador


