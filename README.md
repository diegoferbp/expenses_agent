# Gmail Expense Tracker

Pipeline incremental que lee correos de gastos desde Gmail (solo de remitentes
permitidos), extrae los datos con un LLM y los registra en Google Sheets,
clasificando cada movimiento como `income` o `expense`.

El diseño separa de forma estricta lo determinista de lo probabilístico: los
scripts hacen las acciones (leer, escribir, etiquetar) y el LLM solo transforma
texto en JSON, sin acceso a ninguna herramienta. Ver `docs/ARCHITECTURE.md`.

## Cómo funciona

```
Gmail API → [1. Ingesta] → cuerpo → [2. Extracción LLM] → JSON → [3. Carga] → Sheets
```

1. **Ingesta** (determinista): busca correos sin procesar de los remitentes
   permitidos y extrae el cuerpo en texto plano.
2. **Extracción** (LLM, sin herramientas): convierte el cuerpo en un JSON
   estructurado con `merchant`, `amount`, `currency`, `payment_method` y `type`.
3. **Carga** (determinista): valida, sanitiza, deduplica, escribe la fila en
   Sheets y etiqueta el correo como procesado.

## Prerrequisitos

- Python 3.11+
- Un proyecto en **Google Cloud** con **Gmail API** y **Google Sheets API**
  habilitadas.
- **Credenciales OAuth 2.0** tipo *Desktop app* (`credentials.json`). Para uso
  personal puedes dejar la pantalla de consentimiento en modo *testing* con tu
  propia cuenta.
- Una **API key de Anthropic**.
- Una hoja de Google Sheets destino (con encabezados ya creados).

### Scopes de OAuth (privilegio mínimo)

- `https://www.googleapis.com/auth/gmail.readonly` — leer correos.
- `https://www.googleapis.com/auth/gmail.modify` — aplicar la etiqueta de
  procesado (solo si usas etiquetado).
- `https://www.googleapis.com/auth/spreadsheets` — escribir en la hoja.

## Setup

```bash
# 1. Entorno
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 2. Configuración
cp .env.example .env
#   edita .env con tus valores

# 3. Credenciales de Google
#   coloca credentials.json (OAuth desktop) en la raíz del proyecto.
#   en la primera corrida se abrirá el navegador para autorizar y
#   se generará token.json (guárdalo fuera del repo / en el keychain).
```

## Configuración (`.env`)

Ver `.env.example` para la lista completa. Lo esencial:

- `ANTHROPIC_API_KEY` y `ANTHROPIC_MODEL`
- `SPREADSHEET_ID` y `SHEET_NAME`
- `ALLOWED_SENDERS` — lista de remitentes permitidos, separada por comas.
- `PROCESSED_LABEL` — etiqueta de Gmail para marcar lo ya procesado.

## Ejecución

```bash
python -m src.main         # procesa los correos pendientes una vez
```

Para que sea recurrente, programa esa corrida con **cron** (Linux),
**launchd** (macOS) o el **Programador de tareas** (Windows). El proceso es
idempotente: solo toma los correos sin etiqueta, así que correrlo de más no
duplica filas.

## Estructura del proyecto

```
gmail-expense-tracker/
├── CLAUDE.md            # contexto para asistentes de código
├── README.md
├── .env.example
├── .gitignore
├── requirements.txt
├── docs/
│   ├── ARCHITECTURE.md  # diseño + modelo de amenazas
│   └── SCHEMA.md        # esquema de datos y reglas de validación
└── src/
    ├── ingest.py        # 1. lee Gmail y extrae el cuerpo
    ├── extract.py       # 2. LLM → JSON (sin herramientas)
    ├── load.py          # 3. valida, sanitiza y escribe en Sheets
    └── main.py          # orquestación
```

## Seguridad

Lee `docs/ARCHITECTURE.md`. En resumen: el LLM no tiene herramientas, los
correos se tratan como entrada no confiable, la salida del LLM se valida y
sanitiza (incluida la protección contra inyección de fórmulas en Sheets), los
scopes son mínimos y los secretos nunca van al repo.
