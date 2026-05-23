# ThinkPad X220 — Internet in a Box (Doomsday Edition)

> Servidor de conocimiento offline + LLM local + audio · Guía de referencia rápida

---

## Hardware base

| Componente | Spec | Notas |
|---|---|---|
| CPU | i5-2520M / **i7-2640M** (preferible) | AVX pero no AVX2 — impacta velocidad LLM ~30% |
| RAM | **16GB DDR3** (2×8GB) | Máximo soportado; comprar 1 placa adicional si ya tienes 8GB |
| SSD | 512GB SATA III | OS + LLM + contenido caben en 512GB |
| Batería | 9 celdas (genérica ~20€) | ~5–7h de servidor activo |
| Consumo | ~7–12W en uso | Compatible con panel solar 20W + powerbank 20000mAh |
| Firmware | **Libreboot** recomendado | Elimina Intel ME, firmware 100% libre y auditable |

---

## Inversión estimada

> Asumiendo que ya dispones de: SSD 512GB + 8GB RAM (1 placa)

| Escenario | Componentes a comprar | Coste total |
|---|---|---|
| **Mínimo viable** | X220 + batería | ~45–65€ |
| **Recomendado** | X220 + 8GB RAM extra + batería | ~65–90€ |
| **Completo** | X220 i7 + 16GB RAM + batería + HDD 1TB externo | ~120–150€ |

Fuentes: Wallapop, eBay.es, Back Market.

---

## Arquitectura de almacenamiento

```
SSD 512GB (interno)
├── /          → Debian 12 OS          ~15 GB
├── /llm       → Modelos Ollama         ~5–10 GB
├── /whisper   → Modelos STT            ~300 MB
├── /content
│   ├── zim/   → Wikipedia + WikiMed   ~25–95 GB
│   └── pdf/   → Docs médicos/técnicos ~5–20 GB
├── /books     → Calibre library        ~5–10 GB
└── swap       → 8–16 GB (mejora LLM)

HDD externo USB (opcional, 1–2TB)
└── /backup    → Copia completa + contenido extra
```

---

## Stack de software

| Capa | Software | Instalación |
|---|---|---|
| OS | Debian 12 Server (sin escritorio) | ISO oficial |
| LLM runtime | **Ollama** | `curl -fsSL https://ollama.com/install.sh \| sh` |
| Interfaz web LLM | **Open WebUI** | Docker, puerto 3000 |
| Contenido enciclopédico | **Kiwix-serve** (ZIM) | apt / binario descargable |
| Servidor PDFs/HTML | **Nginx** | `apt install nginx` |
| Biblioteca libros | **Calibre Web** | pip / Docker |
| Voz → Texto | **Whisper.cpp** (`base` en español) | Compilar desde GitHub |
| Texto → Voz | **Piper TTS** (`es_ES-davefx-high`) | Binario descargable |
| Hotspot WiFi | **hostapd + dnsmasq** | `apt install hostapd dnsmasq` |
| Backup | **rsync** (cron diario) | Preinstalado en Debian |

---

## Modelos LLM recomendados para X220

| Modelo | Tamaño Q4 | RAM | Velocidad (X220) | Recomendación |
|---|---|---|---|---|
| `llama3.2:3b` | 1.9 GB | 4 GB libres | 3–5 tok/s | ✅ Mejor balance |
| `phi3:mini` | 2.3 GB | 4 GB libres | 3–5 tok/s | ✅ Muy bueno en español |
| `mistral:7b-q4` | 4.1 GB | 6 GB libres | 1–2 tok/s | ⚠️ Solo con 16GB RAM |
| `llama3.1:8b-q4` | 4.7 GB | 7 GB libres | <1 tok/s | ❌ Demasiado lento |

**Conclusión**: con X220 el modelo natural es `phi3:mini` o `llama3.2:3b`. La ausencia de AVX2 hace inviable cualquier modelo por encima de 7B.

---

## Audio (voz ↔ LLM)

```
Micrófono → Whisper.cpp (base, ~300MB) → texto → Ollama → respuesta → Piper TTS → altavoz
```

- Whisper modelo `base`: precisión suficiente, <500MB RAM adicional
- Piper TTS: respuesta en voz natural en español, ~50MB, sin GPU
- Open WebUI integra STT+TTS nativamente desde v0.3+

---

## Contenido offline: fuentes y tamaños

