# Plan: Aplicación de Solicitudes de Cuentas por Pagar

Estado: BORRADOR para revisión. Sin código ni construcción hasta aprobar.
Reemplaza el enfoque de `SPEC-AppSeguimiento.md` (que era solo monitoreo); este es el flujo operativo completo.

Nota: los skills "Ponytail" y "Engineering Playbook" no existen en esta sesión (búsqueda sin resultados). Usé los skills disponibles: spec-driven-development, planning-and-task-breakdown y api-and-interface-design. Si son skills propios, súbelos/actívalos y reviso el plan contra ellos.

## 1. Problema actual
Buzón compartido "cuentas por pagar" → una persona valida y reenvía a un responsable → un flujo de Power Automate **ligado a cada persona** crea la tarea en Planner → falla de forma intermitente y es difícil de diagnosticar.

Causa raíz probable: el flujo corre con la conexión/permisos de cada persona (reglas de correo, conexiones vencidas, reenvíos que no activan el disparador). Sin registro central ni alertas de fallo.

## 2. Objetivo
Una sola app donde:
1. Cada correo que llega al buzón aparece automáticamente como una **solicitud**.
2. El revisor la valida y la **asigna con un dropdown** a la persona responsable.
3. Al asignar, se envía un **correo** informando "La solicitud X fue asignada a Y".
4. Todo queda registrado (quién, cuándo, estado) en un lugar central.

## 3. Supuestos (corrígeme)
1. Microsoft 365 con Power Apps y Power Automate (ya usan Power Automate y SharePoint).
2. Existe un único revisor principal (tú) y 1–2 suplentes; los responsables son ~5–15 personas.
3. El correo de asignación va al responsable (con copia a ti y al buzón); no solo a ti.
4. Se conserva Planner/Excel mientras el reporte de Power BI `Planner CXP 2026 2.0` dependa de él (ver decisión D3).
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
                              (3) Flujo "Asignación"
                                 ├─ envía correo (asignado + revisor, desde el buzón)
                                 ├─ actualiza estado/fechas
                                 └─ [opcional D3] crea tarea en Planner
                              (4) Flujo "Monitor": alerta si algo falla o hay solicitudes sin asignar > N horas
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

Estados válidos y transiciones: Nueva → Asignada → En curso → Completada; Nueva → Descartada; Asignada → Reasignada (misma tabla, nuevo AsignadoA + registro).

## 7. Pantallas (v1)
1. **Bandeja**: lista de solicitudes Nuevas (primero las más antiguas), filtros por estado/remitente.
2. **Detalle/Asignar**: asunto, remitente, cuerpo, adjuntos, dropdown de persona, prioridad/categoría, botón **Asignar** (confirma y dispara flujo).
3. **Mis solicitudes** (responsables): solo lo asignado a ellos, marcar En curso/Completada.
4. **Historial** (revisor): todo, con tiempos (para reporte).

## 8. Decisiones que necesito de ti
- **D1** ¿Opción A (recomendada) o B/C?
- **D2** ¿A quién va el correo de asignación: responsable + tú + buzón, o solo tú?
- **D3** ¿Seguimos creando la tarea en Planner (para no romper el reporte de Power BI) o el reporte pasa a leer la lista de SharePoint? Recomendado: crear ambas en una fase transitoria, luego migrar el reporte.
- **D4** ¿Los responsables usan la app (marcar En curso/Completada) o solo reciben el correo y tú actualizas?
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
- T2.3 Flujo Asignación: correo + registro (+ Planner si D3). *Verify:* correo llega a asignado y revisor con asunto/enlace correctos.

**Fase 3 – Responsables y fiabilidad**
- T3.1 "Mis solicitudes" y cambio de estado (si D4). *Verify:* un responsable solo ve lo suyo.
- T3.2 Flujo Monitor: alertas de fallo y de "sin asignar > N horas". *Verify:* forzar un error y confirmar alerta.
- T3.3 Reasignación con historial.

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
| El reporte de Power BI depende de Planner | Fase transitoria con ambos (D3) |
| Un solo punto de fallo (el flujo) | Monitor + co-propietarios + log de errores |
| Resistencia al cambio | Piloto en paralelo y apagado gradual |

## 11. Criterios de éxito
- 100% de correos del buzón aparecen como solicitud en < 5 min, sin duplicados.
- Asignar toma ≤ 3 clics y el correo sale en < 1 min.
- 0 fallos silenciosos: todo error genera alerta y registro.
- Tiempo medio de asignación medible desde la app.
