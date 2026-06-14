# CLAUDE.md

Contexto del proyecto para asistentes de código (Claude Code / Cowork).
Lee este archivo antes de generar o modificar código.

## Qué es

Pipeline incremental que lee correos de gastos desde Gmail (solo de remitentes
permitidos), extrae los datos financieros con un LLM y los registra como filas en
una hoja de Google Sheets. Clasifica cada movimiento como `income` o `expense`.

No es un agente autónomo con herramientas: es un **pipeline determinista con un
paso de extracción por LLM**. Esa distinción es deliberada y es la base del
modelo de seguridad. No la rompas.

## Modelo de seguridad (NO NEGOCIABLE)

El cuerpo del correo es **entrada no confiable** (puede estar spoofeado y puede
contener inyección de prompts). Para contenerlo:

1. El LLM se usa como **función pura, SIN herramientas**. No tiene acceso a
   Gmail, ni a Sheets, ni a red, ni a sistema de archivos. Solo recibe texto y
   devuelve JSON. Si una inyección de prompt tiene éxito, lo máximo que logra es
   un JSON malo, que el paso de carga rechaza.
2. Toda acción con efectos (leer Gmail, escribir Sheets, etiquetar) ocurre en
   **scripts deterministas**, nunca decidida por el LLM.
3. La salida del LLM se trata también como no confiable: se **valida** contra el
   schema y se **sanitiza** antes de escribirla.

Nunca le des herramientas al paso de extracción. Nunca dejes que el LLM dispare
escrituras, envíos o borrados.

## Arquitectura (3 componentes)

```
Gmail API → [1. Ingesta] → cuerpo → [2. Extracción LLM] → JSON → [3. Carga] → Sheets
```

- **Script 1 — Ingesta (`src/ingest.py`)** · determinista. Busca correos sin
  procesar de los remitentes permitidos y extrae el cuerpo en texto plano
  (maneja MIME multipart, base64url, y limpia HTML si no hay `text/plain`).
- **Paso 2 — Extracción (`src/extract.py`)** · LLM sin herramientas. Llama a la
  API de Anthropic con structured output / JSON schema, temperatura baja. El
  cuerpo va delimitado claramente como dato, no como instrucción.
- **Script 3 — Carga (`src/load.py`)** · determinista. Valida, sanitiza,
  deduplica, hace append en Sheets y **solo entonces** etiqueta el correo.

Ver `docs/ARCHITECTURE.md` para el modelo de amenazas completo y `docs/SCHEMA.md`
para el esquema de datos y las reglas de validación.

## Reglas que el código debe respetar

- **Idempotencia**: etiqueta el correo como procesado SOLO después de un append
  exitoso. Si falla, no se etiqueta y se reintenta en la siguiente corrida.
- **Privilegio mínimo en los scopes de OAuth**: `gmail.readonly`,
  `gmail.modify` (solo para etiquetar) y `spreadsheets`. Nada más.
- **Inyección de fórmulas**: antes de escribir en Sheets, si un valor de texto
  empieza con `=`, `+`, `-`, `@`, tab o retorno, neutralízalo (prefijo `'` o
  forzar formato texto). Ver `docs/SCHEMA.md`.
- **Datos de tarjeta**: guarda SOLO los últimos 4 dígitos. Nunca el PAN completo.
- **Secretos**: nada de API keys, `credentials.json` ni tokens en el repo.
  Todo por variables de entorno / keychain. Respeta `.gitignore`.
- **Cuarentena**: lo que no pase validación no se descarta en silencio; se loguea
  y el correo queda sin etiquetar para revisión manual.

## Estado / orden de construcción sugerido

1. [ ] Configurar Google Cloud (Gmail + Sheets API) y OAuth — ver README.
2. [ ] `ingest.py`: autenticación + búsqueda + extracción de cuerpo.
3. [ ] `extract.py`: llamada al LLM con JSON schema.
4. [ ] `load.py`: validación + sanitización + append + etiquetado.
5. [ ] Orquestación (`main.py`) y programación con cron/launchd/Task Scheduler.

## Lo que este proyecto NO debe hacer

- No darle herramientas al LLM.
- No confiar en el header `From` como autenticación (es spoofeable).
- No guardar números de tarjeta completos.
- No escribir en Sheets antes de validar y sanitizar.
- No commitear secretos.
