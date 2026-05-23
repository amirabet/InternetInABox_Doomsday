# 🗃️ Internet in a Box — Guía Completa para Doomsday Prepper
### ThinkPad + LLM Local + Contenido Offline · Rangos de precio: 100–300€

---

## 🧠 Stack de Software Recomendado (independiente del presupuesto)

### Contenido offline
| Capa | Software | Formato | Notas |
|---|---|---|---|
| Wikipedia | **Kiwix + ZIM** | `.zim` (HTML comprimido) | Wikipedia ES ~22GB (texto), ~90GB (con imágenes) |
| Docs médicos | Kiwix WikiMed + PDFs | `.zim` / `.pdf` | WikiMed ~1.5GB, añadir PDFs de Merck, ATLS, etc. |
| Libros útiles | Calibre + OPDS | `.epub` / `.pdf` | Calibre actúa como servidor web de libros |
| Mapas | OsmAnd / OSMAnd+ maps | `.obf` | Mapas offline de OSM, útiles sin red |
| Servidor HTTP | **Nginx** | — | Sirve HTML/PDF estáticos desde cualquier carpeta |

### LLM Local
| Opción | Software | Ventajas | RAM mínima |
|---|---|---|---|
| ⭐ Recomendado | **Ollama + Open WebUI** | Un comando, interfaz web bonita, API REST | 6GB libres |
| Más ligero | **llama.cpp + servidor** | Binario puro, sin dependencias, portable | 4GB libres |
| Puro navegador | **WebLLM (MLC)** | Corre en browser vía WebGPU/WASM | 6GB VRAM (lento sin GPU) |
| Histórico/robusto | **GPT4All desktop** | App de escritorio, fácil de usar | 8GB RAM total |

> **Conclusión LLM**: Ollama + Open WebUI es el stack ideal. WebLLM no es práctico en CPU sin GPU — demasiado lento. Mejor Ollama con API accesible desde la red local.

### Audio (voz ↔ LLM)
| Función | Software | Modelo recomendado | RAM |
|---|---|---|---|
| Voz → Texto (STT) | **Whisper.cpp** | `tiny` o `base` en español | ~150–300MB |
| Texto → Voz (TTS) | **Piper TTS** | `es_ES-davefx-high` | ~50MB |
| Pipeline completo | **Open WebUI** (integra ambos) | — | Integrado |

### OS recomendado
- **Debian 12 Bookworm** (sin escritorio, servidor headless) → máxima estabilidad, mínimos recursos
- O **Ubuntu 24.04 Server** si quieres más soporte de paquetes recientes
- **NO** Windows: demasiado consumo de RAM y actualizaciones forzadas

---

## 📦 Modelos LLM recomendados para 8–16GB RAM

| Modelo | Tamaño (Q4) | RAM necesaria | Velocidad (CPU) | Ideal para |
|---|---|---|---|---|
| `phi3:mini` (3.8B) | 2.3 GB | 4 GB | ⚡⚡⚡ Rápido | Respuestas cortas, bajo recurso |
| `llama3.2:3b` | 1.9 GB | 4 GB | ⚡⚡⚡ Rápido | Bueno en español |
| `mistral:7b-q4` | 4.1 GB | 6 GB | ⚡⚡ Medio | Mejor calidad, recomendado con 8GB |
| `llama3.1:8b-q4` | 4.7 GB | 7 GB | ⚡⚡ Medio | Máxima calidad para 8GB |
| `mistral:7b-q8` | 7.7 GB | 10 GB | ⚡ Lento | Solo con 16GB RAM |

> Con 8GB RAM: usa `mistral:7b-q4` o `llama3.2:3b`. Con 16GB: `llama3.1:8b-q4`.

---

## 💾 Arquitectura de almacenamiento

```
ThinkPad SSD 512GB
├── /system          → Debian OS (~15GB)
├── /llm
│   ├── ollama/      → Modelos LLM (~5-10GB)
│   └── whisper/     → Modelos STT (~300MB)
├── /content
│   ├── zim/         → Wikipedia + WikiMed (~25-95GB)
│   ├── pdf/         → Docs médicos, técnicos (~5-20GB)
│   └── books/       → Calibre library (~10GB)
└── /swap            → 8-16GB swap (mejora LLM en RAM ajustada)

[Opcional] HDD externo USB
└── /backup          → Copia completa + contenido extra
```

