# Plan: Aplicación de Solicitudes de Cuentas por Pagar

Estado: BORRADOR para revisión. Sin código ni construcción hasta aprobar.
Reemplaza el enfoque de `SPEC-AppSeguimiento.md` (que era solo monitoreo); este es el flujo operativo completo.

Nota: los skills "Ponytail" y "Engineering Playbook" no existen en esta sesión (búsqueda sin resultados). Usé los skills disponibles: spec-driven-development, planning-and-task-breakdown y api-and-interface-design. Si son skills propios, súbelos/actívalos y reviso el plan contra ellos.

## 1. Problema actual
Buzón compartido "cuentas por pagar" → una persona valida y reenvía a un responsable → un flujo de Power Automate **ligado a cada persona** crea la tarea en Planner → falla de forma intermitente y es difícil de diagnosticar.

Causa raíz probable: el flujo corre con la conexión/permisos de cada persona (reglas de correo, conexiones vencidas, reenvíos que no activan el disparador). Sin registro central ni alertas de fallo.

## 2. Objetivo (actualizado con tu aclaración)
Una herramienta centralizadora y automatizada de seguimiento, conectada a Outlook, donde:
1. Cada correo que llega al buzón aparece automáticamente como una **solicitud** (ninguno se pierde; hay conciliación contra el buzón).
2. La persona encargada **ve el correo dentro de la app** y lo **asigna con un dropdown** al responsable.
3. Al asignar se dispara un flujo que **crea la tarea en Planner** (con el mismo formato de hoy) para seguir alimentando las métricas del reporte de Power BI.
4. La app muestra **quién tiene qué** (asignaciones por persona) y las **solicitudes pendientes** (sin asignar / asignadas sin completar), con antigüedad.
5. (Opcional, D2) Se envía correo de aviso de la asignación.

Decisión ya tomada: **Planner se mantiene** como destino de las tareas (resuelve D3).

## 3. Supuestos (corrígeme)
1. Microsoft 365 con Power Apps y Power Automate (ya usan Power Automate y SharePoint).
2. Existe un único revisor principal (tú) y 1–2 suplentes; los responsables son ~5–15 personas.
3. El correo de asignación va al responsable (con copia a ti y al buzón); no solo a ti.
4. Planner sigue siendo la fuente de las métricas: el reporte `Planner CXP 2026 2.0` lee el Excel `Cuentas por pagar y Pagos 2026.xlsx` (hoja `Tareas`, alimentada desde Planner). La app NO reemplaza Planner; lo alimenta. Hay que confirmar cómo se genera ese Excel (D9).
5. Volumen bajo/medio (decenas a pocos cientos de correos/día).

## 4. Opciones de arquitectura

| Opción | Descripción | A favor | En contra |
|---|---|---|---|
| **A (recomendada)** | Power Apps (canvas) + **Lista de SharePoint** + Power Automate con conexión central | Licencias estándar, todo en M365, rápido de construir, Power BI lee SharePoint fácil | Límite de delegación con listas muy grandes; menos control fino de roles |
| B | Power Apps + Dataverse | Mejor seguridad/roles, escalable | Licencia premium, más complejo |
| C | Web app propia + Microsoft Graph | Control total | Requiere desarrollo, hosting, registro Entra y consentimiento de admin; mantenimiento |

Recomendación: **A**. Si luego crece o se necesitan roles estrictos, migrar a B.

## 5. Arquitectura (opción A)

```
Buzón compartido ──(1) Flujo "Ingesta"──▶ Lista SharePoint "Solicitudes"
                                              │
                      Power Apps "Solicitudes CxP" ◀──(2) revisor ve bandeja
                                              │ asigna (dropdown de lista "Personas")
                                              ▼
                              (3) Flujo "Asignación" (disparado desde la app)
                                 ├─ crea la tarea en Planner (plan, bucket, título, asignado, fechas, descripción + enlace al correo)
                                 ├─ guarda PlannerTaskId en la solicitud y marca Estado = Asignada
                                 └─ [opcional D2] envía correo de aviso
                              (4) Flujo "Sincronización Planner": actualiza en la solicitud el progreso (No iniciado/En curso/Completado)
                                  de la tarea de Planner, para que la app muestre el estado real
                              (5) Flujo "Monitor/Conciliación": alerta de fallos, de solicitudes sin asignar > N horas, y
                                  compara cada noche el buzón vs. la lista para detectar correos no ingestados
```

