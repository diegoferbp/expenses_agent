# Esquema de datos, validación y sanitización

> Convención: la prosa va en español, pero **todos los identificadores**
> (campos, valores de enum, etiquetas, columnas) van en inglés.

## Objeto que produce el LLM

El paso de extracción devuelve **un solo objeto JSON** con esta forma:

```json
{
  "date": "2026-06-10",
  "merchant": "Supermercado Éxito",
  "amount": 84500.0,
  "currency": "COP",
  "payment_method": "****1234",
  "type": "expense"
}
```

## Campos

| Campo | Tipo | Descripción | Reglas |
|---|---|---|---|
| `date` | string | Fecha del movimiento | ISO `YYYY-MM-DD`. Si el correo no la trae, usar la fecha del correo. |
| `merchant` | string | Nombre del comercio o contraparte | No vacío. Sanitizar contra fórmulas. |
| `amount` | number | Valor del movimiento | `> 0`. Solo el número, sin símbolos de moneda. Punto decimal. |
| `currency` | string | Código de moneda | ISO 4217 (`COP`, `USD`, `EUR`...). Whitelist. |
| `payment_method` | string | Medio de pago / tarjeta | Solo últimos 4 dígitos: regex `^\*{0,}\d{4}$`. Nunca el PAN completo. |
| `type` | string | Clasificación | `"income"` o `"expense"` exclusivamente. |

Pistas para la clasificación (`type`): términos del correo como *compra, pago,
cargo, débito, retiro* → `expense`; *abono, depósito, transferencia recibida,
devolución, reembolso* → `income`. (Las palabras clave están en español porque
así llegan los correos; el valor almacenado siempre es el enum en inglés.)

## Reglas de validación (paso de carga)

Antes de escribir cualquier fila se verifica:

1. **Campos requeridos** presentes y con el tipo de dato correcto.
2. **`type` ∈ {`income`, `expense`}**. Si no, → cuarentena.
3. **`currency` ∈ whitelist** de códigos ISO 4217 configurada. Si no, → cuarentena.
4. **`amount` > 0** y dentro de un rango razonable configurable (p. ej.
   `0 < amount <= 100_000_000`) para atrapar valores absurdos. Fuera de rango →
   cuarentena.
5. **`payment_method`** cumple el regex de 4 dígitos. Si no, normalizar o → cuarentena.
6. **`date`** parsea como fecha válida.

Lo que no pase: se **loguea con el `message_id`** y el correo **no se etiqueta**,
para revisión manual. Nunca se descarta en silencio.

## Sanitización contra inyección de fórmulas (OBLIGATORIA)

Google Sheets puede interpretar como **fórmula** cualquier celda de texto que
empiece con `=`, `+`, `-`, `@`, tab (`\t`) o retorno (`\r`/`\n`). Un correo
malicioso podría lograr que el LLM extraiga, por ejemplo, un `merchant` como
`=IMPORTRANGE(...)`.

Regla: a todo **campo de texto** (`merchant`, `payment_method`, y cualquier
string) que empiece con uno de esos caracteres, anteponerle un apóstrofo (`'`) o
un espacio, **o** escribir esas columnas con formato de celda forzado a texto.

```
"=IMPORTRANGE(...)"  →  "'=IMPORTRANGE(...)"
```

Los campos numéricos (`amount`) no se sanitizan así: se validan como número y se
escriben como número.

## Layout de la hoja de Google Sheets

Encabezados (fila 1), en orden de columnas:

```
date | merchant | amount | currency | payment_method | type | message_id | processed_at
```

- `message_id`: id del correo de origen. Sirve para **deduplicar** y para
  trazabilidad.
- `processed_at`: timestamp de cuándo se escribió la fila.

## Deduplicación

Antes del append, verificar que el `message_id` no esté ya en la columna
correspondiente (o mantener un índice local). Combinado con el etiquetado en
Gmail, da doble protección contra duplicados.
