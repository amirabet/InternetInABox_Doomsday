# Búsqueda Semántica Offline — How To Completo
### Meilisearch Hybrid + Embeddings + Extractive QA · Sin LLM · ThinkPad / Hardware bajo recurso

---

## Qué construye este how-to

```
Pregunta en lenguaje natural ("hueso roto")
              ↓
    Modelo de embeddings (~90MB)
    convierte a vector semántico
              ↓
  Meilisearch Hybrid Search
  (textual + semántico simultáneo)
  busca en Wikipedia, PDFs, docs...
              ↓
  Top 5 fragmentos más relevantes
              ↓
  Extractive QA (~400MB)
  subraya la frase exacta de cada fuente
              ↓
  Interfaz web: pregunta + respuestas citadas
```

**RAM total**: ~800MB activo. Sin LLM. Sin generación. Sin alucinaciones.

---

## Stack completo

| Componente | Software | Licencia | RAM |
|---|---|---|---|
| Motor búsqueda híbrida | Meilisearch CE | MIT | ~200MB |
| Embeddings semánticos | sentence-transformers | Apache 2.0 | ~150MB |
| Extractive QA | deepset/roberta-multilingual-squad2 | Apache 2.0 | ~400MB |
| Wikipedia offline | Kiwix-serve | GPL-3.0 | ~90MB |
| Servidor docs | Nginx | BSD-2-Clause | ~10MB |
| Interfaz web | Flask + HTML | BSD | ~30MB |
| Hotspot WiFi | hostapd + dnsmasq | GPL-2.0 | ~15MB |

---

## 0. Preparación del sistema

```bash
# Sistema base
apt update && apt upgrade -y
apt install -y python3 python3-pip curl wget git nginx build-essential

# Directorios del proyecto
mkdir -p /content/{pdf,zim,html}
mkdir -p /opt/{indexer,qa,web}
mkdir -p /var/lib/meilisearch
```

---

## 1. Instalar Meilisearch

```bash
curl -L https://install.meilisearch.com | sh
mv ./meilisearch /usr/local/bin/

cat > /etc/systemd/system/meilisearch.service << 'EOF'
[Unit]
Description=Meilisearch
After=network.target

[Service]
ExecStart=/usr/local/bin/meilisearch \
  --http-addr 0.0.0.0:7700 \
  --db-path /var/lib/meilisearch \
  --env development
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

systemctl enable meilisearch
systemctl start meilisearch

# Verificar
curl http://localhost:7700/health
# → {"status":"available"}
```

---

## 2. Instalar dependencias Python

```bash
pip3 install --break-system-packages \
  meilisearch \
  sentence-transformers \
  transformers \
  torch \
  pdfminer.six \
  beautifulsoup4 \
  flask \
  requests

# Predescargar los modelos (solo la primera vez, requiere internet)
python3 -c "
from sentence_transformers import SentenceTransformer
from transformers import pipeline

print('Descargando modelo de embeddings...')
SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')

print('Descargando modelo QA...')
pipeline('question-answering', model='deepset/xlm-roberta-base-squad2')

print('Modelos descargados y cacheados.')
"
# Los modelos quedan en ~/.cache/huggingface (~500MB total)
# A partir de aquí todo funciona sin internet
```

---

## 3. Configurar Meilisearch para búsqueda híbrida

```bash
# Activar embedder externo (recibe los vectores que generamos nosotros)
curl -X PATCH 'http://localhost:7700/indexes/docs/settings' \
  -H 'Content-Type: application/json' \
  --data '{
    "embedders": {
      "semantic": {
        "source": "userProvided",
        "dimensions": 384
      }
    },
    "searchableAttributes": ["content", "source", "title"],
    "displayedAttributes": ["id", "source", "title", "content", "type", "page"]
  }'
```

---

## 4. Script de indexación

Guarda como `/opt/indexer/indexer.py`:

```python
#!/usr/bin/env python3
"""
Indexador multi-fuente: PDFs + HTML + texto plano
Genera embeddings semánticos y sube a Meilisearch
"""

import os
import sys
import hashlib
import json
import meilisearch
from sentence_transformers import SentenceTransformer
from pdfminer.high_level import extract_text
from bs4 import BeautifulSoup

# ── Configuración ──────────────────────────────────────────────
MEILI_URL    = "http://localhost:7700"
INDEX_NAME   = "docs"
EMBED_MODEL  = "paraphrase-multilingual-MiniLM-L12-v2"
CHUNK_SIZE   = 400   # caracteres por fragmento
CHUNK_OVERLAP = 80   # solapamiento entre fragmentos

SOURCES = {
    "pdf":  "/content/pdf",
    "html": "/content/html",
}

# ── Inicialización ─────────────────────────────────────────────
client = meilisearch.Client(MEILI_URL)
index  = client.index(INDEX_NAME)
model  = SentenceTransformer(EMBED_MODEL)

print(f"Modelo cargado: {EMBED_MODEL}")

# ── Utilidades ─────────────────────────────────────────────────
def chunk_text(text, size=CHUNK_SIZE, overlap=CHUNK_OVERLAP):
    """Divide el texto en fragmentos solapados para mejor cobertura."""
    chunks = []
    start = 0
    text = " ".join(text.split())  # normalizar espacios
    while start < len(text):
        end = start + size
        chunks.append(text[start:end])
        start += size - overlap
    return [c for c in chunks if len(c.strip()) > 50]

def file_hash(path):
    """Hash del archivo para detectar cambios."""
    with open(path, "rb") as f:
        return hashlib.md5(f.read()).hexdigest()[:8]

def extract_pdf(path):
    try:
        return extract_text(path)
    except Exception as e:
        print(f"  ⚠ Error PDF {path}: {e}")
        return ""

def extract_html(path):
    try:
        with open(path, "r", encoding="utf-8", errors="ignore") as f:
            soup = BeautifulSoup(f.read(), "html.parser")
            for tag in soup(["script", "style", "nav", "header", "footer"]):
                tag.decompose()
            return soup.get_text(separator=" ")
    except Exception as e:
        print(f"  ⚠ Error HTML {path}: {e}")
        return ""

# ── Indexación ─────────────────────────────────────────────────
def index_directory(path, filetype):
    documents = []
    extensions = {
        "pdf": [".pdf"],
        "html": [".html", ".htm"],
    }

    files = [
        os.path.join(root, f)
        for root, _, files in os.walk(path)
        for f in files
        if any(f.lower().endswith(ext) for ext in extensions[filetype])
    ]

    print(f"\n📂 {filetype.upper()}: {len(files)} archivos en {path}")

    for i, filepath in enumerate(files):
        filename = os.path.basename(filepath)
        print(f"  [{i+1}/{len(files)}] {filename}")

        text = extract_pdf(filepath) if filetype == "pdf" else extract_html(filepath)
        if not text.strip():
            continue

        chunks = chunk_text(text)
        print(f"    → {len(chunks)} fragmentos")

        for j, chunk in enumerate(chunks):
            embedding = model.encode(chunk, normalize_embeddings=True).tolist()
            doc_id = f"{file_hash(filepath)}_{j}"
            documents.append({
                "id":        doc_id,
                "source":    filename,
                "title":     filename.replace("_", " ").replace("-", " ").rsplit(".", 1)[0],
                "type":      filetype,
                "page":      j + 1,
                "content":   chunk,
                "_vectors":  {"semantic": embedding}
            })

        # Subir en lotes de 50
        if len(documents) >= 50:
            index.add_documents(documents)
            print(f"    ✓ Subidos {len(documents)} fragmentos")
            documents = []

    # Subir resto
    if documents:
        index.add_documents(documents)
        print(f"  ✓ Subidos {len(documents)} fragmentos finales")


# ── Main ───────────────────────────────────────────────────────
if __name__ == "__main__":
    for filetype, path in SOURCES.items():
        if os.path.isdir(path):
            index_directory(path, filetype)
        else:
            print(f"⚠ Directorio no encontrado: {path}")

    print("\n✅ Indexación completada")
    stats = index.get_stats()
    print(f"   Total documentos indexados: {stats.number_of_documents}")
```

```bash
chmod +x /opt/indexer/indexer.py
python3 /opt/indexer/indexer.py
```

---

## 5. Módulo de Extractive QA

Guarda como `/opt/qa/qa.py`:

