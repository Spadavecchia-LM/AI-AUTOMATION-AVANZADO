# Pre entregable: Sincronización del Cerebro Agéntico con Ecosistemas de Negocio

**Autor:** Leonardo Spadavecchia
**Herramienta:** n8n
**Blueprint:** `2026-11-11_workflow_cerebro_agentico_n8n.json`

## Descripción

El workflow recibe emails entrantes y descarta respuestas automáticas para no generar bucles. Con IA clasifica cada mensaje y redacta una respuesta. Después registra o actualiza al remitente en el CRM sin generar duplicados. La respuesta queda como borrador en Gmail para que una persona la apruebe antes de enviarla (Human-in-the-loop), y el equipo recibe un aviso por Telegram.

## Conectores utilizados

| Rol | Herramienta | Operaciones |
|---|---|---|
| CRM | Google Sheets (hoja `clientes`) | Lectura: buscar contacto por email. Escritura: actualizar o agregar fila |
| Casilla de correo | Gmail | Lectura: emails no leídos. Escritura: solo crear borradores |
| Canal del equipo | Telegram | Escritura: mensaje a un único chat |
| Modelo de IA | Anthropic (Claude) | Clasificación y redacción con salida estructurada |

Elegí Google Sheets como CRM y Telegram como canal del equipo porque facilitan las pruebas. El diseño se traslada a HubSpot, Pipedrive o Slack cambiando solo los nodos de conexión.

## Diagrama del flujo

```
[Gmail Trigger: email entrante]
        │
        ▼
① [IF: ¿es auto-reply?] ── Sí ─▶ [NoOp: corta el bucle]
        │ No
        ▼
④ [Set: limpiar payload] → [Filter: validar email]
        │
        ▼
   [AI Agent: clasifica y redacta] ◀── Anthropic Chat Model + Structured Output Parser
        │
        ▼
   [Set: preparar registro CRM]
        │
        ▼
② [Look up en Sheets] → [Merge por email] → [IF: ¿existe?]
                                         ┌────┴────┐
                                         Sí        No
                                         ▼          ▼
                                     [Update]   [Create]
        │
        ▼
③ [Gmail: Create Draft (HITL)]
        │
        ▼
   [Set: payload Telegram] → [Telegram: notificar equipo]
```

## Cumplimiento de la consigna

| Requisito | Implementación |
|---|---|
| Tres conectores nativos (CRM, correo, canal) | Google Sheets, Gmail y Telegram, todos nodos nativos de n8n |
| Autenticación con semáforo en verde | Gmail y Sheets autentican con OAuth2. Telegram usa token de bot y Anthropic usa API key, porque esos servicios no ofrecen OAuth2 |
| Mínimo privilegio en lectura y escritura | Gmail lee no leídos y solo escribe borradores. Sheets lee por email y escribe 4 columnas en el update. Telegram escribe en un único chat ID |
| Nodo IF inmediatamente después del trigger | `¿Es auto-reply?` es el primer nodo después de `Gmail Trigger` |
| Ignorar Auto-reply, Out of office, Undeliverable y no-reply@ | Condiciones OR sin distinguir mayúsculas sobre el asunto (inglés y español) y sobre el remitente (`no-reply`, `noreply`, `mailer-daemon`, `postmaster`) |
| Look up antes de Create (evitar 409) | `Look up cliente` busca por email. Si existe se actualiza la fila, si no se crea. Así no hay duplicados |
| Salida de correo solo como Create Draft (HITL) | El nodo Gmail usa exclusivamente `Draft → Create`. El workflow no tiene ningún nodo que envíe emails |
| Set o filtro antes de la mensajería | `Payload Telegram` arma un texto corto y recortado, sin HTML ni binarios. Además, `Limpiar payload` reduce el email antes de llegar a la IA |
| Test de regresión manual | Ver la sección "Plan de pruebas" |

## Decisiones de diseño

- **Validación antes de la IA.** El Set y el Filter van antes del AI Agent. Así el modelo recibe solo remitente, asunto y cuerpo (máximo 3000 caracteres), y un email vacío o mal formado se descarta antes de llegar a Sheets (evita un 400).
- **Merge después del Look up.** Si llegan varios emails en una misma ejecución, el nodo de Sheets no devuelve nada para los que no encuentra. El Merge por email asegura que todos lleguen a la decisión Update/Create.
- **Salida estructurada.** El agente devuelve JSON con categoría, prioridad, nombre, apellido, empresa, resumen y borrador. El prompt le prohíbe inventar precios, stock, fechas o aprobaciones.
- **Fechas como texto.** Los nodos de Sheets escriben en modo RAW para mantener el formato `yyyy-mm-dd`.

## Datos de prueba

La hoja `clientes` tiene 50 contactos ficticios con estas columnas:

`id_cliente`, `nombre`, `apellido`, `email`, `telefono`, `empresa`, `ciudad`, `provincia`, `segmento`, `estado`, `fecha_alta`, `ultimo_contacto`, `ultimo_asunto`, `categoria_ia`, `notas`

Los emails usan los dominios reservados `example.com`, `example.org` y `example.net`, así que ningún borrador puede llegar a una persona real.

## Cómo importarlo

1. En n8n: *Workflows → Import from File* y seleccionar el blueprint.
2. Asignar las credenciales:
   - Gmail OAuth2: `Gmail Trigger` y `Crear borrador (HITL)`.
   - Google Sheets OAuth2: `Look up cliente`, `Update contacto` y `Crear contacto`.
   - Anthropic: `Anthropic Chat Model` (elegir el modelo desde el desplegable).
   - Telegram: `Notificar equipo`.
3. Apuntar los tres nodos de Sheets a una hoja propia con las mismas columnas. El blueprint referencia la hoja de prueba del autor.
4. Reemplazar `TU_CHAT_ID` en `Notificar equipo`.
5. Verificar que todas las credenciales queden en verde.

## Plan de pruebas

Ejecutar cada nodo con *Execute step* y después el flujo completo con *Execute Workflow*. Marcar cada caso después de comprobarlo.

- [ ] Email con asunto "Out of office": termina en el NoOp, no llama a la IA.
- [ ] Email desde un remitente nuevo: se crea una fila con estado Lead, se crea el borrador y llega el aviso a Telegram.
- [ ] Segundo email del mismo remitente: no se duplica la fila, se actualizan `ultimo_contacto`, `ultimo_asunto` y `categoria_ia`.
- [ ] Email desde un contacto cargado en la hoja: sale por la rama Update.
- [ ] En Gmail, el borrador aparece en el hilo original y no se envió.

## Limitaciones conocidas

- Google Sheets no devuelve un error 409 ante duplicados. El Look up cumple la misma función que en un CRM real.
- Google no permite limitar permisos por columna. El mínimo alcanzable es por API (`gmail.readonly` + `gmail.compose`, `spreadsheets`). La credencial nativa de Gmail en n8n pide un set de scopes fijo, así que para restringirlo al máximo hace falta una credencial *Google OAuth2 API* genérica con scopes custom.
- Las opciones `sendTo` y `threadId` del nodo Gmail (Create Draft) dependen de la versión de n8n. Si no aparecen, se configuran desde la interfaz.