> **¿Separar info del LLM?** No es necesario si tienes el SSD de 512GB. El LLM en SSD es CRÍTICO (latencia), el contenido puede ir en HDD externo si te quedas sin espacio.

---

## 🖥️ ThinkPad: Modelos recomendados

| Modelo | Precio aprox | CPU | RAM max | Notas |
|---|---|---|---|---|
| **X230** | 30–50€ | i5-3320M | 16GB | Clásico, robusto, batería externa disponible |
| **T430** | 35–55€ | i5/i7 3ª gen | 16GB | Pantalla 14", mejor teclado |
| **T440p** | 50–80€ | i5/i7 4ª gen | 16GB | Más moderno, muy barato ahora |
| **T450/T460** | 80–120€ | i5/i7 5ª-6ª gen | 32GB | Buena batería, DisplayPort |
| **X260/X270** | 90–130€ | i5/i7 6ª-7ª gen | 32GB | Ultraportable, 2 baterías |

> Todos estos soportan **coreboot** (firmware abierto) y tienen reputación de durabilidad 10+ años. Busca en Wallapop, eBay.es, o Back Market.

---

## 💰 Combinaciones por rango de precio

> **Recursos que ya tienes**: SSD 512GB + 8GB RAM (1 placa)

---

### 🟤 100€ — Funcional básico

**¿Qué consigues?**
- Servidor de contenido completo (Wikipedia, PDFs, libros)
- LLM 3B–7B con interfaz web (texto solo)
- Accesible desde cualquier dispositivo en red WiFi local

**Hardware**
| Componente | Opción | Precio est. |
|---|---|---|
| ThinkPad X230 o T430 | Wallapop / eBay | 35–50€ |
| Batería reemplazada (si necesita) | Genérica compatible | 15–25€ |
| **Total hardware** | | **~50–75€** |
| **Ya tienes** | SSD 512GB + 8GB RAM | 0€ |
| **Presupuesto software/extras** | | ~25–50€ restantes |

**Stack software**
- OS: Debian 12 Server
- LLM: Ollama + `mistral:7b-q4` o `phi3:mini`
- UI: Open WebUI
- Contenido: Kiwix-serve (Wikipedia ES texto ~22GB) + Nginx (PDFs)
- Audio: ❌ No (se puede añadir después gratis)

**Limitaciones**
- Solo 8GB RAM → LLM responde en 10–30 seg por token (lento pero funcional)
- Sin audio integrado
- Sin redundancia de almacenamiento

**Dificultad de implementación**: ⭐⭐☆☆☆ (2/5) — Todo tiene scripts de instalación en 1 comando  
**Dificultad de actualización**: ⭐⭐☆☆☆ — `ollama pull modelo` para LLM, kiwix descarga ZIMs nuevos  
**Durabilidad**: ⭐⭐⭐☆☆ — ThinkPad X/T230 aguanta 5–8 años más con mantenimiento básico

---

### 🥉 150€ — Confortable con audio

**¿Qué consigues?**
- Todo lo anterior
- 16GB RAM → LLM notablemente más rápido, modelos mejores
- Audio: dictado por voz + respuesta en voz
- Mejor batería (autonomía real 4–6h)

**Hardware**
| Componente | Opción | Precio est. |
|---|---|---|
| ThinkPad T440p o X230 | Wallapop / eBay | 45–65€ |
| 8GB RAM adicional (DDR3) | Compatible ThinkPad | 15–20€ → **16GB total** |
| Batería nueva | Genérica | 15–20€ |
| **Total** | | **~75–105€** |

**Stack software**
- LLM: Ollama + `llama3.1:8b-q4` (mucho mejor que 7B)
- Audio: Whisper.cpp (`base` en español) + Piper TTS
- Open WebUI con integración de voz activada
- Wikipedia completa con imágenes (~90GB ZIM) si el espacio lo permite

**Dificultad de implementación**: ⭐⭐⭐☆☆ — Whisper/Piper requieren algo de configuración  
**Dificultad de actualización**: ⭐⭐☆☆☆ — Scripts de actualización sencillos  
**Durabilidad**: ⭐⭐⭐⭐☆ — 16GB RAM extiende vida útil perceptiblemente

---

### 🥈 200€ — Sistema robusto y completo

**¿Qué consigues?**
- ThinkPad de generación más moderna (5ª–6ª gen Intel)
- 16GB RAM de base o ampliable a 32GB
- HDD externo para backup y contenido extra
- Panel solar pequeño para carga off-grid (real doomsday)

