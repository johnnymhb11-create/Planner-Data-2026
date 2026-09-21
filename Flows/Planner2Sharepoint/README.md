# Planner a Sharepoint List (flujo corregido)

Flujo de Power Automate: hourly recurrence que lee las tareas de un plan de
Planner y hace upsert (agregar/actualizar) en la lista de SharePoint,
identificando cada fila por el ID único de la tarea de Planner (guardado en
la columna `Title`).

> La lista de SharePoint original fue eliminada; el flujo apunta ahora a la
> tabla `87f7e23d-8407-4cd8-9bdb-ab599af14924`, cuyas columnas ya no son
> `field_1`…`field_14` genéricos sino nombres que corresponden a los campos
> de Planner (ver mapeo abajo).

## Diagnóstico: por qué "rehacía cada tarea una y otra vez"

El flujo original, por cada tarea de Planner, hacía dos llamadas HTTP
secuenciales a SharePoint dentro del `Apply_to_each`:

1. `Get_items` con `$filter=Title eq '<id_tarea>'` para ver si ya existía.
2. Según el resultado: `Create_item` o `Update_item`.

Con eso hay dos problemas que se agravan a medida que crece la cantidad de
tareas/filas:

- **`Title` no es una columna indexada.** Cuando la lista de SharePoint
  supera el umbral de vista (5000 elementos), las consultas con `$filter`
  sobre una columna no indexada empiezan a fallar
  ("list view threshold exceeded"). Si eso ocurre, el flujo no logra
  confirmar que la tarea ya existe y termina yendo siempre por la rama de
  `Create_item`, creando un duplicado nuevo en cada corrida — exactamente el
  síntoma de "rehacer cada tarea una y otra vez".
- **No hay control de concurrencia en el disparador.** Al hacer 2 llamadas
  HTTP por tarea, si el plan tiene muchas tareas la corrida puede tardar más
  de una hora. Como el trigger no tenía `concurrency.runs` limitado, la
  siguiente ejecución horaria podía arrancar antes de que la anterior
  terminara. Dos corridas en paralelo revisando la misma tarea (patrón
  "leer y luego decidir", no atómico) pueden concluir ambas que la tarea "no
  existe" y crearla duplicada.

## Qué se cambió

1. **Una sola lectura de SharePoint por corrida**, en vez de una por tarea:
   `Get_all_Sharepoint_items` trae `Id` y `Title` de todos los elementos una
   vez (con paginación hasta 100k), antes del `Apply_to_each`. Dentro del
   loop, `Find_existing_item` busca la tarea en memoria con
   `filter(...)` / `first(...)`, sin llamadas HTTP adicionales. Esto elimina
   el problema del umbral de vista (ya no se filtra por `Title` en
   SharePoint) y reduce las llamadas de 2×N a N+1, bajando drásticamente el
   tiempo de ejecución.
2. **Control de concurrencia en el trigger** (`concurrency.runs = 1`) para
   que nunca haya dos corridas del flujo activas al mismo tiempo.
3. Corregido `Update_item`: no seteaba `field_14` (asignados cuando la tarea
   se completa) como sí lo hacía `Create_item`; ahora ambas ramas quedan
   consistentes.

## Mapeo de campos (Planner → SharePoint)

La nueva lista fue creada con el esquema del export nativo de Planner
("Datos de Planner"). Se hizo viajar cada columna de la hoja **Tareas** del
Excel enviado que tiene equivalente en el esquema de SharePoint, usando el
**nombre interno** real de la columna (tomado del `ListSchema` del CSV, no
el display name):

| Columna en Excel "Tareas" | Campo interno SharePoint | Origen en el flujo |
|---|---|---|
| Id. de tarea | `Title` | `id` de la tarea (clave de upsert) |
| Nombre de la tarea | `Nombredelatarea` | `title` |
| Depósito | `Dep_x00f3_sito` | `bucketId` (id crudo, igual que el Excel) |
| Estado | `Estado` | `percentComplete` traducido a texto (0→"No iniciado", 50→"En curso", 100→"Completado") |
| Priority | `Priority` | `priority` traducido a texto (1→"Urgente", 3→"Importante", 5→"Media", resto→"Baja") |
| Asignado a | `Asignadoa` | `_assignments[].userId` unidos con `;` (id crudo, igual que el Excel) |
| Fecha de creación | `Fechadecreaci_x00f3_n` | `createdDateTime` |
| Fecha de vencimiento | `Fechadevencimiento` | `dueDateTime` |
| Fecha de inicio | `Fechadeinicio` | `startDateTime` |
| Con retraso | `Conretraso` | calculado: `'true'` si hay `dueDateTime`, ya pasó, y la tarea no está al 100%; si no, `'false'` |
| Fecha de finalización | `Fechadefinalizaci_x00f3_n` | `completedDateTime` (la columna es de tipo Texto en el nuevo esquema) |
| Completado por | `Completadopor` | `completedBy/user/id` (id crudo) |
| Notas | `Notas` | `description` de la tarea, vía una llamada nueva `Get_task_details` (Planner no incluye la descripción en `List tasks`, hay que pedirla aparte por tarea) |

Columnas del Excel **sin** equivalente en el esquema de SharePoint (no
viajan): Objetivo, Creado por, Es periódica, Elementos de la lista de
comprobación completados, Elementos de la lista de comprobación, Etiquetas.

**Pendiente de verificar tras importar** (no lo pude confirmar sin acceso al
conector en vivo de este tenant):
- El `operationId` de la acción "Obtener detalles de tarea" de Planner lo
  puse como `GetTaskDetails_V2`. Si al importar Power Automate marca esa
  acción como inválida, hay que abrirla en el editor, buscar la acción
  correcta de Planner para leer notas/descripción de una tarea por su id, y
  reemplazar `Get_task_details` por esa.
- Que la ruta `body('Get_task_details')?['description']` sea efectivamente
  donde esa acción devuelve el texto de notas.

## Recomendaciones adicionales (fuera del flujo)

- **Limpiar duplicados existentes**: mientras el bug estuvo activo se
  pudieron haber creado filas duplicadas con el mismo `Title` (ID de
  tarea). Conviene correr una limpieza una sola vez (dejar la fila más
  reciente por `Title` y borrar el resto) antes de reactivar el flujo
  corregido.
- **Indexar la columna `Title`** en la lista de SharePoint (o marcarla como
  "Enforce unique values"), como resguardo adicional para que nunca se
  puedan volver a crear duplicados aunque algo falle en el flujo.

## Cómo reimportar

1. En Power Automate → Soluciones, importar el paquete
   `Microsoft.Flow/flows/72a5622b-6b1e-4dd0-a35a-a1ef5560ccda/` de esta
   carpeta (o el .zip generado a partir de ella) como actualización del
   flujo existente "Planner a Sharepoint List".
2. Al importar, reconectar las conexiones de Planner y SharePoint si se
   piden.
3. Verificar que el trigger quedó con "Concurrency Control" activado en 1.
