# Radicado Uno — servidor MCP

Servidor **MCP remoto** (Model Context Protocol) que da a un agente acceso de
solo lectura a los datos mercantiles y de contratación pública de Colombia,
cruzados por NIT y **con la fuente oficial de cada dato**.

Endpoint: `https://mcp.radicadouno.co/mcp` · Transporte: Streamable HTTP ·
Autenticación: `Authorization: Bearer rduno_…`

> Este repositorio contiene la documentación y el descriptor del servidor. El
> servidor es **remoto y alojado por nosotros**: no hay nada que instalar ni
> desplegar, y el código de ingesta no es público.

## Qué datos sirve

Quince fuentes oficiales colombianas, replicadas y cruzadas por NIT. Las
cifras son las verificadas el 2026-08-16:

| Fuente | Volumen |
|---|---|
| Contratos electrónicos SECOP II (`jbjy-vk9h`) | 5.910.478 |
| Procesos de contratación, el embudo completo (`p6dx-8zbt`) | 8.549.010 |
| Matrículas del registro mercantil RUES (`c82u-588k`) | 9.324.004 |
| Empresas con ficha pública | 385.225 |

Se añaden el expediente RUES en vivo por NIT (49 campos, vía Confecámaras),
compromisos presupuestales SIIF, órdenes de la Tienda Virtual del Estado,
estados financieros de Supersociedades, sanciones en contratación, sanciones
de la Superintendencia de Industria y Comercio y las listas restrictivas de la
OFAC y la ONU.

Cada registro conserva la URL del documento oficial del que sale y la fecha en
que se capturó, de modo que cualquier respuesta puede comprobarse de forma
independiente.

## Herramientas

Las seis de datos son de **solo lectura** (`readOnlyHint`) y devuelven
`outputSchema`. Las dos últimas no sirven datos: sirven para darse de alta.

| Herramienta | Qué hace |
|---|---|
| `radicadouno_buscar_empresa` | Busca en las 9,3M de matrículas del RUES por prefijo de razón social o por NIT exacto. Devuelve candidatos con NIT para usar en las demás herramientas. |
| `radicadouno_perfil_empresa` | Cruce completo por NIT: registro mercantil, expediente enriquecido, registro de proveedor estatal y agregados de contratación pública. Cada bloque con su provenance. |
| `radicadouno_contratos_empresa` | Contratos del SECOP II donde el NIT es el proveedor adjudicado, con el enlace oficial al expediente de cada contrato. Filtrable por año y estado. |
| `radicadouno_senales_empresa` | Indicadores verificables: presencia física, volumen de contratación, concentración con su mayor cliente, procesos ganados con oferta única, sanciones y últimas cifras financieras. |
| `radicadouno_red_empresa` | Recorre el grafo de contratación a dos saltos (empresa → entidad pública → otra empresa): quién concurre ante los mismos compradores, con similitud normalizada por el tamaño de cada cartera. |
| `radicadouno_estado_fuentes` | Qué fuentes están cargadas, cuántas filas y la fecha de la última captura. Sirve para citar la frescura del dato. |
| `radicadouno_contratar` | **Sin clave.** Abre una caja de pago del plan Business y devuelve la URL y una referencia. |
| `radicadouno_estado_contratacion` | **Sin clave.** Con esa referencia: dice si el pago está confirmado y entrega la clave, una sola vez. |

`radicadouno_senales_empresa` **no es un score crediticio**. La Ley 1266 de
2008 reserva esa actividad a las centrales de riesgo y no somos una: son
hechos con su fuente, sin calificación.

## Cómo conseguir una clave (un agente puede hacerlo solo)

El servidor MCP está incluido en el **plan Business** (619.900 COP al mes), y
el alta no necesita que intervenga nadie por nuestra parte:

1. El cliente llama a **`radicadouno_contratar`** — es una de las dos
   herramientas que funcionan sin clave. Devuelve una URL de pago de Stripe y
   una `referencia`.
2. Se completa el pago en esa URL.
3. El cliente llama a **`radicadouno_estado_contratacion`** con la referencia.
   Mientras el pago no esté confirmado responde `pendiente`; en cuanto lo
   está, **entrega la clave una sola vez** y la borra de nuestro lado.

A partir de ahí la clave viaja en `Authorization: Bearer rduno_…`. Se entrega
una vez y no se puede volver a consultar: a partir de la entrega, en la base
solo queda su SHA-256. Si se pierde, escribe a <hola@radicadouno.co> y se
reemite. Una contratación pagada y no recogida caduca a las 72 horas.

También se puede contratar a mano en <https://radicadouno.co/precios>.

## Conectar desde Claude Desktop

El servidor es remoto, así que se conecta con
[`mcp-remote`](https://www.npmjs.com/package/mcp-remote). En
`claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "radicadouno": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://mcp.radicadouno.co/mcp",
        "--header",
        "Authorization:Bearer ${RADICADOUNO_CLAVE}"
      ],
      "env": { "RADICADOUNO_CLAVE": "rduno_su_clave" }
    }
  }
}
```

## Conectar con curl

```sh
curl -s https://mcp.radicadouno.co/mcp \
  -H 'content-type: application/json' \
  -H 'accept: application/json, text/event-stream' \
  -H "authorization: Bearer $RADICADOUNO_CLAVE" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

Una llamada a una herramienta:

```sh
curl -s https://mcp.radicadouno.co/mcp \
  -H 'content-type: application/json' \
  -H 'accept: application/json, text/event-stream' \
  -H "authorization: Bearer $RADICADOUNO_CLAVE" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":2,
       "params":{"name":"radicadouno_buscar_empresa",
                 "arguments":{"texto":"BANCOLOMBIA","limite":5}}}'