**Por qué esto arregla la fiabilidad:**
- Un solo flujo central, propiedad de una cuenta de servicio/propietario con co-propietarios (no uno por persona).
- Disparador "Cuando llega un correo nuevo a un buzón compartido (V2)" sobre el buzón mismo; ya no depende de reenvíos individuales.
- Idempotencia: se guarda `InternetMessageId` único para evitar duplicados.
- Reintentos + rama de error que notifica y registra en una lista "Errores".
- Estado visible en la app: nada se pierde silenciosamente.

## 6. Modelo de datos (borrador)

**Solicitudes**: Id, MessageId (único), Asunto, Remitente, Cuerpo (texto), LinkCorreo, FechaRecibido, Adjuntos (carpeta/enlaces), Estado (Nueva · Asignada · En curso · Completada · Descartada), Categoría/Bucket (CxP, Diarios de Pago, Presupuesto…), Prioridad, AsignadoA (Persona), AsignadoPor, FechaAsignación, FechaCompletada, Notas.
**Personas** (catálogo del dropdown): Nombre, Correo, Activo, Bucket(s) que maneja.
**Errores** (log del flujo): fecha, flujo, mensaje, MessageId.

Campos adicionales en **Solicitudes**: PlannerTaskId, PlannerBucket, ProgresoPlanner (sincronizado), FechaVencimiento.
El **Excel/Planner resultante debe conservar las columnas que usa el reporte** (Id. de tarea, Nombre de la tarea, Nombre del depósito=bucket, Progreso, Asignado a, Fecha de creación/inicio/vencimiento/finalización, etc.); las tareas creadas por la app deben usar los mismos nombres de bucket (Cuentas por Pagar, Asignación de Diarios de Pagos, Diarios de Pago, Presupuesto, Presupuesto - En Espera de Respuesta).

Estados válidos y transiciones: Nueva → Asignada → En curso → Completada; Nueva → Descartada; Asignada → Reasignada (misma tabla, nuevo AsignadoA + registro).

## 7. Pantallas (v1)
1. **Bandeja**: lista de solicitudes Nuevas (primero las más antiguas), filtros por estado/remitente.
2. **Detalle/Asignar**: asunto, remitente, cuerpo, adjuntos, dropdown de persona, prioridad/categoría, botón **Asignar** (confirma y dispara flujo).
3. **Quién tiene qué**: asignaciones agrupadas por persona (conteo y lista de abiertas, antigüedad en días hábiles, enlace a la tarea de Planner).
4. **Pendientes**: sin asignar + asignadas sin completar, ordenadas por antigüedad, con alerta de atraso.
5. **Historial**: todo, con tiempos de asignación y cierre.
(La vista "Mis solicitudes" para responsables queda opcional, según D4; el trabajo diario de ellos sigue en Planner.)

## 8. Decisiones que necesito de ti
- **D1** ¿Opción A (recomendada) o B/C?
- **D2** ¿A quién va el correo de asignación: responsable + tú + buzón, o solo tú?
- **D3** RESUELTA: se crea la tarea en Planner al asignar.
- **D4** ¿Los responsables usan la app o solo Planner? (Asumo solo Planner; la app lee su progreso.)
- **D9** ¿Qué plan de Planner y qué buckets se usan, y cómo se exporta hoy el Excel que lee Power BI (manual, flujo, Excel de Planner)? La tarea creada debe caer ahí.
- **D10** ¿Una solicitud = una tarea, o un correo puede generar varias tareas (p. ej. un correo con varias facturas)?
- **D11** ¿Quién usa la app para asignar: solo tú o varios encargados?
- **D5** ¿Adjuntos y cuerpo se guardan en SharePoint o solo el enlace al correo?
- **D6** ¿Quién es la cuenta propietaria del flujo (cuenta de servicio vs. tu usuario)? ¿Quién da permiso de "enviar como" el buzón?
- **D7** ¿Categoría/bucket se asigna manualmente o se detecta por asunto/palabras clave?
- **D8** ¿Se necesitan los correos históricos o solo los nuevos desde el arranque?

