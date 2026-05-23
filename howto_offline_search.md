# Búsqueda Multi-Fuente Offline — How To
### Sin LLM · Meilisearch + Embeddings · ThinkPad / Hardware bajo recurso

---

## Stack

| Componente | Software | RAM | Puerto |
|---|---|---|---|
| Motor de búsqueda | Meilisearch | ~200MB | 7700 |
| Embeddings semánticos | sentence-transformers (Python) | ~150MB | — |
| Wikipedia / ZIM | Kiwix-serve | ~100MB | 8888 |
| PDFs indexados | Extracción con pdfminer | — | — |
| Interfaz web | Meilisearch UI (incluida) | — | 7700 |

**RAM total estimada**: ~600MB activo. Corre en X220 con 8GB sin problema.

---

## 1. Instalar Debian 12 (base)

```bash
# Actualizar sistema
apt update && apt upgrade -y
apt install -y python3 python3-pip curl wget git nginx
```

---

## 2. Instalar Meilisearch

```bash
# Descargar e instalar
curl -L https://install.meilisearch.com | sh

# Mover a /usr/local/bin
mv ./meilisearch /usr/local/bin/

# Crear servicio systemd
cat > /etc/systemd/system/meilisearch.service << EOF
[Unit]
Description=Meilisearch
After=network.target

[Service]
ExecStart=/usr/local/bin/meilisearch --http-addr 0.0.0.0:7700 --db-path /var/lib/meilisearch
Restart=always
User=root

[Install]
WantedBy=multi-user.target
EOF

systemctl enable meilisearch
systemctl start meilisearch
```

---

## 3. Instalar dependencias Python (indexación)

```bash
pip3 install --break-system-packages \
  meilisearch \
  sentence-transformers \
  pdfminer.six \
  beautifulsoup4 \
  requests
```

El modelo de embeddings se descarga automáticamente la primera vez (~90MB):
```
paraphrase-multilingual-MiniLM-L12-v2
```
Entiende español, inglés y 50 idiomas más. No necesita GPU.

---

## 4. Indexar PDFs

Guarda este script como `/opt/indexer/index_pdfs.py`:

```python
import os
import meilisearch
from pdfminer.high_level import extract_text
from sentence_transformers import SentenceTransformer

# Configuración
PDF_DIR = "/content/pdf"
MEILI_URL = "http://localhost:7700"
INDEX_NAME = "docs"
CHUNK_SIZE = 500  # caracteres por fragmento

client = meilisearch.Client(MEILI_URL)
model = SentenceTransformer("paraphrase-multilingual-MiniLM-L12-v2")

# Crear índice si no existe
index = client.index(INDEX_NAME)

def chunk_text(text, size=CHUNK_SIZE):
    words = text.split()
    chunks = []
    current = []
    count = 0
    for word in words:
        current.append(word)
        count += len(word)
        if count >= size:
            chunks.append(" ".join(current))
            current = []
            count = 0
    if current:
        chunks.append(" ".join(current))
    return chunks

documents = []
doc_id = 0

for filename in os.listdir(PDF_DIR):
    if not filename.endswith(".pdf"):
        continue
    path = os.path.join(PDF_DIR, filename)
    print(f"Indexando: {filename}")
    try:
        text = extract_text(path)
        chunks = chunk_text(text)
        for i, chunk in enumerate(chunks):
            embedding = model.encode(chunk).tolist()
            documents.append({
                "id": doc_id,
                "source": filename,
                "type": "pdf",
                "page_hint": i + 1,
                "content": chunk,
                "embedding": embedding
            })
            doc_id += 1
    except Exception as e:
        print(f"  Error en {filename}: {e}")

# Subir a Meilisearch en lotes
batch_size = 100
for i in range(0, len(documents), batch_size):
    index.add_documents(documents[i:i+batch_size])
    print(f"  Subidos {min(i+batch_size, len(documents))}/{len(documents)}")

# Activar búsqueda vectorial
client.index(INDEX_NAME).update_settings({
    "embedders": {
        "custom": {
            "source": "userProvided",
            "dimensions": 384
        }
    }
})

print("✓ Indexación completada")
```

```bash
mkdir -p /opt/indexer
# Copiar script y ejecutar
python3 /opt/indexer/index_pdfs.py
```

---

## 5. Instalar Kiwix (Wikipedia offline)

