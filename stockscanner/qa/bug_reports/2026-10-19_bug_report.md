# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-19
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `28512ff68496b415a54274e6927b986f12374eda` (código de `stockscanner/app/` sin cambios desde el commit inicial `48be6c2`, confirmado de nuevo con `git log`)

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 34 | Media | `ui_vistas_rastreador.py` (`_rastrear()`) / `datos_analisis.py` (`analizar()`, `posicion_abierta()`) | El Rastreador en modo "En vivo" no pasa `posiciones=` precalculado a `datos_analisis.analizar()`, pese a que ese parámetro existe precisamente para evitar N consultas a Supabase por ticker en un lote. `tarea_rastreo.py` (el cron nocturno) sí lo precalcula una vez (`_posiciones()`) y lo pasa; `ui_vistas_rastreador.py::_rastrear()` llama `datos_analisis.analizar(t, ligero=True, origen="rastreador")` sin ese argumento | Lectura directa de `datos_analisis.py` (rama `if posiciones is not None: ... else: posicion_abierta(ticker)`) vs. `tarea_rastreo.py` (sí lo pasa) vs. `ui_vistas_rastreador.py` línea 134 (no lo pasa) | Con el tope `RASTREADOR_MAX_TICKERS`, cada rastreo manual en vivo genera hasta 2×N consultas evitables a Supabase (en vez de 2 totales); es justo el flujo que usa el propio Comité para analizar en bloque las candidatas de cada sesión, por lo que el coste innecesario recae en el uso diario de la herramienta por el Comité mismo |

**Cobertura confirmada sin hallazgos nuevos:** se revisó el código real de `config_secretos.py`, `core_interpretar.py`, `core_ponderar.py`, `core_rastreador.py`, `core_referencias.py`, `core_timing.py`, `datos_analisis.py`, `datos_traduccion.py`, `ui_bloque_calidad_fv.py`, `ui_bloque_plan.py`, `ui_bloque_timing.py`, `ui_estilos.py`, `ui_graficos.py`, `ui_vistas_paper.py` y `ui_vistas_rastreador.py` (módulos sin hallazgo explícito previo) sin encontrar más discrepancias de cálculo, unidades o periodos fiscales. El código de `stockscanner/app/` sigue sin cambios desde `48be6c2`; ningún hallazgo previo (#1-33) cambia de estado.

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- Ninguna nueva hoy; los hallazgos abiertos #26, #28, #29, #30 y #33 (Alta/Media) siguen siendo las prioridades de cálculo pendientes.

### UI / UX
- Ninguna nueva hoy más allá de lo ya registrado (#10, #11, #15, #16, #32).

### Arquitectura — Funcionalidades nuevas de alto impacto

**A. Monitor automático de condiciones de watchlist/reconsideración.** `comite_inversion/cartera/empresas_analizadas.md` acumula ya más de 110 tickers descartados o en watchlist, cada uno con una condición textual explícita de "qué cambiaría para reconsiderarlo" (p. ej. "ROIC sostenido >15%", "ruptura confirmada sobre $602"). Hoy esa condición vive solo en prosa y depende de que un analista la recuerde y la reverifique a mano sesión tras sesión — exactamente el tipo de carga que StockScanner debería absorber. Propuesta: una tabla `watchlist_condiciones` (`ticker`, `tipo`: `roic_min` | `precio_cruce` | `rs_positiva` | `upside_min` | `otro`, `umbral`, `texto`, `sesion_origen`) rellenada por el Supervisor al cerrar cada sesión con watchlist, evaluada cada noche por `tarea_rastreo.py` contra el último análisis del cron (mismo patrón que `core_alertas.py`) y que dispara una alerta de Telegram cuando se cumple (ej. "BTI: ROIC pasó de 9,68% a 16,2%, condición de reconsideración cumplida"). Reduce el riesgo de violar sin querer la regla de no-repetición y ahorra horas de research repetido en cada sesión.

**B. Puente Comité → Cartera/Paper Trading.** Las decisiones reales del Comité (MU, V, AEM, ACN, "pendiente de ejecución") viven solo en `posiciones_actuales.md`, mientras que StockScanner tiene un motor completo de Cartera y Paper Trading (`core_cartera.py`, `core_paper.py`) que hoy no sabe nada de esas posiciones salvo que alguien las introduzca a mano desde la UI. Son dos fuentes de verdad en paralelo, con riesgo de divergencia silenciosa. Propuesta: una función `sincronizar_veredicto(ticker, plan_entrada, stop_tecnico, stop_fundamental, sesion)` invocable al cerrar la Fase 4, que crea/actualiza un plan en `paper_planes` (estado "vigilancia") con los niveles exactos del Veredicto de Inversión, de forma que Paper Trading refleje automáticamente lo que el Comité decidió sin reescritura manual ni riesgo de que ambos registros se desincronicen.

## 3. Alertas para el debate de inversión

- Ninguna candidata llegó a proponerse en la Fase 1 de hoy (las tres investigaciones independientes concluyeron sin candidata válida), por lo que no hay ninguna alerta de datos que aplicar a una candidata concreta de esta sesión. Se mantiene vigente la advertencia general de sesiones anteriores (hallazgo #33) para cualquier futura candidata internacional de baja cobertura de analistas.