```python
#!/usr/bin/env python3
"""
Extractive QA: dado una pregunta y un fragmento de texto,
extrae la frase exacta que responde la pregunta.
Sin generación. Sin alucinaciones.
"""

from transformers import pipeline

# Cargar modelo una sola vez al importar
_qa_pipeline = None

def get_pipeline():
    global _qa_pipeline
    if _qa_pipeline is None:
        _qa_pipeline = pipeline(
            "question-answering",
            model="deepset/xlm-roberta-base-squad2",
            tokenizer="deepset/xlm-roberta-base-squad2"
        )
    return _qa_pipeline

def extract_answer(question: str, context: str) -> dict:
    """
    Devuelve la frase exacta del contexto que responde la pregunta.
    
    Returns:
        {
            "answer": "texto extraído del documento",
            "score": 0.87,   # confianza 0-1
            "start": 142,    # posición en el contexto
            "end": 198
        }
    """
    qa = get_pipeline()
    try:
        result = qa(question=question, context=context, max_answer_len=200)
        return result
    except Exception:
        return {"answer": "", "score": 0.0, "start": 0, "end": 0}

def extract_answers_from_results(question: str, results: list) -> list:
    """
    Aplica QA extractivo sobre una lista de fragmentos de Meilisearch.
    Descarta respuestas con score < 0.15.
    """
    enriched = []
    for r in results:
        qa_result = extract_answer(question, r["content"])
        if qa_result["score"] > 0.15 and qa_result["answer"].strip():
            enriched.append({
                **r,
                "answer":       qa_result["answer"],
                "answer_score": round(qa_result["score"], 3)
            })
    # Ordenar por confianza del QA
    enriched.sort(key=lambda x: x["answer_score"], reverse=True)
    return enriched
```

---

## 6. Interfaz web

Guarda como `/opt/web/app.py`:

```python
#!/usr/bin/env python3
from flask import Flask, request, jsonify, render_template_string
import meilisearch
import sys
sys.path.insert(0, "/opt/qa")
from qa import extract_answers_from_results
from sentence_transformers import SentenceTransformer

app = Flask(__name__)
client = meilisearch.Client("http://localhost:7700")
index  = client.index("docs")
model  = SentenceTransformer("paraphrase-multilingual-MiniLM-L12-v2")

HTML = """
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>OfflineBox</title>
<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body {
    font-family: 'Georgia', serif;
    background: #0f0f0f;
    color: #e8e8e8;
    min-height: 100vh;
    padding: 40px 20px;
  }
  .container { max-width: 760px; margin: 0 auto; }
  h1 { color: #c8a96e; font-size: 1.6em; margin-bottom: 4px; letter-spacing: 1px; }
  .subtitle { color: #666; font-size: 0.85em; margin-bottom: 32px; }
  .search-bar {
    display: flex;
    gap: 10px;
    margin-bottom: 32px;
  }
  input[type=text] {
    flex: 1;
    padding: 14px 18px;
    background: #1a1a1a;
    border: 1px solid #333;
    border-radius: 6px;
    color: #eee;
    font-size: 1em;
    font-family: inherit;
    outline: none;
    transition: border-color 0.2s;
  }
  input[type=text]:focus { border-color: #c8a96e; }
  button {
    padding: 14px 24px;
    background: #c8a96e;
    color: #0f0f0f;
    border: none;
    border-radius: 6px;
    font-size: 1em;
    font-weight: bold;
    cursor: pointer;
    transition: background 0.2s;
  }
  button:hover { background: #e0c080; }
  .result {
    background: #161616;
    border: 1px solid #2a2a2a;
    border-left: 3px solid #c8a96e;
    border-radius: 6px;
    padding: 18px 20px;
    margin-bottom: 16px;
  }
  .result-header {
    display: flex;
    justify-content: space-between;
    align-items: baseline;
    margin-bottom: 10px;
  }
  .source-name { color: #c8a96e; font-size: 0.9em; font-weight: bold; }
  .badge {
    font-size: 0.75em;
    padding: 2px 8px;
    border-radius: 10px;
    background: #222;
    color: #888;
  }
  .answer {
    background: #1e1e14;
    border-left: 3px solid #e0c040;
    padding: 10px 14px;
    margin-bottom: 10px;
    color: #f0e080;
    font-style: italic;
    border-radius: 0 4px 4px 0;
    font-size: 0.95em;
  }
  .context {
    color: #999;
    font-size: 0.85em;
    line-height: 1.6;
  }
  .no-results { color: #666; text-align: center; padding: 40px; }
  .loading { color: #c8a96e; text-align: center; padding: 40px; display: none; }
  .links { margin-top: 10px; font-size: 0.8em; }
  .links a { color: #5a9; text-decoration: none; margin-right: 12px; }
  .links a:hover { text-decoration: underline; }
  .score { color: #555; font-size: 0.75em; }
  .quick-links {
    display: flex;
    gap: 12px;
    margin-bottom: 32px;
    flex-wrap: wrap;
  }
  .quick-links a {
    color: #666;
    text-decoration: none;
    font-size: 0.85em;
    padding: 6px 12px;
    border: 1px solid #2a2a2a;
    border-radius: 4px;
    transition: all 0.2s;
  }
  .quick-links a:hover { color: #c8a96e; border-color: #c8a96e; }
</style>
</head>
<body>
<div class="container">
  <h1>⬡ OfflineBox</h1>
  <p class="subtitle">Búsqueda semántica offline · Pregunta en lenguaje natural</p>

  <div class="quick-links">
    <a href="http://192.168.4.1:8888" target="_blank">📖 Wikipedia</a>
    <a href="http://192.168.4.1/docs" target="_blank">📄 Documentos</a>
  </div>

  <div class="search-bar">
    <input type="text" id="q" placeholder="ej: hueso roto, agua contaminada, motor que no arranca..."
           onkeydown="if(event.key==='Enter') search()">
    <button onclick="search()">Buscar</button>
  </div>

  <div class="loading" id="loading">Buscando...</div>
  <div id="results"></div>
</div>

<script>
async function search() {
  const q = document.getElementById('q').value.trim();
  if (!q) return;
  document.getElementById('loading').style.display = 'block';
  document.getElementById('results').innerHTML = '';
  try {
    const res = await fetch('/search', {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({q})
    });
    const data = await res.json();
    render(data.results, q);
  } catch(e) {
    document.getElementById('results').innerHTML =
      '<p class="no-results">Error de conexión</p>';
  }
  document.getElementById('loading').style.display = 'none';
}

function render(results, q) {
  const el = document.getElementById('results');
  if (!results || results.length === 0) {
    el.innerHTML = '<p class="no-results">Sin resultados para "' + q + '"</p>';
    return;
  }
  el.innerHTML = results.map(r => `
    <div class="result">
      <div class="result-header">
        <span class="source-name">📄 ${r.title || r.source}</span>
        <span class="badge">${r.type} · p.${r.page || '—'}</span>
      </div>
      ${r.answer ? `<div class="answer">"${r.answer}"
        <span class="score"> · confianza ${Math.round(r.answer_score*100)}%</span>
      </div>` : ''}
      <div class="context">${highlight(r.content, r.answer)}</div>
      <div class="links">
        <a href="/docs/${encodeURIComponent(r.source)}" target="_blank">Ver documento completo →</a>
      </div>
    </div>
  `).join('');
}

function highlight(text, answer) {
  if (!answer) return text.substring(0, 300) + '...';
  const idx = text.indexOf(answer);
  if (idx === -1) return text.substring(0, 300) + '...';
  const start = Math.max(0, idx - 80);
  const end   = Math.min(text.length, idx + answer.length + 80);
  let snippet = (start > 0 ? '...' : '') + text.substring(start, end) + (end < text.length ? '...' : '');
  return snippet.replace(answer, `<strong style="color:#f0e080">${answer}</strong>`);
}
</script>
</body>
</html>
"""

@app.route("/")
def home():
    return render_template_string(HTML)

@app.route("/search", methods=["POST"])
def search():
    data = request.json
    q = data.get("q", "").strip()
    if not q:
        return jsonify({"results": []})

    # Generar embedding de la pregunta
    q_vector = model.encode(q, normalize_embeddings=True).tolist()

    # Búsqueda híbrida: textual + semántica simultánea
    search_results = index.search(q, {
        "limit": 8,
        "hybrid": {
            "semanticRatio": 0.6,   # 60% semántico, 40% textual
            "embedder": "semantic"
        },
        "vector": q_vector,
        "attributesToRetrieve": ["id", "source", "title", "type", "page", "content"]
    })

    hits = search_results.get("hits", [])

    # Aplicar Extractive QA sobre los fragmentos recuperados
    enriched = extract_answers_from_results(q, hits)

    # Si QA no extrae nada útil, devolver igual los fragmentos
    if not enriched:
        enriched = hits

    return jsonify({"results": enriched[:5]})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=False)
```

---

## 7. Servicio systemd para la interfaz web

```bash
cat > /etc/systemd/system/offlinebox.service << 'EOF'
[Unit]
Description=OfflineBox Web Interface
After=meilisearch.service

[Service]
ExecStart=/usr/bin/python3 /opt/web/app.py
Restart=always
User=root
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
EOF

systemctl enable offlinebox
systemctl start offlinebox
```

---

## 8. Instalar Kiwix (Wikipedia offline)

