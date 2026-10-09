# TaskFlow · Handoff

Estado a fecha 2026-10-09. Este repositorio contiene solo TaskFlow. Para
trabajar en él, lee primero este archivo, luego `README.md`, `PLAN.md` y
`CLAUDE.md` / `AGENTS.md`, que son las reglas del proyecto.

## Origen

Hasta el 2026-10-09 TaskFlow vivía en la carpeta `taskflow/` de la rama
`claude/taskflow-app-t97qt5` del repositorio `Vict0rL1/Sport-Betting`
(antes llamado `Vict0rL1/s`), con su PR #8 en borrador. Antes de llamarse
TaskFlow fue "Cumbre", en la carpeta `cumbre/`.

Se movió aquí con todo su historial:

- Los 78 commits que tocaban `cumbre/`, `taskflow/` o los dos workflows de
  TaskFlow, con los mismos mensajes, fechas y autores. Quedan fuera solo los
  2 commits de la antigua app de presupuesto sobre la que se apoyaba la rama.
- Los 176 archivos son idénticos a los de la rama original.
- Al pasar de la carpeta a la raíz cambian los identificadores de los
  commits. Los originales siguen en `Sport-Betting`, y la conversación de
  revisión sigue en su PR #8, ya cerrado.

Cambios hechos al moverlo, en un solo commit encima del historial:

- `.github/workflows/taskflow-ci.yml` corre en cada PR y en cada push a
  `main`, desde la raíz.
- `.github/workflows/taskflow-clock.yml` no cambia de comportamiento; solo
  se ajustó un comentario que daba por hecho un repositorio público.
- `README.md`, paso 9: en Vercel el directorio raíz es `/`.

## Qué hay que hacer a mano

1. **Secretos del reloj.** En este repositorio, Settings → Secrets and
   variables → Actions, crear `APP_URL` (la URL de la app en Vercel, sin
   barra al final) y `CRON_SECRET` (el mismo valor que está en las variables
   de Vercel). En el repositorio anterior el reloj nunca llegó a ejecutarse,
   porque GitHub solo lanza tareas programadas desde la rama por defecto y
   allí la rama por defecto era otra. Aquí `main` es la rama por defecto,
   así que empezará a llamar a `/api/sync` cada hora en cuanto tenga los
   secretos.
2. **Vercel.** En el proyecto que ya existe: Settings → Git, conectar este
   repositorio y cambiar Root Directory de `taskflow` a `/`. Así se
   conservan el dominio, las variables y las URL de redirección registradas
   en Supabase, Google y Canvas.
3. Lo que el PR #8 ya listaba como pendiente: correr `supabase/schema.sql`
   en Supabase, las variables en Vercel y, si quieres Calendar, el cliente
   OAuth. Todo está en `README.md`, paso 2c.

## Resumen del trabajo hasta la migración

Descripción del PR #8, escrita por el chat que construyó TaskFlow. Las rutas
`taskflow/…` de entonces son ahora la raíz.

### Verla funcionando

```bash
npm install && npm run demo
```

Entra con `victor@ejemplo.com` / `contrasena`. El demo levanta un Supabase
falso local (PostgREST y GoTrue reducidos a lo que la app usa) sobre Postgres
de verdad (PGlite), con el esquema real, **la RLS activa** y el mismo tope de
1000 filas por lectura que Supabase. Está sembrado con el semestre real de
otoño 2026 y compila y sirve en modo producción.

**CI:** `.github/workflows/taskflow-ci.yml` corre tests, tipos, lint, build y
`check:secrets`. No usa ningún secreto. Al cerrar el PR: **392 tests en 36
archivos**, en verde.

### British Columbia dejó de cambiar la hora

Desde el **1 de noviembre de 2026** BC se queda en **UTC-7 todo el año**. Un
Node con una base de zonas anterior al cambio (por ejemplo tzdata 2025c)
seguiría atrasando la hora, y TaskFlow mostraría clases, deadlines y avisos
una hora corridos.