## 9. Fases, tareas y verificación

**Fase 0 – Prerrequisitos (bloqueante)**
- T0.1 Confirmar licencias/permisos: crear flujos y apps, acceso al buzón compartido, "Enviar como". *Verify:* flujo de prueba envía un correo desde el buzón.
- T0.2 Crear sitio/lista SharePoint y cuenta propietaria. *Verify:* acceso confirmado.

**Fase 1 – Ingesta (vertical mínima)**
- T1.1 Listas Solicitudes/Personas/Errores. *Verify:* columnas y tipos revisados.
- T1.2 Flujo Ingesta con idempotencia y manejo de errores. *Verify:* enviar 3 correos de prueba (con adjunto, sin adjunto, reenviado) → 3 filas, 0 duplicados; reenviar el mismo → sigue 1 fila.

**Fase 2 – App: bandeja + asignación**
- T2.1 Bandeja y detalle (solo lectura). *Verify:* revisor ve las solicitudes de prueba.
- T2.2 Dropdown de personas + botón Asignar (cambia Estado/Fecha). *Verify:* datos actualizados en la lista.
- T2.3 Flujo Asignación: crea tarea en Planner (bucket correcto, asignado, descripción con enlace al correo), guarda PlannerTaskId, aviso por correo si D2. *Verify:* tarea aparece en Planner del responsable con los campos esperados y la fila queda "Asignada" con PlannerTaskId; reintentar no crea tarea duplicada.

**Fase 3 – Seguimiento y fiabilidad**
- T3.1 Vistas "Quién tiene qué" y "Pendientes". *Verify:* conteos por persona coinciden con Planner.
- T3.2 Flujo Sincronización de progreso Planner → solicitud. *Verify:* completar una tarea en Planner se refleja en la app.
- T3.3 Flujo Monitor/Conciliación: fallos, "sin asignar > N horas" y comparación buzón vs. lista. *Verify:* forzar un error y quitar una fila de prueba; ambos se alertan.
- T3.4 Reasignación con historial (actualiza la tarea de Planner).
- T3.5 Verificación con Power BI: la tarea creada por la app aparece en el reporte con bucket, fechas y asignado correctos.

**Fase 4 – Piloto y cambio**
- T4.1 Piloto en paralelo con el proceso actual 1–2 semanas; comparar conteo de correos vs. solicitudes (debe coincidir 100%).
- T4.2 Capacitación breve, apagado de los flujos por persona, migración del reporte de Power BI (D3).

Orden/dependencias: 0 → 1 → 2 → 3 → 4. T2.1 y T1.2 pueden solaparse tras T1.1.

## 10. Riesgos
| Riesgo | Mitigación |
|---|---|
| No hay permiso de "Enviar como" el buzón | Enviar desde la cuenta de servicio; confirmar en T0.1 |
| Límite de delegación/umbral de 5000 filas en SharePoint | Índices en Estado/FechaRecibido, archivo mensual |
| Correos con adjuntos grandes/formatos raros | Guardar adjuntos en biblioteca y enlazar; probar en T1.2 |
| El reporte de Power BI depende de Planner | Se sigue creando en Planner con los mismos buckets/campos; T3.5 lo valida |
| Planner no permite fijar todos los campos al crear (p. ej. progreso, fecha de creación es automática) | Confirmar en T0.1 qué campos admite la acción de Planner; la fecha de creación será la de asignación, no la del correo (impacta DiasHabiles; decidir si se guarda FechaRecibido aparte) |
| Un solo punto de fallo (el flujo) | Monitor + co-propietarios + log de errores |
| Resistencia al cambio | Piloto en paralelo y apagado gradual |

## 11. Criterios de éxito
- 100% de correos del buzón aparecen como solicitud en < 5 min, sin duplicados.
- Asignar toma ≤ 3 clics y el correo sale en < 1 min.
- 0 fallos silenciosos: todo error genera alerta y registro.
- Tiempo medio de asignación medible desde la app.
