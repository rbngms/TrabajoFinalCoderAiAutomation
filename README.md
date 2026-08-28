# Automatización de calificación de leads con IA (n8n + Airtable)

Fondo de inversión — flujo que recibe leads por formulario, genera un briefing de ventas con IA, asigna al
asesor correcto según presupuesto, confirma atención humana por Telegram y actualiza la base de datos en
Airtable, con registro de errores automatizado.

## Entregables del repositorio

| # | Entregable | Archivo | Contenido |
|---|------------|---------|-----------|
| 1 | Diagrama de arquitectura | [`Flujo.pdf`](./Flujo.pdf) | Mapa visual del workflow de n8n: los 12 nodos del flujo (formulario, AI Agent, registro en Airtable, búsqueda de asesor, enrutamiento por Switch, confirmación humana por Telegram, actualización de registro y notificación por correo), incluyendo las ramas de error (`Create a record1`). |
| 2 | Video de ejecución del flujo | [`videoFlujo.mp4`](./videoFlujo.mp4) | Grabación de una ejecución real del workflow en n8n, de principio a fin: envío del formulario, generación del briefing por IA, asignación de asesor, confirmación por Telegram y actualización final en Airtable. |
| 3 | Matriz de optimización de costos | [`Matriz_Optimizacion_Costos.pdf`](./Matriz_Optimizacion_Costos.pdf) | Comparativa independiente de GPT-4o-mini, Claude y Batch API por tipo de tarea, con `max_tokens` estimado, justificación y ahorro proyectado. |
| 4 | Enlace a la base de datos | [`LINK_BASE_DE_DATOS.pdf`](./LINK_BASE_DE_DATOS.pdf) | Documento con el enlace funcional a la base en Airtable (tablas Clientes, Asesores y Errores). |
| 5 | Enlace al dashboard de control | [`LINK_Dashboard_Errores.pdf`](./LINK_Dashboard_Errores.pdf) | Documento con el enlace funcional a la Shared View de Airtable con KPIs y tasa de errores del sistema. |

## Enlaces funcionales

- **Base de datos (Airtable — Clientes, Asesores, Errores):** https://airtable.com/appHBqG4mqsMhhmQe/shrfbLcnqw2APyUwY
- **Dashboard de control (Shared View con KPIs y tasa de errores):** https://airtable.com/appHBqG4mqsMhhmQe/shrtOvKRebzwWdqh3

## Workflow n8n

- [`TrabajoFinal.json`](./TrabajoFinal.json) — exportación del workflow productivo. Importar en una instancia de
  n8n propia; requiere credenciales de Airtable, OpenAI y Telegram configuradas localmente (no se incluyen en
  el JSON por seguridad).

## Estructura resumida del flujo

1. **On form submission** — formulario de contacto (Nombre, DNI, Teléfono, Correo, Presupuesto, Producto de interés, Comentario).
2. **Code in JavaScript → AI Agent** — genera el briefing de ventas (perfil, gancho, puntos clave, objeciones, potencial de negocio) con salida forzada por **Structured Output Parser**. Si la API de OpenAI falla, el item sale por la rama de error (`onError: continueErrorOutput`) hacia el log de **Errores**.
3. **Create a record** — guarda al cliente en la tabla **Clientes** de Airtable.
4. **Search records** — busca el asesor correspondiente al rango de presupuesto en la tabla **Asesores**.
5. **Switch** — enruta el mensaje de Telegram al asesor asignado.
6. **Send message and wait for response(1-3)** — punto de *human-in-the-loop*: el asesor confirma si atendió al cliente e ingresa su DNI.
7. **Update record** — actualiza la fila del cliente en **Clientes** usando el DNI como llave de correlación. Si falla, también cae en la rama de error.
8. **Send a message** — correo de cierre con el resumen del lead y su estado de atención.
9. **Create a record1** — nodo receptor de todas las ramas de error anteriores; escribe en la tabla **Errores** (Fecha, Nombre Error, Error Detalle, DNI, Execution ID).

## Manejo de errores: qué está cubierto y qué falta

- **Cubierto (Resume):** fallos puntuales en la llamada a la IA (`AI Agent`) y en las operaciones de Airtable
  (`Create a record`, `Update record`). El workflow no se detiene; el error queda registrado en la tabla
  **Errores** con nodo, mensaje, DNI del cliente en curso y ID de ejecución — visible en el dashboard vía la
  tasa de errores del sistema.
- **Pendiente (Break):** un workflow separado con nodo **Error Trigger**, configurado como *Error Workflow* del
  flujo principal (Settings → Error Workflow), para capturar fallos no controlados en el resto de nodos
  (Switch, Telegram, Gmail) y alertar al administrador.

## Seguridad y resiliencia

- Minimización de datos: solo se captura lo necesario para calificar y asignar al lead (sin datos financieros sensibles).
- Human-in-the-loop: la IA nunca contacta al cliente directamente ni marca un lead como atendido por sí sola.
- Manejo de errores: log automático en Airtable + rama dedicada para no interrumpir el servicio ante fallos puntuales de la API de IA.