```bash
# Descargar Kiwix tools
wget https://download.kiwix.org/release/kiwix-tools/kiwix-tools_linux-x86_64-3.7.0.tar.gz
tar xzf kiwix-tools_linux-x86_64-3.7.0.tar.gz
mv kiwix-tools_linux-x86_64-3.7.0/kiwix-serve /usr/local/bin/

# Directorio para ZIMs
mkdir -p /content/zim

# Descargar Wikipedia ES (texto, ~22GB)
# Desde: https://download.kiwix.org/zim/wikipedia/
# Ejemplo:
wget -P /content/zim \
  https://download.kiwix.org/zim/wikipedia/wikipedia_es_all-mini_2024-10.zim

# Servicio systemd
cat > /etc/systemd/system/kiwix.service << EOF
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

## 6. Servir PDFs con Nginx

```bash
# Crear enlace simbólico de los PDFs al servidor web
mkdir -p /var/www/html/docs
mount --bind /content/pdf /var/www/html/docs

# O simplemente copiar/enlazar:
ln -s /content/pdf /var/www/html/docs

# Habilitar autoindex en Nginx
cat > /etc/nginx/sites-available/docs << EOF
server {
    listen 80;
    server_name _;

    location /docs {
        alias /content/pdf;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
    }
}
EOF

ln -s /etc/nginx/sites-available/docs /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

---

## 7. Configurar hotspot WiFi

```bash
apt install -y hostapd dnsmasq

# Configurar hotspot
cat > /etc/hostapd/hostapd.conf << EOF
interface=wlan0
driver=nl80211
ssid=OfflineBox
hw_mode=g
channel=6
wpa=2
wpa_passphrase=doomsday2024
wpa_key_mgmt=WPA-PSK
EOF

cat > /etc/dnsmasq.conf << EOF
interface=wlan0
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
EOF

# IP estática para wlan0
cat >> /etc/network/interfaces << EOF
auto wlan0
iface wlan0 inet static
  address 192.168.4.1
  netmask 255.255.255.0
EOF

systemctl enable hostapd dnsmasq
systemctl start hostapd dnsmasq
```

---

## 8. Página de inicio (portal)

```bash
cat > /var/www/html/index.html << EOF
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>OfflineBox</title>
  <style>
    body { font-family: monospace; background: #111; color: #eee;
           max-width: 600px; margin: 60px auto; padding: 20px; }
    h1 { color: #f90; }
    a { color: #8cf; display: block; margin: 12px 0; font-size: 1.1em; }
    .tag { font-size: 0.8em; color: #888; margin-left: 8px; }
  </style>
</head>
<body>
  <h1>OfflineBox</h1>
  <a href="http://192.168.4.1:7700">🔍 Búsqueda multi-fuente <span class="tag">Meilisearch</span></a>
  <a href="http://192.168.4.1:8888">📖 Wikipedia offline <span class="tag">Kiwix</span></a>
  <a href="http://192.168.4.1/docs">📄 Documentos PDF <span class="tag">Nginx</span></a>
</body>
</html>
EOF
```

---

## 9. Verificar que todo funciona

```bash
systemctl status meilisearch
systemctl status kiwix
systemctl status nginx
systemctl status hostapd

# Test búsqueda vía API
curl -X POST 'http://localhost:7700/indexes/docs/search' \
  -H 'Content-Type: application/json' \
  --data '{"q": "hemorragia interna tratamiento", "limit": 3}'
```

---

## Acceso desde cualquier dispositivo

Conectar al WiFi **OfflineBox** y abrir el navegador:

| Servicio | URL |
|---|---|
| Portal de inicio | `http://192.168.4.1` |
| Búsqueda multi-fuente | `http://192.168.4.1:7700` |
| Wikipedia | `http://192.168.4.1:8888` |
| PDFs | `http://192.168.4.1/docs` |

---

## Añadir más documentos (mantenimiento)

```bash
# Copiar nuevos PDFs al directorio
cp nuevos_docs/*.pdf /content/pdf/

# Re-indexar (solo los nuevos si modificas el script para checkear IDs)
python3 /opt/indexer/index_pdfs.py

# Nuevos ZIMs (WikiMed, Appropedia, etc.)
wget -P /content/zim https://download.kiwix.org/zim/...
systemctl restart kiwix
```

---

## Resumen de recursos en reposo

| Servicio | RAM |
|---|---|
| Meilisearch | ~180MB |
| Kiwix-serve | ~90MB |
| Nginx | ~10MB |
| Hostapd + dnsmasq | ~15MB |
| OS Debian base | ~200MB |
| **Total** | **~500MB** |

Quedan libres ~7.5GB en un sistema con 8GB RAM.
Sin LLM, sin Docker, sin dependencias pesadas.