**Hardware - Opción A: Portabilidad máxima**
| Componente | Precio est. |
|---|---|
| ThinkPad X260 (i5, 8GB) | 90–110€ |
| RAM: 8GB adicional DDR4 | 20–25€ |
| Batería (ya tiene 2 slots) | 15€ |
| HDD externo 1TB USB3 | 40–50€ |
| **Total** | **~165–200€** |

**Hardware - Opción B: Potencia máxima**
| Componente | Precio est. |
|---|---|
| ThinkPad T460 (i7, 8GB) | 100–130€ |
| RAM: 8GB adicional DDR4 | 20€ |
| HDD externo 2TB | 55–70€ |
| **Total** | **~175–220€** |

**Stack software**
- LLM: `llama3.1:8b-q4` o incluso `mistral:7b-instruct` con mejor calidad
- Wikipedia ES completa + WikiMed + OpenStreetMap
- RAG básico: Ollama + script que indexa los PDFs y los inyecta al LLM como contexto
- Audio completo: Whisper + Piper en pipeline automático
- Hotspot WiFi: `hostapd` → cualquier dispositivo conectado accede a todo

**Nueva funcionalidad: RAG sobre documentos propios**
> Con **Ollama + Open WebUI + ChromaDB** puedes subir tus PDFs médicos y el LLM los leerá directamente. Preguntas como *"¿cómo trato una hemorragia interna según el manual Merck?"* con respuesta citada.

**Dificultad de implementación**: ⭐⭐⭐☆☆  
**Dificultad de actualización**: ⭐⭐⭐☆☆ — RAG requiere re-indexar si añades docs  
**Durabilidad**: ⭐⭐⭐⭐☆ — Hardware moderno, HDD de backup extiende vida del sistema

---

### 🥇 300€ — Sistema definitivo

**¿Qué consigues?**
- Hardware de última generación (7ª–8ª gen Intel o AMD)
- 32GB RAM → LLMs de mayor calidad, multitarea real
- SSD secundario o NVMe externo ultrarrápido
- Panel solar + batería externa (autonomía real off-grid)
- Coreboot instalado (máxima seguridad y control del firmware)

**Hardware - Opción A: ThinkPad premium**
| Componente | Precio est. |
|---|---|
| ThinkPad T470 (i7-7500U, 16GB) | 140–170€ |
| RAM: 16GB adicional DDR4 → 32GB | 35–45€ |
| HDD externo 2TB USB3 | 55€ |
| Panel solar 20W + banco baterías 20000mAh | 40–60€ |
| **Total** | **~270–330€** |

**Hardware - Opción B: Mini PC (alternativa más potente)**
| Componente | Precio est. |
|---|---|
| Beelink Mini S12 Pro (N100, 16GB, 500GB SSD) | 150–170€ |
| Monitor 13" portátil USB-C | 70€ |
| HDD externo 2TB | 55€ |
| Panel solar + batería | 45€ |
| **Total** | **~320–340€** |

> ⚠️ Opción B no es portátil, pero el N100 tiene **iGPU eficiente** que acelera los LLMs vía llama.cpp con soporte de capas en GPU. Mucho más rápido que CPU pura por watt.

**Stack software completo**
- **LLM**: Ollama + `llama3.1:8b-q4` o `deepseek-r1:7b` (razonamiento)
- **RAG avanzado**: Open WebUI + ChromaDB + indexación automática de PDFs
- **Audio completo**: Whisper `small` (mejor precisión) + Piper TTS neural
- **Mapas**: Servidor OsmAnd o MAPS.ME datos locales
- **Calibre Web**: Biblioteca de libros técnicos accesible en red
- **Hotspot**: `hostapd` + `dnsmasq` → portal cautivo propio
- **Backup automático**: `rsync` al HDD externo cada noche
- **Dashboard**: Página web custom de inicio con acceso a todo

**Dificultad de implementación**: ⭐⭐⭐⭐☆ — RAG + hotspot + pipeline de audio require tiempo  
**Dificultad de actualización**: ⭐⭐⭐☆☆  
**Durabilidad**: ⭐⭐⭐⭐⭐ — Redundancia, off-grid, hardware de 2017–2020 con piezas disponibles

---

## 📊 Tabla resumen por rango de precio