- `lib/date.ts` fija UTC-7 para `America/Vancouver` desde ese instante; con
  un Node al día da exactamente lo mismo. El generador del semestre usa la
  misma regla, y un test comprueba que los dos coincidan.
- Los tests de cambio de hora pasaron a usar transiciones que ya ocurrieron
  (2 nov 2025 y 8 mar 2026). Un bloque nuevo fija la regla de BC, y pasa
  igual con tzdata vieja (local) que con la nueva (CI).
- **Si importaste un `.ics` antes de este cambio, vuelve a importarlo**: las
  clases de noviembre y diciembre pueden haber quedado una hora tarde.
  Reimportar no duplica nada.

### Segunda ronda: que nada se pare en silencio, y planificación académica

Fases A→K del pedido, en orden y sin avanzar con tests rotos.

- **Producción (A–C).**
  - Errores con código estable y logs JSON que tachan secretos.
  - Canvas: timeouts y reintentos; un 429 ya no se lee como token vencido;
    después de un fallo se reintenta en una hora; el último éxito se guarda
    aparte del último intento; lo borrado no resucita; lo entregado se
    completa; no hay sync a ciegas después de una lectura incompleta.
  - El reloj deja latido. Push y Telegram clasifican sus fallos.
  - **Ajustes → Estado del sistema**, con botones de prueba. Si algo falla,
    aparece un aviso arriba de cada vista y una nota en el aviso de la
    mañana.
  - **Actividad** de usuario, Canvas, sistema y Claude, con 90 días de
    retención y sin secretos.
- **Planificación (D–F).**
  - Tipo, peso, curso y dificultad por tarea.
  - Carga de las próximas semanas, determinista.
  - **Plan de preparación** y **Replanificar**: el servidor calcula y Claude
    sólo nombra las sesiones. Siempre es una propuesta que hay que confirmar.
  - Cronómetro por tarea.
- **Uso diario (G–J).**
  - Paleta ⌘K y búsqueda de texto completo.
  - Papelera de 30 días.
  - Respaldo v2, que se importa con vista previa y sin pisar nada.
  - PWA instalable con captura sin conexión.
  - Hoy ordenado por lo que importa ahora; carga por día en Semana.
  - Rutinas con cumplimiento de 7 y 30 días, racha opcional y pausa.
- **Google Calendar (K).** Sólo lectura, con permiso cifrado y sync
  idempotente cada 3 h; nunca escribe en Google.

### Revisión posterior: bugs que no se veían

- **El tope de 1000 filas de Supabase.**
  - Pasadas las mil tareas, la vista perdía justo las más nuevas.
  - **Canvas pisaba tareas editadas y desmarcaba las hechas.**
  - El respaldo salía incompleto sin decirlo.
  - Ahora toda lectura que tiene que verlo todo pide de a mil, y el harness
    impone el tope para que esto no vuelva a pasar sin que se vea.
- **Captura sin conexión.** Un error pasajero de la base al volver la red
  tiraba lo anotado; ahora queda en la cola. Un bloque sin hora se guarda
  como nota en vez de perderse.
- **Google Calendar.** El calendario principal no se leía si Google no
  mandaba `selected`.
- **Accesibilidad** (auditoría con axe en todas las vistas, celular y
  escritorio, claro y oscuro):
  - Contraste del texto secundario a 4.5:1.
  - `aria-current` en los filtros.
  - Listas de definición válidas.
  - Encabezados en orden.
  - `<main>` en el login.

**Base de datos.** Todo está en `supabase/schema.sql`, idempotente. Hay dos
tablas nuevas, `activity_log` y `gcal_links`, cada una con RLS y su política;
las pruebas de aislamiento entre usuarios las cubren. **Hay que volver a
correr el archivo en Supabase.**

### Lo que falta

- **Nada de esto ha hablado con los servicios reales.** Canvas, Google
  Calendar, Anthropic y la entrega de push están probados contra servidores
  falsos.
- **Configuración manual**, ver "Qué hay que hacer a mano" arriba.
