# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-09-23
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** 0b989f3

Confirmación de hallazgos previos (#1-11): verificado con `git log` que ningún módulo asociado a esos 11 hallazgos ha cambiado desde el commit inicial (`48be6c2`) — siguen todos en el mismo estado que el registro. No se reprocesan como nuevos.

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 12 | **Alta** | Arquitectura/despliegue (cron GitHub Actions) | Los workflows que ejecutan el rastreo nocturno y las alertas horarias no están en la única ruta que GitHub Actions reconoce (`.github/workflows/` en la raíz del repo). Viven en `stockscanner/app/.github/workflows/{rastreo,alertas}.yml`, más un `stockscanner/app/rastreo.yml` suelto. En la raíz solo existe `notificar_sesion.yml`. | `find . -iname "*.yml"` confirma que en `.github/workflows/` de la raíz solo hay `notificar_sesion.yml`; los propios ficheros llevan en su cabecera indicaciones del tipo "copiar a `.github/workflows/rastreo.yml` (única carpeta del repo)". | Si esa copia no se ha hecho en el repositorio real desplegado, el rastreo nocturno de índices y las alertas de Telegram (niveles de plan, movimientos de cartera, screener) **no se ejecutan nunca**, pese a que la documentación interna de la app (`tarea_rastreo.py`, `tarea_alertas.py`, `ui_vistas_rastreador._nocturno`) asume que sí. Debe verificarse manualmente antes de confiar en cualquier candidata que provenga del modo Screener del Rastreador. |
| 13 | Media | `core_calidad.py` (`metricas()`, `calidad_beneficio`) | `calidad_beneficio = fcf_ttm / beneficio[-1]` mezcla FCF **TTM** con beneficio neto **anual** del último ejercicio fiscal, desalineados hasta ~12 meses. | `core_calidad.py`, función `metricas()` (bloque de `calidad_beneficio`). | Mismo patrón que el bug ya conocido #2 (P/FFO) pero en el bloque de Calidad: en empresas con FCF o beneficio que se mueve deprisa entre trimestres, la métrica "calidad del beneficio" queda distorsionada por comparar periodos distintos. |
| 14 | Media | `core_cartera.py` (`resumen()`) | Con una posición sin precio disponible (`n_sin_precio > 0`), `invertido_eur` suma el coste de todas las posiciones (incluida la sin precio), pero `latente` solo suma sobre las posiciones con precio. `retorno_total_pct = (latente + realizado) / invertido`. | `core_cartera.py`, función `resumen()`. | El retorno total de la cartera se infravalora silenciosamente: el capital de la posición sin precio pesa en el denominador sin su parte de plusvalía/minusvalía latente, sin aviso en la UI. |
| 15 | Baja/Media | `ui_vistas_analisis.py` (`_earnings()`) | Al comparar ingresos reales vs. estimados del último trimestre, el real se formatea con `ui.fmt_importe(...)` (incluye divisa y conversión a EUR) pero el estimado usa `ui.fmt_grande(...)` (sin divisa ni conversión). | `ui_vistas_analisis.py`, bloque de comparación real vs. estimado en `_earnings()`. | El usuario ve dos cifras con formato distinto sin saber si están en la misma unidad, justo en la comparación que sostiene la lectura de sorpresa de resultados. |
| 16 | Baja | `ui_metricas.py` (fila "Máximo 52 semanas") | El semáforo usa un signo simple que pinta en rojo cualquier distancia negativa al máximo de 52 semanas, lo cual ocurre casi siempre por definición. | `ui_metricas.py`, fila de "Máximo 52 semanas". | Ruido visual: una lectura "mal" casi universal entrena al usuario a ignorarla, contrario al principio de "un dato normal no debe gritar" que sigue el resto del semaforizado. |
| 17 | Baja (doc.) | `config_sectores.py` (comentario `INDUSTRIAS_REIT`) | El comentario dice que el fair value de REITs se apoya en EV/EBITDA y consenso "hasta implementar P/FFO", pero P/FFO ya está implementado en `core_fair_value.py` (método `p_ffo`) y en `FV_PESOS_REIT`. | `config_sectores.py`, comentario sobre `INDUSTRIAS_REIT`. | Comentario obsoleto que puede inducir a error sobre qué métodos cubren realmente a los REITs. |
| 18 | Baja (doc.) | `config_settings.py` (`TIMING_PESOS`, comentario) | Comentario residual/duplicado ("Momentum y flujo (27)" seguido de "(26)"); la suma real de pesos (rsi+macd+obv+adx) es 26. No afecta al cálculo. | `config_settings.py`, bloque `TIMING_PESOS`. | Ninguno funcional; solo confunde a quien audite los pesos a mano. |

No se han encontrado discrepancias nuevas de severidad crítica.

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- **Validación cruzada de fundamentales entre fuentes.** Hoy Calidad y Fair Value dependen en exclusiva de yfinance (ROIC, márgenes, deuda/EBITDA); solo earnings, recomendaciones, noticias y peers cruzan con Finnhub. Añadir una comprobación ligera contra el endpoint de métricas de Finnhub para 3-4 cifras clave, marcando un aviso (mismo patrón que el aviso de divisa en `core_fair_value.calcular`) cuando ambas fuentes difieran más de un umbral razonable. Justificación directa de esta sesión: el Analista 1 tuvo que reconciliar manualmente un ROIC de ADBE que varía entre 22% y 59-61% según metodología/fuente — exactamente el tipo de discrepancia que un chequeo automático habría señalado antes de llegar al debate.
- Corregir #13 (`calidad_beneficio` con FCF TTM vs. beneficio anual) con el mismo criterio de TTM ya aplicado al resto de Calidad.

### UI / UX
- **Aviso de "cron atascado".** Motivado por el hallazgo #12: comparar la fecha del último pase de rastreo/alertas contra la cadencia esperada y mostrar un banner en la vista Rastreador y en el resumen de Telegram si el cron lleva más de N días sin correr. Hoy no hay ninguna forma, ni en la app ni en Telegram, de detectar que el cron ha dejado de ejecutarse.
- Corregir el semáforo de "Máximo 52 semanas" (#16) usando un tramo con umbral (p. ej. neutro hasta -3%, rojo solo más allá) en vez de un simple signo.
- Unificar el formato de `_earnings()` (#15) usando `fmt_importe` también para el valor estimado.

### Arquitectura
- **Mover los workflows `rastreo.yml`/`alertas.yml` a `.github/workflows/` en la raíz del repo** (o dejar constancia verificable de que ya se hizo en el despliegue real) para que el cron nocturno y de alertas exista de verdad — máxima prioridad de esta sesión.
- (Refuerza #9, ya en el registro) los motores puros auditados hoy (`core_calidad`, `core_fair_value`, `core_ponderar`, `core_referencias`, `core_indicadores`, `core_timing`, `core_confluencia`, `core_cartera`, `core_rastreador`) siguen sin ningún test unitario; los hallazgos #13/#14 de hoy son el tipo de regresión silenciosa que un test con fixtures pequeñas habría atrapado.

## 3. Alertas para el debate de inversión

- El hallazgo #12 (cron potencialmente no operativo) no contamina ninguna cifra citada en las tres candidatas de hoy — ninguna proviene del modo Screener del Rastreador, todas se investigaron con fuentes primarias/agregadores externos vía búsqueda web. Pero es una alerta operativa real: si el comité empezara a apoyarse en el Screener del Rastreador para descubrir candidatas, debería confirmarse antes que el cron nocturno corre de verdad.
- Ninguna de las tres candidatas de hoy (ADBE, PRX, 005930.KS) es un REIT, por lo que el hallazgo #2 (P/FFO) no aplica hoy. Las tres tienen historial largo, por lo que el hallazgo #1 (stop DCA sin ATR) tampoco aplica hoy, pero sigue como bloqueante documentado.
- **Relevante para el debate de hoy:** la propia discrepancia de ROIC de ADBE (22% vs 59-61% según metodología) es un ejemplo en vivo de por qué la mejora de validación cruzada de fundamentales (sección 2) tendría valor inmediato — no es un bug de StockScanner (el dato no proviene de la app), pero ilustra el mismo tipo de riesgo de datos que el comité debe vigilar manualmente hoy.

## 4. Estado de hallazgos previos (confirmación)

Confirmado vía `git log` que ningún fichero de `stockscanner/app/` ha cambiado desde el commit inicial (`48be6c2`); por tanto los hallazgos #1-11 permanecen exactamente en el mismo estado y código descritos en el registro. No se ha revisado línea por línea cada uno hoy (no hay cambio de código que lo justifique); se confirma su vigencia por ausencia de cambios en el repositorio.

**Informes fuente:** [2026-09-19_bug_report.md](2026-09-19_bug_report.md), [2026-09-21_bug_report.md](2026-09-21_bug_report.md)