```

Estado del servicio, sin clave: <https://mcp.radicadouno.co/health>

## Límites y condiciones

- **El saludo del protocolo es abierto**: `initialize`, `ping` y `tools/list`
  responden sin clave, para que cualquier cliente pueda ver qué hay aquí antes
  de contratar. Los datos no: `tools/call` exige clave salvo en las dos
  herramientas de contratación.
- **120 peticiones por minuto y por IP**, antes de comprobar la clave.
- Un **límite por clave** además del anterior, según el plan.
- Sin clave válida la respuesta es `401` con el motivo; con un plan que no
  incluye MCP, `403`. Al pasarse de vueltas, `429` con `Retry-After`.
- Modo **sin estado**: cada petición es independiente (no hay sesiones).
- Solo lectura: ninguna herramienta escribe nada.
- Uso medido por llamada, para facturación y para saber qué se pregunta.

## Lo que este servidor no hace

Dicho por delante, para que nadie se lleve una sorpresa:

- **No calcula score ni calificación crediticia** (Ley 1266 de 2008).
- **Los estados financieros no cubren todas las empresas**: dependen de lo que
  reporta Supersociedades. Si una empresa no reporta, no habrá cifras.
- **Las personas naturales comerciantes están excluidas a propósito** (Ley 1581
  de 2012, habeas data), aunque figuren en el RUES.
- **Las sanciones de la Superintendencia de Industria y Comercio no traen NIT**
  en la fuente oficial: solo se enlazan cuando el nombre cruza de forma única
  contra el registro mercantil. Antes un hueco que un falso positivo.
- **Las listas restrictivas (OFAC, ONU) se cruzan solo por NIT exacto**
  declarado por la propia lista, nunca por parecido de nombre. La ONU no
  publica NIT, así que sus entradas nunca aparecen enlazadas a una empresa.
- **El dato hereda los errores de su fuente.** Si el SECOP registra mal un
  valor, aquí aparece igual — con el enlace al expediente oficial para poder
  comprobarlo.
- **No hay acuerdo de nivel de servicio** en el plan estándar.

Metodología completa, con la cadencia de actualización de cada fuente:
<https://radicadouno.co/metodologia>

## Fuentes

Datos abiertos del Estado colombiano publicados en
[datos.gov.co](https://www.datos.gov.co) bajo licencia
[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/), más la
consulta en vivo al [RUES](https://www.rues.org.co) de Confecámaras y las
listas OFAC y ONU. El acceso público conforme a la Ley 1712 de 2014.

La licencia MIT de este repositorio cubre su documentación y configuración, no
los datos servidos, que conservan la licencia de su fuente.

---

# Radicado Uno — MCP server (English summary)

A **remote MCP server** giving agents read-only access to Colombian company
and public-procurement data, cross-referenced by NIT (the national tax ID),
**with the official source of every field**.

- **Endpoint:** `https://mcp.radicadouno.co/mcp` (Streamable HTTP)
- **Auth:** `Authorization: Bearer rduno_…` — included in the Business plan
  (<https://radicadouno.co/precios>)
- **Self-service sign-up, no human in the loop:** call `radicadouno_contratar`
  (no key needed) for a Stripe checkout URL and a reference, pay, then call
  `radicadouno_estado_contratacion` with that reference — it hands over the key
  once and forgets it. The MCP handshake (`initialize`, `ping`, `tools/list`)
  is open so any client can inspect the server before paying.
- **Health:** <https://mcp.radicadouno.co/health>

**Data:** 5.9M SECOP II public contracts, 8.5M procurement processes, 9.3M
RUES commercial-registry records, 385,225 published company profiles, plus
live RUES lookups, Supersociedades financial statements, procurement
sanctions, and OFAC/UN restrictive lists. Every record keeps its source URL
and capture date.

**Tools (the six data tools are read-only):** `radicadouno_buscar_empresa` (search by name or
NIT), `radicadouno_perfil_empresa` (full profile), `radicadouno_contratos_empresa`
(public contracts with official links), `radicadouno_senales_empresa`
(objective signals — **not** a credit score), `radicadouno_red_empresa`
(two-hop procurement graph), `radicadouno_estado_fuentes` (source freshness).

**Limits:** 120 requests/minute per IP before key validation, plus a
per-key limit; stateless; no SLA on the standard plan.

**Known gaps, stated up front:** no credit scoring (Colombian Law 1266/2008);
financial statements only for companies that report to Supersociedades;
sole-trader records excluded on privacy grounds (Law 1581/2012); SIC sanctions
linked only on a unique exact name match; OFAC/UN lists matched only on exact
NIT; data inherits its source's errors.

MIT licensed (docs and configuration). The data keeps its own source licence
(CC BY-SA 4.0 for datos.gov.co).
