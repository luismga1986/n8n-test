# Bot Tubosa - Consultas Comerciales

Workflow de n8n para el bot comercial de Tubosa en Telegram.  
ID en producción: `F2ipp8XPwX5AQ6Dp`  
Instancia n8n: `https://n8n.tubosa.net`

## Qué hace

Bot de Telegram que responde consultas comerciales de clientes de Tubosa (empresa venezolana fabricante de tuberías y accesorios PVC). Permite:

- Buscar productos en el inventario (stock disponible + backorder)
- Consultar precio de lista (sin IVA) con descuentos por categoría
- Responder sobre descuentos por línea de producto

## Arquitectura

```
Telegram Trigger → Normalizar → Agente IA (GPT-4o) → Enviar Respuesta IA
                                      ↑
                     OpenAI GPT-4o · Memoria (10 msgs) · Tool: buscar_productos · Tool: ver_precio_lista
```

## Fuentes de datos

Ambas hojas son públicas (acceso sin auth):

| Hoja | Spreadsheet ID | GID | Contenido |
|------|---------------|-----|-----------|
| Inventario/Backorder | `1G8wEi3bOLxzFKzm086op_tkvciQ7ZTA4METm4pP_g30` | `427774701` | Clasificacion, Grupo, Material, Descripcion, Stock, Entregas, Backorder, Kg |
| Precios/Códigos | `1UA7W5emR48SLSPQ0kgz7o6Z6g0moByHj7hIFRPMCKOs` | `1313468133` | Material, Descripcion, Precio Sin IVA, AAA+, AAA, AA, A |

> La hoja de precios tiene una fila de números en la fila 1. Los encabezados reales están en la fila 2. Por eso se usa `range=A2:K` en la URL de gviz/tq.

## Cómo funcionan los tools

Los tools usan la Google Visualization API (no requiere auth para hojas públicas):

```
https://docs.google.com/spreadsheets/d/{ID}/gviz/tq?tqx=out:json&gid={GID}
```

### Búsqueda fuzzy (ambos tools)

1. Normaliza el query: minúsculas + sin tildes + solo alfanumérico
2. Divide en tokens (palabras ≥ 2 chars)
3. Busca cuántos tokens aparecen en `codigo + descripcion` de cada producto
4. Prioriza coincidencias totales; si hay parciales, las muestra también
5. Si hay > 8 resultados: pide más detalle
6. Si hay 0 resultados: sugiere probar con otro término
7. Si hay > 3 resultados en precios: muestra lista numerada para que el usuario elija

### Tool: buscar_productos

- Fuente: hoja de inventario
- Columnas clave: `Material`, `Texto breve de material`, `Inven Disp Unid`, `Unid Backorder Total`
- Muestra: código | descripción | ✅/❌ disponible | backorder si aplica

### Tool: ver_precio_lista

- Fuente: hoja de precios
- Columnas clave: `Material`, `Descripcion`, `Precio Sin Iva`, `AAA+`, `AAA`, `AA`, `A`
- Muestra: código | descripción | precio lista | descuentos por categoría
- Si el precio dice "Confirmar precio de venta": muestra "⚠️ Consultar con jefe de ventas"

## Credenciales en n8n

| Servicio | Credential ID | Nombre |
|----------|-------------|--------|
| Telegram | `cNYD0tar0isiULdH` | Telegram account 2 |
| OpenAI | `H9fEsQq5kGztwbzW` | OpenAi account |

## Cómo actualizar el workflow via API

```python
import json, urllib.request, ssl

API_KEY = "..."  # Ver settings.json de Claude Code
WORKFLOW_ID = "F2ipp8XPwX5AQ6Dp"

body = json.dumps(workflow_dict).encode('utf-8')
req = urllib.request.Request(
    f"https://n8n.tubosa.net/api/v1/workflows/{WORKFLOW_ID}",
    data=body, method='PUT',
    headers={'X-N8N-API-KEY': API_KEY, 'Content-Type': 'application/json'}
)
with urllib.request.urlopen(req, context=ssl.create_default_context()) as resp:
    print(json.loads(resp.read()).get('updatedAt'))
```

## Historial de cambios

### v5 → v5.1 (2026-04-17)
- Implementado código JavaScript en `Tool Buscar Productos`: búsqueda fuzzy en inventario via Google Sheets
- Implementado código JavaScript en `Tool Ver Precio Lista`: búsqueda fuzzy en precios via Google Sheets
- Ambos tools ahora piden aclaración cuando hay demasiados resultados o la búsqueda es ambigua
- Actualizado system prompt para que el agente transmita preguntas de aclaración al usuario
- Actualizado description de ambos tools para que el agente LLM sepa cuándo usarlos

## Pendiente / Mejoras futuras

- Caché de datos de Google Sheets para evitar fetch en cada consulta
- Tool para buscar en ambas hojas simultáneamente (inventario + precio en una sola consulta)
- Manejo de sinónimos venezolanos (ej: "niple" vs "unión", medidas en pulgadas vs mm)
