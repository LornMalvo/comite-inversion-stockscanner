# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-09-30
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2d26b5934428245cd77a6c284f2a8f6a06` (2026-09-19) — confirmado con `git log -- stockscanner/app/`: es el único commit que toca ese directorio; no ha habido ningún cambio de código desde la auditoría del 2026-09-28.

## 0. Nota de cobertura (antes de los hallazgos)

Se cruzaron los 41 archivos `.py` de `stockscanner/app/` contra (a) el módulo/archivo de cada uno de los 25 hallazgos de [registro_hallazgos.md](../registro_hallazgos.md) y (b) la lista explícita de "módulos auditados el 28-sep-2026 sin hallazgos" que cierra el [informe del 2026-09-28](2026-09-28_bug_report.md). Resultado: **los 41 archivos ya han sido revisados en alguna auditoría anterior** — o tienen un hallazgo abierto, o quedaron explícitamente confirmados como "sin hallazgo" el 28-sep (`app.py`, `config_secretos.py`, `core_confluencia.py`, `core_indicadores.py`, `core_interpretar.py`, `core_ponderar.py`, `core_rastreador.py`, `core_referencias.py`, `core_timing.py`, `datos_finnhub.py`, `datos_traduccion.py`, `ui_bloque_calidad_fv.py`, `ui_bloque_plan.py`, `ui_bloque_timing.py`, `ui_estilos.py`, `ui_graficos.py`, `ui_interfaz.py`, `ui_vistas_paper.py`, `ui_vistas_rastreador.py`). No queda ningún módulo "virgen" en `stockscanner/app/`.

Por eso, en lugar de re-narrar código ya leído dos veces, se releyeron completos esos módulos "sin hallazgo" buscando específicamente casos límite (siguiendo la recomendación de proceso que el propio informe del 28-sep dejó para futuras auditorías) en vez de solo lectura superficial. No se encontró ningún bug genuino nuevo — el código está notablemente bien defendido frente a `None`/`NaN` (todo pasa por `es_dato()`/`ponderar()`) y las fórmulas revisadas (Fair Value multi-método con banda de cordura, Timing ponderado con gate de calidad, Confluencia por gaussianas) son consistentes con su propia documentación interna.

**Recomendación de proceso:** con el código 11 días sin cambiar y ya 100% cubierto por auditorías previas, el valor marginal de seguir re-leyendo los mismos módulos es bajo. Se sugiere que la próxima auditoría con "commit nuevo = 0" dedique el grueso del tiempo a funcionalidades nuevas y a pruebas de ejecución real (no solo lectura) de los motores puros (hallazgo #9, cero tests), vía más eficiente para encontrar próximos casos límite reales, como ya pasó con el hallazgo #24 (RSI en racha alcista pura, solo visible ejecutando código).

## 1. Bugs y discrepancias de datos

**No se han encontrado bugs nuevos genuinos.** Todo el código no cubierto por los 25 hallazgos del registro fue auditado ya el 2026-09-28 sin hallazgos, y no ha habido cambios de código desde entonces (commit único `48be6c2`). No se inventan bugs para rellenar esta sección.

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| — | — | — | Sin hallazgos nuevos en esta sesión | — | — |

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- **Rango de incertidumbre del Fair Value (en vez de 3 escenarios fijos).** `core_fair_value.calcular()` reporta sensibilidad en exactamente tres escenarios discretos y deterministas (`FV_ESCENARIOS`, `core_fair_value.py:282-287`), sin decir qué tan ancha es la incertidumbre real. Muestrear los multiplicadores de escenario dentro de un rango (p. ej. ligado a la dispersión histórica del PER propio ya calculada en `per_historico()`) y devolver percentiles (p10/p50/p90) sería una extensión de bajo riesgo (reutiliza `_valorar()` tal cual) y encaja con el mandato de CLAUDE.md de declarar estimaciones como tales, no como hechos.

### UI / UX
- **Slider interactivo "qué pasa si"** sobre el bloque de Fair Value: mover múltiplo/crecimiento y ver el Fair Value recalculado en vivo con la misma función pura `core_fair_value._valorar()`. Bajo coste, alto valor de confianza/pedagógico.
- **Insignia de "dato desactualizado"** cuando el precio en caché lleva más de una sesión de mercado sin refrescar en vistas críticas (Paper Trading, Cartera) — hoy `ui.frescura()` solo muestra la hora en texto pequeño, sin alerta visual.

### Arquitectura
- **Smoke test de los motores puros en CI** (complementa el hallazgo #9 ya registrado): 3-4 fixtures sintéticas por motor (`core_fair_value`, `core_timing`, `core_confluencia`, `core_ponderar`) cubriendo los casos límite ya conocidos (racha alcista pura, stop sin ATR, PEG con crecimiento negativo) evitaría reintroducir un caso límite ya corregido en un cambio futuro.

### Funcionalidades nuevas de alto impacto
- **Detector de divergencias RSI/MACD vs. precio.** Ninguno de los componentes actuales de `TIMING_PESOS` detecta divergencias clásicas (precio hace máximo más alto mientras RSI/histograma MACD hacen máximo más bajo, o viceversa en mínimos) — señal técnica estándar en cualquier screener profesional, calculable con los indicadores que ya produce `core_indicadores.py` y los pivotes que ya usa `core_confluencia.pivotes()`. Hoy la app puede dar "buen timing" en un valor con agotamiento de tendencia sin capturarlo.
- **Seguimiento activo de las condiciones de invalidación de la tesis.** Hoy `a["invalidacion"]` es texto narrativo estático (mostrado en `ui_bloque_plan.py`) que nunca se vuelve a comprobar. Estructurar cada condición como expresión cuantificable (`{"metrica": "roic", "operador": "<", "umbral": 0.10}`) guardada junto al plan y comprobada en cada análisis/rastreo nocturno, disparando alerta Telegram si se cumple, convertiría una nota cualitativa en vigilancia activa real — coherente con el "qué invalidaría la tesis" que exige CLAUDE.md en cada análisis. Especialmente relevante hoy: la tesis de AEM se apoya en que la caída reciente es un "efecto de capital circulante"/incidente puntual, no deterioro — una condición perfecta para vigilancia cuantificada automática en vez de revisión manual.
- **Alerta de concentración de cartera vía Telegram.** El peso por sector ya se calcula y se marca visualmente en `ui_graficos.grafico_sectores`, pero solo dentro de la app (nadie la ve si no abre "Gestión de Cartera" ese día). Extender `core_alertas.py` (que ya compone/envía por Telegram) con un evento de concentración sectorial en el pase nocturno reutiliza infraestructura existente.

## 3. Alertas para el debate de inversión

> Cualquier problema de datos que pueda afectar la valoración de candidatas de mercados no estadounidenses debe listarse aquí. Nota positiva: las tres candidatas de hoy (LPLA, AEM, ERO) cotizan en USD en NYSE, por lo que no se activa el hallazgo #20 (sufijos bursátiles no cubiertos) ni el hallazgo #7/#10 (conversión EUR solo fiable para EUR/USD) — a diferencia de sesiones anteriores con candidatas en HKD/KRW/DKK.

- Hallazgo #12 (abierto, alta): ningún cron corre en producción real — cualquier posición nueva hoy (si la hubiera) no recibiría alertas automáticas de stop; el seguimiento de MU y V ya en cartera, y de cualquier posición nueva, sigue siendo manual.
- Hallazgo #24 (abierto, media): RSI puede salir "sin dato" (no 100) en racha alcista pura — vigilar si alguna candidata de hoy muestra momentum técnico muy fuerte con esta lectura ausente.
- Cobertura de Finnhub fuera de EE. UU. es limitación del proveedor, no de la app — no aplica hoy porque las tres candidatas cotizan en NYSE.

---

## Resumen para Fase 2 de la sesión

Confirmado: cero commits nuevos en `stockscanner/app/` desde 2026-09-28 (único commit `48be6c2`, verificado con `git log`). Los 25 hallazgos previos siguen abiertos sin cambios de código. Cero hallazgos nuevos: se cruzaron los 41 archivos de la app contra el registro y el informe del 28-sep, confirmando que los 41 ya han sido auditados al menos una vez (con hallazgo o explícitamente "sin hallazgo"); una relectura de los módulos "limpios" buscando casos límite no encontró bugs genuinos nuevos. Prioridades de mejora propuestas: (1) detector de divergencias RSI/MACD vs. precio como nuevo componente de Timing (señal técnica profesional ausente hoy); (2) seguimiento activo y cuantificado de las condiciones de invalidación de tesis (hoy son solo texto estático, nunca se revisan contra datos nuevos). Se recomienda que futuras auditorías sin commits nuevos prioricen ejecutar los motores puros con casos límite (no solo releer código) y funcionalidades nuevas, dado el 100% de cobertura ya alcanzado.