| Contenido | Fuente | Tamaño | Formato |
|---|---|---|---|
| Wikipedia ES (texto) | kiwix.org/zim | ~22 GB | .zim |
| Wikipedia ES (completa) | kiwix.org/zim | ~90 GB | .zim |
| WikiMed (medicina) | kiwix.org/zim | ~1.5 GB | .zim |
| Appropedia (tecnología) | kiwix.org/zim | ~3 GB | .zim |
| Where There Is No Doctor | hesperian.org | ~50 MB | PDF libre |
| Manual Merck | varias fuentes | ~200 MB | PDF |
| Mapas OSM por país | download.geofabrik.de | variable | .pbf / OsmAnd |
| Libros técnicos libres | gutenberg.org | variable | EPUB/PDF |

---

## Funcionalidades por inversión

| Funcionalidad | 65€ (mínimo) | 90€ (recomendado) | 150€ (completo) |
|---|:---:|:---:|:---:|
| Wikipedia offline (texto) | ✅ | ✅ | ✅ |
| Wikipedia completa (imágenes) | ❌ espacio | ✅ | ✅ |
| PDFs médicos y técnicos | ✅ | ✅ | ✅ |
| LLM chat (texto) | ✅ lento | ✅ fluido | ✅ fluido |
| Voz → LLM → Voz | ❌ | ✅ | ✅ |
| RAG sobre documentos propios | ❌ | ❌ | ✅ |
| Hotspot WiFi (otros dispositivos) | ✅ | ✅ | ✅ |
| Backup en HDD externo | ❌ | ❌ | ✅ |
| Off-grid (solar + powerbank) | Opcional | Opcional | ✅ |
| Libreboot (firmware libre) | Opcional | Opcional | ✅ recomendado |

---

## Comandos de inicio rápido

```bash
# Instalar Ollama y modelo recomendado
curl -fsSL https://ollama.com/install.sh | sh
ollama pull phi3:mini

# Open WebUI (interfaz web para el LLM)
docker run -d -p 3000:8080 \
  -v open-webui:/app/backend/data \
  --name open-webui \
  ghcr.io/open-webui/open-webui:main

# Kiwix (servidor Wikipedia)
wget https://download.kiwix.org/release/kiwix-tools/kiwix-tools_linux-x86_64.tar.gz
tar xzf kiwix-tools_linux-x86_64.tar.gz
./kiwix-serve --port=8888 /content/zim/*.zim &

# Nginx para PDFs
apt install -y nginx
ln -s /content/pdf /var/www/html/docs
# Acceder desde red local: http://[IP-thinkpad]/docs

# Hotspot WiFi
apt install -y hostapd dnsmasq
# Configurar /etc/hostapd/hostapd.conf con SSID y contraseña
```

---

## Acceso desde otros dispositivos

Una vez configurado el hotspot, cualquier dispositivo (móvil, tablet, otro portátil) conectado a la red WiFi del X220 puede acceder a:

| Servicio | URL |
|---|---|
| LLM chat (Open WebUI) | `http://192.168.4.1:3000` |
| Wikipedia offline (Kiwix) | `http://192.168.4.1:8888` |
| Documentos PDF | `http://192.168.4.1/docs` |
| Biblioteca libros (Calibre) | `http://192.168.4.1:8083` |

---

## Ventajas únicas del X220 para este proyecto

- **Libreboot compatible** — el único ThinkPad con firmware 100% libre estable
- **Piezas disponibles en AliExpress** por 5–15€ (pantallas, teclados, bisagras, placas)
- **Consumo ultrabajo** — viable con energía solar o powerbank
- **Teclado 7 filas clásico** — el más duradero de ThinkPad
- **Comunidad activa** — thinkwiki.org, r/thinkpad, Coreboot Wiki
- **Sin batería CMOS que falle** — el reloj se puede resetear manualmente

## Limitaciones conocidas

- **Sin AVX2**: LLM ~30–50% más lento que ThinkPads post-2013
- **Máx. 16GB RAM**: techo para modelos LLM; suficiente para 7B Q4
- **Pantalla 1366×768**: baja resolución, irrelevante para uso headless/servidor
- **Upgrade path**: si en 3–5 años el LLM necesita más, migrar SSD a T450 (~30€ entonces)

---

*Generado para proyecto Internet in a Box — Doomsday Edition*
