# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-21
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2d26b5934428245cd77a6c284f2a8f6a06` (código de `stockscanner/app/` sin cambios desde el commit inicial, confirmado de nuevo con `git log --oneline -- stockscanner/app/`)

## 1. Bugs y discrepancias de datos

Ningún hallazgo nuevo hoy. Se revisó de nuevo el patrón de coerción numérica insegura (`int(x or 0)` / `float(x or 0)`, origen de los hallazgos #24, #31 y #33) en todo `stockscanner/app/` (`grep` sistemático), incluyendo `core_analistas.py` (líneas 27, 35, 83, 93), que no había sido citado explícitamente en #24/#31/#33: ese módulo repite el mismo patrón (`int(mes.get(k) or 0)`) sobre la serie de recomendaciones de analistas, pero consume `recomendaciones.valor` ya validado por `datos_analisis.py` línea 116 (solo se llama si el `Resultado` de `datos_finnhub`/`datos_yfinance` fue `ok`), por lo que **no constituye hoy una ruta de fallo independiente** de #33 — se deja anotado como vigilancia, no como hallazgo nuevo, porque si el fix de #33 cambiara la forma en que se valida `ok` sin revisar también `core_analistas.py`, el mismo patrón quedaría expuesto de nuevo.

Ningún hallazgo previo (#1-34) cambia de estado. Cobertura revisada hoy sin novedades: `core_analistas.py`, `core_cartera.py`, `core_fair_value.py`, `core_fundamentales.py`, `core_paper.py`, `core_referencias.py`, `datos_analisis.py`, `datos_finnhub.py`, `datos_yfinance.py`, `tarea_rastreo.py`, `ui_vistas_cartera.py`, `ui_vistas_favoritos.py`, `ui_vistas_paper.py`, `ui_vistas_rastreador.py`.

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- Ninguna nueva hoy; los hallazgos abiertos #26, #28, #29, #30 y #33 (Alta/Media) siguen siendo las prioridades de cálculo pendientes.

### UI / UX
- Ninguna nueva hoy más allá de lo ya registrado (#10, #11, #15, #16, #32).

### Arquitectura — Funcionalidades y mejoras nuevas de alto impacto

**C. Coerción numérica segura centralizada (prevención de raíz de #24/#31/#33).** `core_ponderar.py` ya define la "regla de oro del proyecto" (`es_dato()`: ni `None`, ni `NaN`, ni infinito) y todo cálculo ponderado la respeta. Pero fuera de esa ruta, al menos 5 puntos del código (`datos_finnhub.py:250`, `datos_yfinance.py:443`, y ahora también `core_analistas.py:27,35,83,93`) parsean series externas con el idioma Python `int(x.get(k) or 0)` — un atajo que asume que un valor "falsy" es sustituible por 0, pero `float('nan')` es *truthy* en Python, así que `NaN or 0` nunca cae a `0` y `int(NaN)` lanza `ValueError` no capturado en el bucle. Es la misma causa raíz que ya produjo tres hallazgos independientes (#24 RSI, #31 ADX, #33 recomendaciones de analistas) en módulos distintos, lo que indica un patrón sistémico, no un error puntual. Propuesta: añadir a `core_ponderar.py` una función `entero_seguro(valor, defecto=0) -> int` (usa `es_dato()` internamente, nunca `or`) y sustituir los 5 puntos localizados por ella; esto no corrige retroactivamente #24/#31/#33 (que tienen su propia causa de UI/semántica) pero cierra la puerta a que el mismo patrón reaparezca en código nuevo o en módulos aún no auditados.

**D. Backtest de las decisiones reales del Comité de Inversión (no solo del veredicto automático del Rastreador).** `core_rastreador.evaluar_senales()` y `resumen_backtest()` ya miden, con datos reales de mercado, el retorno a 3/6/12 meses de cualquier análisis guardado en `analisis_historico` frente al benchmark, agrupado por veredicto automático de la app o por señal de timing — pero hoy solo se alimentan con lo que rastrea `tarea_rastreo.py` (el cron nocturno sobre `INDICES`), nunca con las decisiones reales que toma el Comité en `comite_inversion/cartera/posiciones_actuales.md` (MU, V, AEM, ACN, con fecha y precio de entrada ideal documentados en cada Veredicto de Inversión). El propio comité no tiene hoy ninguna forma de verificar con el motor de backtest ya construido si sus elecciones humanas-IA superan al veredicto automático de la app para los mismos tickers y fechas. Propuesta: una función `filas_comite()` en `core_rastreador.py` que lea las 4 posiciones vigentes (ticker, fecha de sesión, precio de entrada ideal citado en el Veredicto) como filas de `analisis_historico` sintéticas (motor_version="comite"), reutilizando tal cual `evaluar_senales()`/`resumen_backtest()` sin nueva lógica de cálculo — y una pestaña o sección en el Rastreador que muestre ese resumen junto al de los veredictos automáticos, para que el propio Comité pueda auditarse frente a su herramienta.

## 3. Alertas para el debate de inversión

- Pendiente de completar con las candidatas de la Fase 1 de hoy (en curso). Se revisará esta sección antes de cerrar la sesión si alguna candidata depende de un módulo de StockScanner con un hallazgo abierto relevante (en particular #26 para el stop-loss de un posible Plan DCA, #28 para cualquier Fair Value calculado con baja cobertura de métodos, y #33 para candidatas internacionales con poca cobertura de analistas).
