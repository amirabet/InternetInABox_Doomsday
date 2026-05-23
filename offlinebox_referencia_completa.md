# OfflineBox — Documento de Referencia del Proyecto
### Servidor de conocimiento offline · Búsqueda semántica multiplataforma · Doomsday Edition

---

## Índice

1. [Filosofía del proyecto](#1-filosofía-del-proyecto)
2. [Contenido del SSD externo](#2-contenido-del-ssd-externo)
3. [Software en el PC](#3-software-en-el-pc)
4. [Búsqueda semántica: arquitectura doble](#4-búsqueda-semántica-arquitectura-doble)
5. [Apéndice de hardware](#5-apéndice-de-hardware)

---

## 1. Filosofía del proyecto

### Principios

- **Todo offline**: ningún componente requiere internet para funcionar una vez configurado
- **Todo open source**: sin licencias privativas, sin dependencias de servicios externos
- **Multiplataforma**: el SSD funciona en cualquier OS con browser moderno
- **Consulta directa**: la máquina es el dispositivo de consulta, no un servidor headless
- **Durable**: hardware reparable, software basado en Debian estable, formatos universales

### Capas del sistema

```
┌─────────────────────────────────────────────────┐
│  CAPA 3 — Búsqueda semántica Linux completa      │
│  Meilisearch + embeddings + Extractive QA        │
├─────────────────────────────────────────────────┤
│  CAPA 2 — Búsqueda semántica universal           │
│  Transformers.js + PGlite + pgvector (browser)  │
├─────────────────────────────────────────────────┤
│  CAPA 1 — Contenido universal                    │
│  Kiwix ZIM + PDF + EPUB (cualquier OS)           │
└─────────────────────────────────────────────────┘
```

---

## 2. Contenido del SSD externo

### 2.1 Estructura de directorios

```
SSD_OfflineBox/
│
├── kiwix/
│   ├── apps/
│   │   ├── Kiwix-desktop-linux-x86_64.AppImage
│   │   ├── Kiwix-desktop-windows.exe
│   │   └── Kiwix-desktop-mac.dmg
│   ├── installers/                    ← plan B
│   │   ├── kiwix-installer-linux.deb
│   │   ├── kiwix-installer-windows.msi
│   │   └── kiwix-installer-mac.pkg
│   └── zim/                           ← contenido ZIM
│       ├── wikipedia_es_all_maxi.zim
│       ├── wikipedia_en_all_maxi.zim
│       ├── wikimed_es.zim
│       ├── wikimed_en.zim
│       ├── wikivoyage_es.zim
│       ├── wiktionary_es.zim
│       ├── appropedia_en.zim
│       ├── stackoverflow_en_technology.zim
│       └── gutenberg_es.zim
│
├── content/
│   ├── medical/
│   │   ├── merck_manual_es.pdf
│   │   ├── where_there_is_no_doctor.pdf
│   │   ├── where_there_is_no_dentist.pdf
│   │   ├── wilderness_medicine.pdf
│   │   ├── atls_manual.pdf
│   │   ├── cdc_emergency_preparedness.pdf
│   │   ├── botiquin_campo.pdf
│   │   └── primeros_auxilios_cruz_roja.pdf
│   ├── technical/
│   │   ├── electricidad_basica.pdf
│   │   ├── mecanica_diesel.pdf
│   │   ├── fontaneria_basica.pdf
│   │   ├── construccion_emergencia.pdf
│   │   └── radio_comunicaciones.pdf
│   ├── survival/
│   │   ├── purificacion_agua.pdf
│   │   ├── agricultura_basica.pdf
│   │   ├── conservacion_alimentos.pdf
│   │   └── navegacion_sin_gps.pdf
│   └── books/
│       └── [colección EPUB Gutenberg]
│
├── maps/
│   ├── README_mapas.txt
│   └── osmand/
│       ├── world_basemap.obf
│       ├── europe_basemap.obf
│       ├── spain_all.obf
│       └── [otros países según necesidad]
│
├── search/
│   ├── universal/                     ← CAPA 2: funciona en cualquier browser
│   │   ├── index.html                 ← abrir esto en cualquier OS
│   │   ├── assets/
│   │   │   ├── transformers.min.js
│   │   │   └── pglite.wasm
│   │   ├── model/
│   │   │   └── gte-small.onnx         ← modelo embeddings ~90MB
│   │   └── db/
│   │       └── offlinebox.db          ← PGlite con pgvector, pregenerado
│   │
│   └── linux/                         ← CAPA 3: Linux, semántico completo
│       ├── meilisearch                ← binario Linux x86_64
│       ├── data/                      ← índice Meilisearch pregenerado
│       ├── models/                    ← embeddings cacheados Python
│       ├── web/
│       │   └── app.py                 ← interfaz Flask + Extractive QA
│       └── start.sh                   ← arranca todo con un comando
│
└── README.html                        ← guía de inicio en cualquier browser
```

---

### 2.2 Contenido ZIM — tamaños reales

| Archivo ZIM | Contenido | Tamaño |
|---|---|---|
| `wikipedia_en_all_maxi.zim` | Wikipedia EN completa con imágenes | ~111 GB |
| `wikipedia_es_all_maxi.zim` | Wikipedia ES completa con imágenes | ~30 GB |
| `wikimed_en.zim` | Medicina Wikipedia EN | ~1 GB |
| `wikimed_es.zim` | Medicina Wikipedia ES | ~500 MB |
| `wikivoyage_es.zim` | Guías de viaje y geografía | ~700 MB |
| `wiktionary_es.zim` | Diccionario y definiciones | ~2 GB |
| `appropedia_en.zim` | Tecnología apropiada y sostenible | ~3 GB |
| `stackoverflow_en_technology.zim` | Stack Overflow tecnología | ~15 GB |
| `gutenberg_es.zim` | Project Gutenberg en español | ~10 GB |

Fuente de descargas: https://download.kiwix.org/zim/

---

### 2.3 Mapas offline

**Software**: OsmAnd+ (Android) o MAPS.ME como apps de consulta
**Datos**: OpenStreetMap, descarga por región desde https://download.geofabrik.de

| Mapa | Tamaño aprox |
|---|---|
| Basemap mundial | ~800 MB |
| España completa | ~1.2 GB |
| Europa (por países) | ~200–800 MB c/u |
| Mundo completo (tiles) | ~60 GB |

Para uso en portátil: servidor de tiles local con **TileServer GL** (binario incluible en el SSD).

---

### 2.4 Documentación médica recomendada (libre distribución)

| Documento | Fuente | Licencia |
|---|---|---|
| Where There Is No Doctor | hesperian.org | Creative Commons |
| Where There Is No Dentist | hesperian.org | Creative Commons |
| A Community Guide to Environmental Health | hesperian.org | Creative Commons |
| CDC Emergency Preparedness | cdc.gov | Dominio público |
| WHO Essential Medicines | who.int | Dominio público |
| Manual Merck (edición libre) | merckmanuals.com | Uso personal |
| Wilderness Medicine protocols | varias fuentes | Dominio público |

---

### 2.5 Cálculo de espacio total

| Categoría | Tamaño |
|---|---|
| Wikipedia EN + ES completas | 141 GB |
| Resto ZIMs (WikiMed, Gutenberg, etc.) | 35 GB |
| PDFs médicos y técnicos | 15 GB |
| Libros EPUB | 10 GB |
| Mapas (España + Europa) | 10 GB |
| Índice Meilisearch pregenerado | 20 GB |
| Base de datos PGlite pregenerada | 5 GB |
| Modelo ONNX + Transformers.js | 200 MB |
| Kiwix apps (3 plataformas) | 400 MB |
| **Total estimado** | **~237 GB** |

**Capacidad recomendada del SSD: 500 GB** (cubre todo con margen)
**Capacidad ideal: 1 TB** (añadir mapas mundiales, más idiomas, futuro LLM)

---

## 3. Software en el PC

### 3.1 Sistema operativo

| Opción | Base | RAM reposo | Recomendado para |
|---|---|---|---|
| **MX Linux XFCE** | Debian 12 | ~320 MB | ✅ Primera elección — MX Snapshot + MX Tools |
| LMDE 6 XFCE | Debian 12 | ~400 MB | ✅ Si se prefiere Mint — mayor respaldo institucional |
| Debian 12 + XFCE manual | Debian 12 | ~350 MB | Usuarios con experiencia Linux |

**NO recomendado**: Ubuntu (base menos estable a largo plazo), rolling releases (Arch, Manjaro), Windows.

Descarga MX Linux: https://mxlinux.org/download-links/
Descarga LMDE: https://linuxmint.com/download_lmde.php

---

### 3.2 Software base del sistema

```bash
apt install -y \
  firefox-esr \          # navegador principal
  nginx \                # servidor HTTP para PDFs
  python3 python3-pip \  # runtime para indexación y búsqueda
  curl wget git \        # utilidades
  hostapd dnsmasq \      # hotspot WiFi
  rsync \                # backup automático
  xfce4-terminal         # terminal
```

---

### 3.3 Motor de búsqueda — Capa 3 Linux

#### Meilisearch
- **Licencia**: MIT (Community Edition)
- **Instalación**: `curl -L https://install.meilisearch.com | sh`
- **Web**: https://meilisearch.com
- **Función**: motor de búsqueda híbrido (textual + semántico), interfaz web incluida
- **Puerto**: 7700

#### Dependencias Python para indexación

```bash
pip3 install --break-system-packages \
  meilisearch \                # cliente Meilisearch
  sentence-transformers \      # modelo embeddings multilingüe
  transformers \               # modelos HuggingFace
  torch \                      # runtime PyTorch (CPU)
  pdfminer.six \               # extracción texto PDF
  beautifulsoup4 \             # extracción texto HTML
  flask                        # servidor interfaz web
```

#### Modelos de embeddings (descarga automática al primer uso)

| Modelo | Tamaño | Función | Idiomas |
|---|---|---|---|
| `paraphrase-multilingual-MiniLM-L12-v2` | ~90 MB | Embeddings búsqueda semántica | 50+ incl. ES/EN |
| `deepset/xlm-roberta-base-squad2` | ~400 MB | Extractive QA (subrayado de respuesta exacta) | Multilingüe |

Ambos se cachean en `~/.cache/huggingface/` y funcionan 100% offline a partir de la primera descarga.

---

### 3.4 Motor de búsqueda — Capa 2 Universal (Transformers.js + PGlite)

No requiere instalación en el PC — vive en el SSD como archivos estáticos.

Para generarlo (una sola vez en Linux):

```bash
pip3 install --break-system-packages \
  pglite-python \        # si disponible, o generar via Node.js
  sentence-transformers \
  pdfminer.six
```

O vía Node.js/npm:
```bash
npm install @electric-sql/pglite @huggingface/transformers
```

Referencia: https://github.com/huggingface/transformers.js-examples/tree/main/pglite-semantic-search

---

### 3.5 Kiwix

- **Licencia**: GPL-3.0
- **Web**: https://kiwix.org
- **Instalación**: `apt install kiwix-tools` o AppImage portable
- **Función**: servidor y lector de archivos ZIM (Wikipedia offline)
- **Puerto**: 8888

---

### 3.6 Calibre Web (biblioteca)

- **Licencia**: GPL-3.0
- **Instalación**: `pip3 install calibreweb --break-system-packages`
- **Función**: biblioteca EPUB/PDF accesible desde browser en red local
- **Puerto**: 8083

---

### 3.7 Hotspot WiFi

```bash
# Permite que otros dispositivos (móvil, tablet) accedan a todo el sistema
apt install -y hostapd dnsmasq
# SSID: OfflineBox · Seguridad: WPA2
# IP del portátil en la red local: 192.168.4.1
```

---

### 3.8 Servicios systemd (arranque automático)

| Servicio | Puerto | Descripción |
|---|---|---|
| `meilisearch.service` | 7700 | Motor búsqueda semántica |
| `kiwix.service` | 8888 | Wikipedia y ZIMs |
| `offlinebox.service` | 5000 | Interfaz web Flask |
| `nginx.service` | 80 | Servidor PDFs y portal |
| `hostapd.service` | — | Hotspot WiFi |
| `dnsmasq.service` | — | DHCP red local |

---

### 3.9 URLs de acceso desde la red local

| Servicio | URL |
|---|---|
| Portal principal + búsqueda | `http://192.168.4.1` |
| Meilisearch (búsqueda directa) | `http://192.168.4.1:7700` |
| Wikipedia / ZIMs (Kiwix) | `http://192.168.4.1:8888` |
| Documentos PDF | `http://192.168.4.1/docs` |
| Biblioteca libros | `http://192.168.4.1:8083` |

---

### 3.10 Backup con MX Snapshot

MX Linux incluye **MX Snapshot**: crea una ISO bootable del sistema completo con todo configurado e indexado. Grabar en USB = respaldo completo del sistema.

```
MX Menu → System → MX Snapshot → Create Live ISO
```

---

### 3.11 Opcional futuro: LLM local (síntesis de resultados)

Si en el futuro se quiere añadir síntesis de respuestas (Capa 4):

| Software | Licencia | Instalación |
|---|---|---|
| **Ollama** | MIT | `curl -fsSL https://ollama.com/install.sh \| sh` |
| **Open WebUI** | MIT | Docker o pip |

Modelos recomendados para síntesis ligera (no razonamiento):

| Modelo | RAM necesaria | Velocidad X220/T440p |
|---|---|---|
| `llama3.2:1b` | +1 GB | ~8–12 tok/s |
| `phi3:mini` (3.8B) | +3 GB | ~3–5 tok/s |
| `mistral:7b-q4` | +5 GB | ~1–2 tok/s |

---

## 4. Búsqueda semántica: arquitectura doble

### Capa 2 — Universal (Transformers.js + PGlite)

**Funciona en**: cualquier OS con Chrome, Firefox o Edge moderno
**Instalar**: nada — solo abrir `search/universal/index.html` del SSD

```
Pregunta en lenguaje natural
        ↓
Transformers.js (ONNX/WASM en browser)
genera embedding de la pregunta
        ↓
PGlite + pgvector (PostgreSQL en WASM)
búsqueda por similitud vectorial HNSW
en base de datos pregenerada
        ↓
Top 5 fragmentos semánticamente relevantes
con fuente y enlace al documento
```

- "hueso roto" → encuentra "fractura" ✅
- Sin servidor, sin Python, sin instalación
- Primera carga: ~3–5 seg (modelo ONNX)
- Búsquedas posteriores: instantáneas

**Limitación**: no incluye Extractive QA (no subraya la frase exacta dentro del fragmento)

---

### Capa 3 — Linux completa (Meilisearch + Extractive QA)

**Funciona en**: Linux (cualquier distro con Python)
**Iniciar**: `./search/linux/start.sh` desde el SSD, o servicios systemd permanentes

```
Pregunta en lenguaje natural
        ↓
Meilisearch Hybrid Search
(60% semántico + 40% textual simultáneo)
busca en todos los índices
        ↓
Top 8 fragmentos relevantes
        ↓
Extractive QA (xlm-roberta)
subraya la frase exacta de cada fuente
        ↓
Interfaz: lista de resultados con
respuesta destacada + fuente + enlace
```

- Búsqueda semántica + Extractive QA
- Interfaz web propia accesible desde otros dispositivos vía hotspot
- Velocidad: ~500ms por búsqueda en X220/T440p

---

### Comparativa de capas

| | Capa 1 | Capa 2 | Capa 3 |
|---|---|---|---|
| **Tecnología** | Kiwix + PDF viewer | Transformers.js + PGlite | Meilisearch + Python |
| **Plataforma** | Cualquier OS | Cualquier browser | Solo Linux |
| **Instalar** | Kiwix app | Nada | Servicios systemd |
| **Semántica** | ❌ | ✅ | ✅ |
| **Extractive QA** | ❌ | ❌ | ✅ |
| **Multi-fuente** | Manual | ✅ | ✅ |
| **Velocidad** | Instantánea | ~3–5s primera vez | ~500ms |

---

## 5. Apéndice de hardware

### 5.1 Portátiles recomendados

#### ThinkPad — Serie T/X (preferidos)

| Modelo | Precio est. | CPU gen. | AVX2 | RAM máx | Pantalla | Notas |
|---|---|---|---|---|---|---|
| X220 i5/i7 | 50–80€ | Sandy Bridge 2ª | ❌ | 16 GB DDR3 | 12.5" 1366×768 | Libreboot · teclado clásico · muy reparable |
| T430 i5/i7 | 45–70€ | Ivy Bridge 3ª | ❌ | 16 GB DDR3 | 14" 1600×900 | Teclado clásico · mejora mínima vs X220 |
| **T440p i5/i7** | **100–180€** | **Haswell 4ª** | **✅** | **16 GB DDR3L** | **14" hasta FHD** | **CPU socketed (upgrade a i7-4810MQ ~15€)** |
| T450 i5/i7 | 90–140€ | Broadwell 5ª | ✅ | 32 GB DDR3L | 14" hasta FHD | Doble batería hot-swap · excelente off-grid |
| **T460 i5/i7** | **110–160€** | **Skylake 6ª** | **✅** | **32 GB DDR4** | **14" hasta FHD** | **Mejor iGPU · USB-C · DDR4** |
| X260 i5/i7 | 100–150€ | Skylake 6ª | ✅ | 32 GB DDR4 | 12.5" hasta FHD | Compacto · doble batería |
| T470 i5/i7 | 130–180€ | Kaby Lake 7ª | ✅ | 32 GB DDR4 | 14" hasta FHD | Más moderno · Thunderbolt en algunos |

**Sweetspot recomendado**: T440p i7 con upgrade CPU (~130€ total) o T460 (~130€).
**Para formato compacto 12"**: X260 con FHD.

---

#### Alternativas otras marcas

| Modelo | Precio est. | AVX2 | RAM máx | Robustez | Notas |
|---|---|---|---|---|---|
| **HP EliteBook 840 G3** | **70–110€** | **✅ Skylake** | **32 GB DDR4** | MIL-STD-810G | Mejor precio que T460 equivalente · IP65 no |
| HP EliteBook 840 G4 | 90–130€ | ✅ Kaby Lake | 32 GB DDR4 | MIL-STD-810G | Un año más nuevo |
| Dell Latitude E7470 | 80–120€ | ✅ Skylake | 32 GB DDR4 | Corporativo | Muy reparable · Dell Parts Direct |
| Dell Latitude 7490 | 110–160€ | ✅ 8ª gen | 32 GB DDR4 | Corporativo | Más moderno |
| **Panasonic CF-54** | **100–160€** | **✅ Haswell** | **16 GB DDR3** | **MIL-STD-810H** | **IP65 · caídas 1.8m · -10°C a +50°C** |
| Framework 13 | 350–450€ | ✅ 12ª gen | 64 GB DDR5 | Normal | iFixit 10/10 · máxima reparabilidad |

**Para doomsday real** (agua, polvo, frío, golpes): Panasonic CF-54.
**Mejor precio/prestaciones**: HP EliteBook 840 G3.

---

#### Criterios de compra

- Buscar **sin SSD** o **sin RAM** para aprovechar los componentes ya disponibles (512 GB SSD + 8 GB RAM)
- Evitar revendedores profesionales en Wallapop — buscar particulares con fotos caseras
- Activar alertas en **Idealo.es** y **CamelCamelCamel** para variaciones de precio
- Plataformas: Wallapop · Milanuncios · Back Market · eBay.es (vendedores DE/FR suelen ser más baratos)

---

#### Cambio de teclado a español

Todos los modelos de la lista permiten cambio de teclado. Teclados ES disponibles en AliExpress y eBay.

| Modelo | Dificultad cambio | Precio teclado ES |
|---|---|---|
| ThinkPad X220 | ⭐☆☆ Muy fácil | 12–20€ |
| ThinkPad T440p / T450 / T460 | ⭐☆☆ Muy fácil | 15–25€ |
| HP EliteBook 840 G3/G4 | ⭐⭐☆ Media | 20–35€ |
| Dell Latitude E7470 | ⭐⭐☆ Media | 20–30€ |

---

### 5.2 Almacenamiento externo de contenido

#### Estrategia: SSD interno 2.5" SATA + enclosure + funda

Un SSD externo dedicado (Samsung T7 Shield, SanDisk Extreme) es esencialmente un SSD M.2 dentro de una carcasa propietaria. Si el controlador USB falla, recuperar los datos puede ser imposible sin servicio técnico. Un SSD estándar 2.5" SATA en un enclosure genérico ofrece mayor universalidad: si el enclosure falla, el SSD se conecta directamente a cualquier portátil vía SATA o a cualquier otro enclosure. Para un proyecto de emergencia, esa flexibilidad es prioritaria.

**El enclosure no necesita adaptador de corriente.** Un SSD 2.5" SATA consume tan poca energía que se alimenta completamente por USB. Solo los enclosures para HDD 3.5" o docking stations de múltiples bahías requieren corriente externa.

---

#### SSD interno 2.5" SATA recomendados

| Modelo | Capacidad | Precio est. | Garantía | Notas |
|---|---|---|---|---|
| **Samsung 870 EVO** | 1 TB | 70–90€ | 5 años | El más fiable del mercado |
| **Crucial MX500** | 1 TB | 55–75€ | 5 años | Mejor relación calidad/precio |
| Kingston KC600 | 1 TB | 55–70€ | 5 años | Sólido, ampliamente disponible |
| WD Blue 3D NAND | 1 TB | 60–80€ | 5 años | También muy fiable |
| Samsung 870 EVO | 500 GB | 45–60€ | 5 años | Si el presupuesto es ajustado |
| Crucial MX500 | 500 GB | 40–55€ | 5 años | Alternativa económica a 500 GB |

---

#### Enclosures recomendados (bus-powered, sin adaptador de corriente)

| Modelo | Material | USB | Precio est. | Notas |
|---|---|---|---|---|
| **Ugreen USB 3.1 Gen2** | Aluminio | 3.1 Gen2 (10 Gbps) | 18–25€ | ✅ Recomendado — rápido, compacto, buena disipación |
| Inateck FE2025 | Aluminio | 3.0 (5 Gbps) | 15–20€ | Sin tornillos, acceso rápido |
| Orico 2577U3 Rugged | Aluminio + goma | 3.0 (5 Gbps) | 25–35€ | Protección ante golpes integrada |

> ⚠️ Evitar el Sabrent EC-UASP: carcasa de plástico básico, sin protección real, no adecuado para uso de emergencia.

---

#### Funda protectora

Una funda de neopreno para disco duro 2.5" añade absorción de golpes sin coste relevante. Alternativa: funda rígida EVA con cierre de cremallera (~8–12€ en AliExpress), más resistente y con espacio para el cable USB.

| Tipo | Precio | Protección |
|---|---|---|
| Funda neopreno 2.5" | 3–6€ | Golpes leves, arañazos |
| Funda rígida EVA con cremallera | 8–12€ | Golpes moderados, polvo |

---

#### Coste total del conjunto

| Combinación | Componentes | Coste total |
|---|---|---|
| **Económica** | Crucial MX500 500 GB + Orico 2577U3 + funda neopreno | ~75–90€ |
| **Recomendada** | Crucial MX500 1 TB + Ugreen Gen2 + funda EVA | ~85–105€ |
| **Premium** | Samsung 870 EVO 1 TB + Ugreen Gen2 + funda EVA rígida | ~95–115€ |

---

#### Ventajas frente a SSD externo dedicado

| Criterio | SSD externo dedicado | SSD interno + enclosure |
|---|---|---|
| Si el enclosure falla | ❌ Datos en riesgo | ✅ SSD usable en cualquier portátil |
| Trasladar a otra máquina | Solo por USB | ✅ USB o bahía SATA interna |
| Precio por GB | Mayor | ✅ Menor |
| Repuesto si se rompe la carcasa | Unidad completa | ✅ Solo enclosure (~20€) |
| Formato propietario | ❌ Sí (M.2 interno sellado) | ✅ No (SATA estándar universal) |
| Certificación IP integrada | IP55–65 en modelos premium | Con Orico Rugged: protección básica |

---

#### Capacidad necesaria

| Escenario | Espacio necesario | SSD recomendado |
|---|---|---|
| Wikipedia ES + docs básicos | ~80 GB | 250 GB (ajustado) |
| Wikipedia ES + EN + docs completos | ~240 GB | **500 GB** |
| Todo lo anterior + mapas mundiales + futuro LLM | ~320 GB | **1 TB** ✅ |
| Múltiples idiomas Wikipedia + máxima expansión | ~500 GB+ | 2 TB |

---

### 5.3 RAM adicional

Si el portátil adquirido tiene menos de 16 GB:

| Generación | Tipo | Capacidad | Precio est. |
|---|---|---|---|
| ThinkPad X220 / T430 | DDR3 1600 MHz SO-DIMM | 8 GB | 12–18€ |
| ThinkPad T440p / T450 | DDR3L 1600 MHz SO-DIMM | 8 GB | 12–18€ |
| ThinkPad T460 / X260 / EliteBook 840 G3 | DDR4 2133 MHz SO-DIMM | 8 GB | 15–20€ |
| ThinkPad T470 / Dell 7490 | DDR4 2400 MHz SO-DIMM | 16 GB | 25–35€ |

**Nota**: verificar siempre el modelo exacto del portátil antes de comprar RAM — algunas variantes del mismo modelo usan frecuencias diferentes.

---

## Resumen de inversión total

> Asumiendo componentes ya disponibles: SSD interno 512 GB (para el OS) + 8 GB RAM

### Escenario mínimo viable (~110–140€)

| Componente | Precio est. |
|---|---|
| Portátil X220 i5 sin SSD/RAM (Wallapop particular) | 50–70€ |
| RAM 8 GB DDR3 adicional → 16 GB total | 15€ |
| Crucial MX500 500 GB (contenido) | 40–50€ |
| Enclosure Ugreen USB 3.1 Gen2 | 20€ |
| Funda neopreno 2.5" | 5€ |
| **Total** | **~130–160€** |

### Escenario recomendado (~190–230€)

| Componente | Precio est. |
|---|---|
| Portátil T460 i5 o EliteBook 840 G3 sin SSD/RAM | 90–120€ |
| RAM 8 GB DDR4 adicional → 16 GB total | 18€ |
| Crucial MX500 1 TB (contenido) | 60–70€ |
| Enclosure Ugreen USB 3.1 Gen2 | 20€ |
| Funda EVA rígida con cremallera | 10€ |
| **Total** | **~198–238€** |

### Escenario completo (~280–330€)

| Componente | Precio est. |
|---|---|
| Portátil T460 i7 FHD + teclado ES | 150–180€ |
| RAM ya incluida (16 GB) | — |
| Samsung 870 EVO 1 TB (contenido) | 75–90€ |
| Enclosure Ugreen USB 3.1 Gen2 | 20€ |
| Funda EVA rígida con cremallera | 10€ |
| Batería externa 20000 mAh (off-grid) | 30–40€ |
| **Total** | **~285–340€** |

---

*Documento generado para proyecto OfflineBox — Doomsday Edition*
*Revisión: mayo 2026 · Todo el software referenciado es open source*
*Almacenamiento externo: SSD 2.5" SATA interno + enclosure bus-powered + funda protectora*