```bash
wget https://download.kiwix.org/release/kiwix-tools/kiwix-tools_linux-x86_64-3.7.0.tar.gz
tar xzf kiwix-tools_linux-x86_64-3.7.0.tar.gz
mv kiwix-tools_linux-x86_64-3.7.0/kiwix-serve /usr/local/bin/

# Descargar contenido ZIM (requiere internet, una sola vez)
# Wikipedia ES texto (~22GB):
wget -P /content/zim \
  "https://download.kiwix.org/zim/wikipedia/wikipedia_es_all-mini_2024-10.zim"

# WikiMed medicina (~1.5GB):
wget -P /content/zim \
  "https://download.kiwix.org/zim/wikipedia/wikipedia_es_medicine_2024-10.zim"

cat > /etc/systemd/system/kiwix.service << 'EOF'
[Unit]
Description=Kiwix Server
After=network.target

[Service]
ExecStart=/usr/local/bin/kiwix-serve --port=8888 --address=0.0.0.0 /content/zim
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl enable kiwix
systemctl start kiwix
```

---

## 9. Nginx para servir PDFs

```bash
cat > /etc/nginx/sites-available/offlinebox << 'EOF'
server {
    listen 80;
    server_name _;

    # Portal principal → interfaz web
    location / {
        proxy_pass http://127.0.0.1:5000;
    }

    # PDFs navegables
    location /docs {
        alias /content/pdf;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        charset utf-8;
    }
}
EOF

rm -f /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/offlinebox /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

---

## 10. Hotspot WiFi

```bash
apt install -y hostapd dnsmasq

cat > /etc/hostapd/hostapd.conf << 'EOF'
interface=wlan0
driver=nl80211
ssid=OfflineBox
hw_mode=g
channel=6
wpa=2
wpa_passphrase=offline2024
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP
EOF

cat > /etc/dnsmasq.conf << 'EOF'
interface=wlan0
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
address=/#/192.168.4.1
EOF

ip addr add 192.168.4.1/24 dev wlan0 2>/dev/null || true

cat >> /etc/network/interfaces << 'EOF'
auto wlan0
iface wlan0 inet static
  address 192.168.4.1
  netmask 255.255.255.0
EOF

systemctl enable hostapd dnsmasq
systemctl start hostapd dnsmasq
```

---

## 11. Verificación del sistema completo

```bash
# Estado de todos los servicios
for s in meilisearch kiwix offlinebox nginx hostapd; do
  status=$(systemctl is-active $s)
  echo "$s: $status"
done

# Test búsqueda semántica directa
curl -s -X POST http://localhost:7700/indexes/docs/search \
  -H 'Content-Type: application/json' \
  --data '{"q": "hueso roto", "limit": 3}' | python3 -m json.tool | head -30

# Test interfaz web
curl -s -X POST http://localhost:5000/search \
  -H 'Content-Type: application/json' \
  --data '{"q": "hemorragia interna"}'
```

---

## Acceso desde otros dispositivos

Conectar al WiFi **OfflineBox** (contraseña: `offline2024`) y abrir el navegador:

| Servicio | URL |
|---|---|
| **Portal + búsqueda semántica** | `http://192.168.4.1` |
| Wikipedia / WikiMed | `http://192.168.4.1:8888` |
| PDFs navegables | `http://192.168.4.1/docs` |

---

## Añadir documentos nuevos

```bash
# Copiar PDFs nuevos
cp *.pdf /content/pdf/

# Re-indexar (detecta y salta los ya indexados si añades lógica de hash)
python3 /opt/indexer/indexer.py

# Reiniciar interfaz (recarga el modelo QA)
systemctl restart offlinebox
```

---

## Recursos en reposo

| Servicio | RAM aprox |
|---|---|
| Meilisearch | ~180 MB |
| OfflineBox (Flask + modelos) | ~420 MB |
| Kiwix-serve | ~90 MB |
| Nginx | ~10 MB |
| hostapd + dnsmasq | ~15 MB |
| OS Debian base | ~200 MB |
| **Total** | **~915 MB** |

Con 8GB RAM disponibles: **~7GB libres** para el sistema operativo y swap.

---

## Parámetro clave: semanticRatio

En `/opt/web/app.py`, línea `"semanticRatio": 0.6`:

| Valor | Comportamiento |
|---|---|
| `0.0` | Solo textual (palabras exactas) |
| `0.5` | Balance textual + semántico |
| `0.6` | **Recomendado**: predomina semántico |
| `1.0` | Solo semántico (puede perder términos técnicos exactos) |

Para contenido médico/técnico con mucha terminología específica, `0.6` es el balance óptimo: entiende "hueso roto" → "fractura" pero también respeta búsquedas exactas como "ibuprofeno 600mg".
