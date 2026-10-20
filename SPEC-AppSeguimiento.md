# Spec: AppSeguimiento

Status: DRAFT — awaiting review. No plan or code until approved.

## Assumptions (correct me)
1. "AppSeguimiento" = a task-tracking app for the Dirección Administrativa y Financiera team, built on the same Planner export that feeds the existing Power BI report `Planner CXP 2026 2.0` (this repo). It is not an existing artifact; nothing named AppSeguimiento is in the repo.
2. Platform: Power Apps (canvas app) — the Power Platform plugins are installed and the data lives in SharePoint/Microsoft 365. Alternative: code app (React/Vite).
3. Source of truth stays the Excel `Cuentas por pagar y Pagos 2026.xlsx` (sheet `Tareas`, SharePoint `DireccinAdministrativayFinanciera/.../Proyecciones/PowerBI Data1/`). The app reads it; the Power BI report keeps consuming it unchanged.
4. v1 is read-mostly "seguimiento" (monitoring/follow-up), with light write-back limited to a follow-up note per task (see Open Questions).
5. Users: Spanish-speaking finance staff and their supervisor; UI language is Spanish (es-MX).
6. Dates/time logic reuses the report's rules (business days Mon–Fri, no holiday calendar).

## Objective
Give the team a daily operational view of Planner tasks per bucket so overdue/stalled work is spotted and followed up without opening Power BI.

Buckets (from `Tareas[NumBucket]`): Cuentas por Pagar, Asignación de Diarios de Pagos, Diarios de Pago, Presupuesto, Presupuesto - En Espera de Respuesta.

User stories:
- As a supervisor, I see per-bucket counts of tasks *No iniciado / En curso / Completado*, and tasks created yesterday.
- As an analyst, I see my assigned open tasks sorted by age (business days) and due date.
- As a supervisor, I filter by bucket, assignee, priority, month, and age category and open a task's detail (name, description, checklist progress, labels, dates).
- As a user, I flag/annotate a task with a follow-up note and next-action date (if write-back approved).

## Tech Stack
Power Apps canvas app + Excel Online (Business) connector on the existing workbook (proposed). Power Fx. No new Dataverse tables in v1.

## Data Contract (from existing semantic model)
Columns: Id. de tarea, Nombre de la tarea, Nombre del depósito (bucket), Progreso, Priority, Asignado a, Creado por, Fecha de creación/inicio/vencimiento/finalización, Es periódica, Con retraso, Completado por, Elementos de la lista de comprobación (completados), Etiquetas, Descripción, Objetivo.

Derived (must match the report):
- `DiasHabiles` = Mon–Fri days from Fecha de creación to Fecha de finalización (or today if open), inclusive of both ends.
- `Categoria de Tiempo` buckets: <5, 6–10, 11–15, 16–20, 21–25, ≥25 business days. Note: the report's thresholds are `<5, <10, <15, <20, <25`, so exactly 5/10/15/20 fall in the *next* labeled band (e.g. 5 → "Entre 6 y 10"). Spec follows the report's behavior unless told otherwise.
- Measures to port: Tareas creadas ayer (per bucket), No iniciadas/En curso hasta hoy, % del total, promedio de días hábiles por bucket/mes.

## Screens (v1)
1. **Resumen** — KPI tiles per bucket (open, created yesterday, avg business days), "Actualizado al" stamp.
2. **Tareas** — filterable/sortable list with age category chip and overdue flag.
3. **Detalle** — one task, all fields, follow-up note.
4. **Mis tareas** — list pre-filtered to the signed-in user (match on `Asignado a`).

## Commands
Exact commands depend on the platform decision. For Power Apps: `pac canvas pack/unpack` (source in `AppSeguimiento/`), Power Apps Studio for preview. For a code app: `npm run dev`, `npm run build`, `npm test`, `npm run lint`.

## Project Structure (proposed)
```
SPEC-AppSeguimiento.md          → this spec
AppSeguimiento/                 → app source (unpacked canvas app or Vite project)
Planner CXP 2026 2.0.Report/    → existing Power BI report (unchanged)
Planner CXP 2026 2.0.SemanticModel/ → existing model (unchanged; reference for logic)
tasks/plan.md, tasks/todo.md    → created in Plan phase
```

## Code Style
Spanish UI strings, English-free labels; Power Fx names mirror model names so logic can be cross-checked:
```
Set(varDiasHabiles, CountIf(ForAll(Sequence(DateDiff(Fecha, Coalesce(Fin, Today()), TimeUnit.Days) + 1, 0), Weekday(DateAdd(Fecha, Value - 1, TimeUnit.Days), StartOfWeek.Monday) < 6), Value))
```
(Exact formula to be validated against the DAX in Testing.)

## Testing Strategy
- Parity check: for a fixed snapshot of the workbook, app KPIs (per-bucket counts, avg DiasHabiles, created-yesterday) equal the Power BI report values.
- Edge cases: blank Fecha de finalización, tasks created on weekends, same-day completion, blank bucket (NumBucket null), delegation limit (>500/2000 rows).
- Manual UAT on phone and desktop with 2 real users.

## Boundaries
- Always: keep Power BI report and workbook schema unchanged; match report's business-day logic; respect SharePoint permissions.
- Ask first: write-back to the workbook, adding Dataverse/SharePoint lists, new connectors, changing bucket names/thresholds.
- Never: commit credentials/connection IDs or tenant data exports; edit the workbook's existing columns; delete rows.

## Success Criteria
- Resumen KPIs match the Power BI report within 0 difference on the same data snapshot.
- List of ≤2,000 tasks loads in <3 s and filters respond in <1 s.
- A user finds their open tasks in ≤2 taps and sees overdue ones flagged.
- Works for all 5 buckets with no blank-bucket errors.

## Open Questions
1. What did you mean by AppSeguimiento — is the assumption (task-tracking app on the Planner/Excel data) right? Any existing mockup or requirements?
2. Platform: Power Apps canvas (recommended), code app, or something else?
3. Read-only, or write-back (follow-up notes, status changes)? If write-back, where stored — new column in the workbook, SharePoint list, or Dataverse?
4. Who are the users and how is "Mis tareas" identified (Asignado a is text — display name vs email)?
5. Should business days exclude Dominican holidays? (Report does not.)
6. Keep the report's `Categoria de Tiempo` boundary quirk or fix it to the labels?
7. Notifications (e.g., Teams/email for overdue tasks) in scope?