| Funcionalidad | 100€ | 150€ | 200€ | 300€ |
|---|:---:|:---:|:---:|:---:|
| Wikipedia offline | ✅ texto | ✅ completa | ✅ completa | ✅ completa |
| PDFs / libros técnicos | ✅ | ✅ | ✅ | ✅ |
| LLM chat texto | ✅ lento | ✅ fluido | ✅ muy fluido | ✅ excelente |
| LLM por voz (STT+TTS) | ❌ | ✅ básico | ✅ completo | ✅ neural |
| RAG sobre documentos propios | ❌ | ❌ | ✅ | ✅ avanzado |
| Hotspot WiFi (otros dispositivos) | ✅ | ✅ | ✅ | ✅ |
| Backup redundante (HDD externo) | ❌ | ❌ | ✅ | ✅ |
| Off-grid (solar) | ❌ | ❌ | Opcional | ✅ |
| Mapas offline | Básico | ✅ | ✅ | ✅ completos |
| RAM disponible para LLM | 6–7GB | 14–15GB | 14–15GB | 28–30GB |
| Velocidad LLM aprox. | 2–5 tok/s | 5–10 tok/s | 5–10 tok/s | 10–20 tok/s |

---

## 🚀 Guía de implementación rápida (cualquier rango)

```bash
# 1. Instalar Debian 12 Server (sin escritorio)
# 2. Instalar Ollama
curl -fsSL https://ollama.com/install.sh | sh
ollama pull mistral:7b-q4

# 3. Instalar Open WebUI (interfaz web para el LLM)
docker run -d -p 3000:8080 \
  -v open-webui:/app/backend/data \
  --name open-webui ghcr.io/open-webui/open-webui:main

# 4. Instalar Kiwix para Wikipedia
wget https://download.kiwix.org/release/kiwix-tools/kiwix-tools_linux-x86_64.tar.gz
# Descargar ZIM: https://download.kiwix.org/zim/wikipedia/
kiwix-serve --port=8888 /content/zim/*.zim

# 5. Servidor de PDFs con Nginx (acceso web a carpeta)
apt install nginx
# Enlazar /content/pdf a /var/www/html/docs

# 6. Hotspot WiFi
apt install hostapd dnsmasq
# Configurar con SSID "DoomsdayBox" sin contraseña o WPA2
```

---

## 📁 Fuentes de contenido recomendadas

### Wikipedia / Enciclopedias
- https://wiki.kiwix.org/wiki/Content_in_all_languages → descargas ZIM
- Wikipedia ES (texto): ~22GB · Wikipedia ES (completa): ~90GB
- WikiMed (medicina): ~1.5GB

### Documentación médica
- Manual Merck (versión libre en PDF)
- Where There Is No Doctor / No Dentist (Hesperian Foundation — licencia libre)
- ATLS Student Manual (buscar edición PDF)
- Wilderness Medicine (Paul Auerbach)
- CDC Emergency Preparedness PDFs (gratuitos)

### Técnico / Supervivencia
- Project Gutenberg: https://www.gutenberg.org (libros libres de derechos)
- Appropedia: tecnología apropiada y sostenible (descargable via Kiwix)
- Internet Archive (collections de referencia técnica)

### Mapas
- OpenStreetMap tiles: https://download.geofabrik.de
- OsmAnd maps: descarga por país

---

## ⚡ Consideraciones finales

**¿WebAssembly para LLM?** → WebLLM (mlc.ai/web-llm) funciona pero requiere WebGPU y es lento en CPU. **No recomendado** como solución principal. Úsalo solo como fallback para dispositivos que se conecten al hotspot y no quieran instalar nada.

**¿LLM entrenado con el contenido?** → Entrenar (fine-tune) requiere recursos que van más allá de este hardware. Lo más práctico y efectivo es **RAG** (Retrieval Augmented Generation): el LLM base busca en tus documentos y responde citando fuentes. Con Open WebUI + ChromaDB funciona out of the box desde el rango de 200€.

**¿Formato más universal?** → Para contenido: **HTML estático** (sin JS complejo) y **PDF/A**. Para LLM: **GGUF** (formato de llama.cpp, estándar de facto, corre en cualquier plataforma sin dependencias).

**Durabilidad del hardware** → Los ThinkPad T/X series tienen piezas disponibles en AliExpress/iFixit por 10+ años. Evita modelos E-series o L-series (menor calidad de construcción).
