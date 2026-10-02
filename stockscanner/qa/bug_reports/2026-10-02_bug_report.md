# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-02
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2d26b5934428245cd77a6c284f2a8f6a06` (2026-09-19) — confirmado con `git log --oneline -- stockscanner/app/`: sigue siendo el único commit que toca ese directorio, cero cambios desde la auditoría anterior (2026-09-30).

## 0. Nota de metodología

Siguiendo la recomendación del informe del 2026-09-30, esta auditoría se centró en **ejecución real** de los motores puros (no solo relectura de código). Se instalaron `numpy`/`pandas` (no estaban disponibles en el entorno) y se ejecutaron con datos sintéticos `core_indicadores`, `core_confluencia`, `core_timing`, `core_plan_dca`, `core_calidad`, `core_fair_value`, `core_cartera` y `core_analistas`, incluyendo un pipeline de integración completo (histórico OHLCV sintético → indicadores → confluencia → fair value → calidad → timing → plan) que corrió sin excepciones y con valores razonables. Los 25 hallazgos previos del registro siguen abiertos sin cambios de código; no se repite ninguno.

De esa ejecución salió **un bug nuevo genuino y reproducible** en `core_plan_dca.plan()`, descrito abajo con evidencia de ejecución y verificado por el Supervisor leyendo directamente `stockscanner/app/core_plan_dca.py:141-146`.

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 26 (nuevo) | **Alta** | `core_plan_dca.py` (`plan()`, líneas 141-146) | El stop-loss "con ATR" aplica el tope de caída máxima (`DCA_STOP_CAIDA_MAX`, 25 %) y **a continuación** lo vuelve a recortar con `stop = min(stop, n3 * 0.995)` (para que el stop nunca quede por encima de N3). Cuando N3 cae por debajo del nivel que marca ese tope de 25 %, este segundo `min()` empuja el stop de vuelta por debajo del suelo que el `max()` anterior acababa de fijar, **rompiendo la garantía de pérdida máxima que el propio `motivo_stop` anuncia al usuario**. Esto ocurre siempre que E3 (sintético o real) quede más de ~25 % por debajo del coste medio ponderado — algo normal en valores con poca estructura de soporte por debajo del precio (tendencia fuerte, máximos nuevos, zona de entrada sin confluencia técnica cercana). | Ejecutado directamente: `core_plan_dca.plan({"zonas": []}, {"atr": 10.0, ...}, precio=100.0, fair_value=None)` → entradas `[80, 60, 40]` (ladder 100 % sintético, normal sin zonas en rango), `coste_medio=63.0`, `stop=39.8` → **`riesgo_pct=36.8 %`**, mientras `motivo_stop` sigue diciendo *"2.5 x ATR bajo el coste medio, con margen bajo E3 y caída máxima 25 %"*. Segunda ejecución con un caso más realista (2 zonas técnicas reales para E1/E2, E3 sintético): `riesgo_pct=25.96 %` — ya por encima del 25 % anunciado con solo una desviación moderada. `ui_bloque_plan.py:81-82` muestra `riesgo_pct` y `motivo_stop` uno junto a otro, así que el usuario ve el número real (37 % o 26 %) justo al lado de un texto que promete un tope del 25 % — contradicción directa en pantalla. Verificado por el Supervisor leyendo `core_plan_dca.py:141-146`: la línea 144 fija el suelo (`max(..., coste_medio * (1 - DCA_STOP_CAIDA_MAX))`) y la línea 145 lo puede deshacer (`min(..., n3 * 0.995)`). | El plan de entrada/stop es el output central de cada sesión de inversión (CLAUDE.md exige "stop loss técnico" en el veredicto). Un stop que arriesga 37 % cuando se anuncia 25 % infravalora el riesgo real de cualquier posición nueva que se abra con este plan — es el tipo de fallo de cálculo financiero que el proyecto marca como máxima prioridad a evitar. No es el mismo bug que el hallazgo #1 del registro (ese es sobre la rama *sin* ATR, que directamente no aplica ningún tope frente a N3; este es sobre la rama *con* ATR, que sí aplica el tope pero en el orden equivocado y acaba violándolo). |

No se han encontrado más bugs nuevos genuinos: la ejecución de `core_indicadores` (RSI/ADX/ATR/MACD en racha alcista pura, racha bajista pura y precio totalmente plano), `core_confluencia`, `core_calidad` y `core_fair_value` con datos sintéticos y con el pipeline de integración completo no reveló comportamiento distinto del ya documentado en el registro (p. ej. RSI → NaN en racha alcista/plana confirma el hallazgo #24 ya abierto, no se cuenta como nuevo).

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- **Fix directo para el bug nuevo #26 de esta sesión:** en `core_plan_dca.py:141-146`, aplicar el tope de `DCA_STOP_CAIDA_MAX` **después** del tope de N3, no antes, o acotar explícitamente el resultado final al rango `[coste_medio * (1 - DCA_STOP_CAIDA_MAX), n3 * 0.995]` y, si ese rango es vacío (floor > techo), priorizar el tope de riesgo máximo y relajar el margen bajo N3 en vez de al revés — hoy la prioridad implícita es "nunca por encima de N3" por encima de "nunca pierdas más del 25 %", que es la prioridad que el propio texto y el resto del diseño (`DCA_STOP_CAIDA_MAX` como "restricción es 'no arriesgo más de un X % de lo invertido'", según el docstring del módulo) dice que debería tener. Pseudocódigo:
  ```python
  techo_n3 = n3 * 0.995
  floor_riesgo = coste_medio * (1 - DCA_STOP_CAIDA_MAX)
  stop_atr = coste_medio - DCA_STOP_ATR_MULT * atr
  stop_margen_n3 = n3 - DCA_STOP_MARGEN_ATR_BAJO_N3 * atr
  stop = min(stop_atr, stop_margen_n3, techo_n3)
  if stop < floor_riesgo:
      # el margen bajo N3 exige arriesgar más del tope: se avisa, no se oculta
      stop = floor_riesgo
      motivo_stop += " (ajustado: el margen bajo E3 habría superado el tope de caída máxima)"
  ```

### UI / UX
- **Resaltar cuando `riesgo_pct` supera el `DCA_STOP_CAIDA_MAX` nominal.** Mientras no se corrija el cálculo, `ui_bloque_plan.py` podría al menos comparar `riesgo_pct` contra `DCA_STOP_CAIDA_MAX*100` y marcar visualmente la discrepancia (hoy el texto y el número conviven sin que nada avise de que se contradicen).

### Arquitectura
- (Sin novedad sobre lo ya registrado: sigue en pie la recomendación de smoke tests de los motores puros, hallazgo #9 + recomendación del informe anterior. El bug de esta sesión es exactamente el tipo de caso límite que un test de 3-4 fixtures por motor habría atrapado antes de producción.)

### Funcionalidades nuevas de alto impacto
- **Módulo de backtesting.** Los docstrings de `core_fair_value.py`, `core_timing.py` y `core_indicadores.py` afirman explícitamente, en tres módulos distintos, que el diseño está pensado para backtesting ("función pura y reconstruible... recibe fundamentales, estados, histórico y precio"; "reconstruible para el backtesting a cualquier fecha"). Sin embargo, no existe ningún archivo `core_backtest.py` ni vista de backtesting entre los 41 `.py` de `stockscanner/app/`. Es la funcionalidad de mayor valor que el propio código ya anuncia como objetivo de diseño pero que nunca se construyó: recorrer el histórico fecha a fecha, recalcular Calidad/Fair Value/Timing/Plan con los datos disponibles hasta esa fecha (sin look-ahead) y medir qué habría pasado con los veredictos y stops generados — validaría o refutaría de forma cuantitativa toda la metodología de valoración del comité, no solo de la app.
- **Calculadora de tamaño de posición por presupuesto de riesgo.** Hoy `core_plan_dca.plan()` calcula `riesgo_pct` (% de pérdida hasta el stop) pero nada traduce eso a cuántas acciones comprar dado un presupuesto de riesgo de cartera (p. ej. "no arriesgar más del 1 % del valor total de la cartera en una sola posición"). `core_cartera.resumen()` ya expone el valor total de la cartera (`valor_eur`); conectar ambos (`tamaño_posición = (valor_cartera * riesgo_maximo_cartera_pct) / riesgo_pct_del_plan`) convertiría el plan DCA en una recomendación accionable de tamaño, no solo de niveles de precio.
- **Aviso de correlación al evaluar una candidata nueva.** `core_cartera.correlacion()` ya calcula pares de posiciones con correlación alta, pero solo se usa dentro de la vista de Gestión de Cartera sobre posiciones ya abiertas — no se consulta nunca al analizar o decidir sobre una candidata nueva. Antes de abrir una posición, cruzar sus retornos diarios contra los de la cartera actual con la misma función y avisar si la nueva candidata está altamente correlacionada con una posición existente evitaría concentrar riesgo de facto (aunque los sectores declarados sean distintos) sin esperar a que aparezca en el panel de cartera el día después de comprar.

## 3. Alertas para el debate de inversión

> Cualquier problema de datos que pueda afectar la valoración de una candidata de esta sesión debe listarse aquí para que los analistas lo integren en la Fase 3.

- **El bug nuevo #26 es universal y directamente relevante hoy:** si el Veredicto de Inversión de esta sesión abre una posición nueva (candidatas de hoy: ACN, MUV2.DE), el Supervisor debe recalcular manualmente el riesgo real (`(coste_medio - stop) / coste_medio`) antes de fijar el stop loss técnico del plan de acción, en vez de confiar en el `riesgo_pct`/`motivo_stop` tal cual los mostraría la app — el bug puede hacer que el riesgo real supere el 25% anunciado, especialmente en valores con alta volatilidad reciente y sin estructura de soporte técnico claro por debajo del precio (el propio patrón que describen ACN tras su hueco alcista del 1-oct-2026).
- Hallazgos ya abiertos que siguen aplicando igual que en sesiones anteriores: #12 (cron no corre en producción — ninguna alerta automática de stop para una posición nueva de hoy), #24 (RSI puede salir "sin dato" en racha alcista pura — vigilar si ACN, que acaba de tener un salto de precio muy fuerte, muestra esta lectura ausente en vez de sobrecompra).
- Ninguna de las dos candidatas de hoy (ACN en NYSE/USD, MUV2.DE en XETRA/EUR) depende de los sufijos bursátiles no cubiertos del hallazgo #20 (`.T`, `.HK`, `.KS`, `.AX`, `.SA`); MUV2.DE sí usaría la conversión EUR/USD del hallazgo #7/#10, que sí está soportada.

---

## Resumen para Fase 2 de la sesión

Cero cambios de código desde 2026-09-19 (commit único `48be6c2`, confirmado). Siguiendo la recomendación de la auditoría anterior, esta vez se instalaron las dependencias (`numpy`/`pandas`) y se ejecutaron realmente los motores puros con datos sintéticos y un pipeline de integración completo, en vez de solo releer código. Esa ejecución encontró un **bug nuevo de severidad alta** (#26): en `core_plan_dca.plan()` (líneas 141-146), el stop-loss "con ATR" puede arriesgar bien por encima del tope del 25 % que su propio texto (`motivo_stop`) anuncia al usuario, porque el recorte final "nunca por encima de N3" se aplica después del tope de caída máxima y puede deshacerlo — reproducido con `riesgo_pct` de 36,8 % y 26,0 % en dos escenarios distintos, y verificado por el Supervisor leyendo el código directamente. Se proponen tres funcionalidades nuevas de alto impacto: un módulo de backtesting (que el propio código lleva anunciando en sus docstrings desde el principio sin haberse construido nunca), una calculadora de tamaño de posición basada en presupuesto de riesgo de cartera, y un aviso de correlación al evaluar candidatas nuevas (reutilizando `core_cartera.correlacion()` antes de comprar, no solo después).
