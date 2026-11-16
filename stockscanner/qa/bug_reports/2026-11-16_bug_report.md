# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-11-16
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2` (confirmado sin cambios vía `git log --oneline` y `git diff 48be6c2 -- stockscanner/app/` — diff vacío; **decimoctava confirmación consecutiva** desde 2026-09-19)

## 1. Bugs y discrepancias de datos

Los 38 hallazgos del registro ([registro_hallazgos.md](../registro_hallazgos.md)) mantienen el mismo estado que en la última revisión (2026-11-13); no se reportan de nuevo. Se encontró **un hallazgo genuinamente nuevo (#39)**, por cruce de código entre el camino de escritura y el de lectura del Screener — no por cambio de código, sino por una comprobación que ninguna auditoría anterior había hecho explícitamente: comparar qué escribe `fila_historico()` en `analisis_historico` contra qué columnas vuelve a leer `listar_screener()` de esa misma tabla.

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 39 | Media (bloqueante para la propuesta de arquitectura de esta sesión) | `db_supabase.py` (`listar_screener()`, líneas 148-173) / `core_rastreador.py` (`fila_desde_historico()`, líneas 92-114) | `datos_analisis.fila_historico()` (línea 196) calcula y persiste correctamente `cobertura: {"calidad":…, "fair_value":…, "timing":…}` en la columna `cobertura` (jsonb) de `analisis_historico` — la columna existe desde `sql_migracion_001.sql` línea 32. Pero `listar_screener()` selecciona explícitamente una lista de columnas (línea 156-157) que **no incluye `cobertura`**, así que cada fila que llega al modo Screener del Rastreador la pierde. `fila_desde_historico()`, que construye el dict de trabajo a partir de esa fila, en consecuencia no tiene ninguna clave de cobertura (confirmado leyendo las dos funciones: su `dict` de salida, líneas 101-112, no contiene `calidad_cobertura` ni ninguna otra clave de cobertura). Nótese además que incluso `core_rastreador.fila()` (camino "en vivo", línea 49) solo expone `calidad_cobertura` — ninguna cobertura de Fair Value ni de Timing llega nunca más allá de los motores `core_fair_value.calcular()`/`core_timing.calcular()` (que sí la calculan, campo `"cobertura"`, ver #38) | Lectura cruzada de `datos_analisis.py:196`, `db_supabase.py:156-157`, `core_rastreador.py:34-89` y `92-114`, y `sql_migracion_001.sql:32`. Confirmado también que hoy esto no provoca una excepción visible: la vista de comparación lado a lado en modo Screener (`ui_vistas_rastreador.py`, `_columna_comparacion()`, líneas 263-266) solo se alimenta de filas reanalizadas en vivo (botón "Comparar en vivo", líneas 233-241), nunca directamente de `fila_desde_historico()`; y la tabla del modo Screener (`_tabla()`, líneas 150-160) nunca pinta columnas de cobertura. Es decir: hoy es una pérdida de dato silenciosa sin síntoma visible, no un crash | Sin consecuencia visible hoy (ningún usuario ve un valor erróneo). Pero es exactamente el dato que la propuesta de arquitectura de esta sesión (`core_screener.candidatas_automaticas()`, más abajo) necesitaría para aplicar el gate de `cobertura_minima` que motivó el hallazgo #38: si se construye sobre `listar_screener()`/`fila_desde_historico()` tal cual existen hoy, la preselección automática de candidatas **no podría filtrar por cobertura en absoluto** y heredaría el problema de #38 sin el control manual que hoy mitiga su efecto (un analista viendo la ficha completa) |

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- Ninguna nueva. Las propuestas previas de este bloque (P/FFO TTM consistente con #2, fill de Paper Trading en huecos alcistas con #4) siguen abiertas en el registro y sin cambios de código que las afecten.

### UI / UX
- Ninguna nueva hoy más allá de la ya propuesta el 2026-11-13 (vista "Candidatas sugeridas" en Streamlit si se implementa el screener automático), que sigue vigente y se detalla abajo junto al resto de la arquitectura.

### Arquitectura — profundización de `core_screener.candidatas_automaticas()` (prioridad máxima, continúa el 2026-11-13)

El boceto del 2026-11-13 planteaba la idea correcta (preseleccionar candidatas reutilizando las tres notas ya calculadas por la app, en vez de que el analista adivine tickers a mano) pero su pseudocódigo asumía un diseño más caro de lo necesario: recibía `resultados_analisis` como si hubiera que ejecutar `datos_analisis.analizar()` sobre un universo completo en el momento de la sesión. Revisando `tarea_rastreo.py` y `db_supabase.listar_screener()` se confirma que **eso ya existe y ya se ejecuta cada noche sin coste añadido**: el cron (`tarea_rastreo.py`, L-V 22:30 UTC) recorre todos los índices activos de `config_settings.INDICES` más el índice virtual "Comparables" y persiste Calidad/Fair Value/Timing/Veredicto/Banda/Sector de cada ticker en `analisis_historico` con `origen="cron"` — exactamente el universo y las notas que la propuesta pedía calcular aparte. El diseño correcto es una capa fina sobre lo que el modo "Screener" del Rastreador ya consulta (`db_supabase.listar_screener()` + `core_rastreador.fila_desde_historico()` + `core_rastreador.filtrar()`/`ordenar()`), no un pipeline nuevo. Esto depende, eso sí, de corregir primero el hallazgo #39 (sin cobertura no hay gate fiable) y de la constante `COBERTURA_MINIMA_FIABLE` propuesta el 2026-11-11 (#38).

**Paso 0 — requisitos previos (fijan #39 y preparan #38):**

```python
# db_supabase.py — ampliar la selección de columnas de listar_screener()
def listar_screener(desde: str, origen: str = "cron") -> list[dict]:
    columnas = ("ticker,fecha_analisis,motor_version,precio,divisa,nombre,sector,calidad,"
                "fair_value,upside_pct,timing,timing_bruto,senal_timing,veredicto,perfil,"
                "banda,plan,cobertura")   # <- única línea nueva: estaba persistida y no se leía (#39)
    ...  # resto igual


# core_rastreador.py — fila() y fila_desde_historico() exponen las 3 coberturas,
# no solo la de calidad, para que filtrar() y candidatas_automaticas() puedan usarlas
def fila(a: dict) -> dict:
    ...
    f["calidad_cobertura"] = cal.get("cobertura")
    f["fv_cobertura"] = fv.get("cobertura")          # <- nuevo
    f["timing_cobertura"] = t.get("cobertura")        # <- nuevo
    ...

def fila_desde_historico(h: dict) -> dict:
    cob = h.get("cobertura") or {}
    ...
    f["calidad_cobertura"] = cob.get("calidad")
    f["fv_cobertura"] = cob.get("fair_value")
    f["timing_cobertura"] = cob.get("timing")
    ...
```

**Paso 1 — módulo nuevo `core_screener.py`, función pura sin llamadas a red ni a Supabase (recibe las filas ya leídas, igual que el resto de `core_*.py`):**

```python
"""Preselección cuantitativa de candidatas sobre lo que el cron nocturno
(tarea_rastreo.py) ya ha analizado y persistido en `analisis_historico`.
No decide por sí sola ni sustituye el análisis cualitativo de la Fase 1:
reduce el universo de "adivinar un ticker y comprobar si cumple la triple
condición" a "pedir los que ya la cumplen según el motor", para que el
analista dedique su tiempo a validar el foso, los riesgos y la narrativa
de un puñado de nombres en vez de descartar quince al día a ciegas."""

from __future__ import annotations

from core_ponderar import es_dato


def candidatas_automaticas(
    filas: list[dict],               # core_rastreador.fila_desde_historico() de db_supabase.listar_screener()
    ya_tratados: set[str],           # tickers de empresas_analizadas.md (o db_supabase.ya_analizada(), propuesta 2026-11-04)
    minimo_calidad: float = 70.0,     # umbral de "Máxima Calidad" (CLAUDE.md, pilar 1)
    minimo_descuento_pct: float = 15.0,   # umbral de "Infravaloración" (CLAUDE.md, pilar 2) == upside_pct
    minimo_timing: float = 60.0,      # umbral de "Timing Óptimo" (CLAUDE.md, pilar 3)
    cobertura_minima: float = COBERTURA_MINIMA_FIABLE,  # constante única propuesta el 2026-11-11 (#38)
    max_candidatas: int = 15,
) -> list[dict]:
    """De las filas ya calculadas por el cron, las que cumplen la triple
    condición SIMULTÁNEAMENTE y con cobertura de dato suficiente en cada
    uno de los tres pilares, excluyendo lo ya tratado (regla de
    no-repetición de CLAUDE.md). Nunca se llama con filas sin `fila_desde_
    historico()` aplicado antes — asume el shape de ese dict."""
    candidatas = []
    for f in filas:
        ticker = f["ticker"]
        if ticker in ya_tratados:
            continue   # regla de no-repetición; un cambio sustancial documentado se evalúa a mano, no aquí
        calidad, upside, timing = f.get("calidad"), f.get("upside_pct"), f.get("timing")
        cob_cal, cob_fv, cob_t = f.get("calidad_cobertura"), f.get("fv_cobertura"), f.get("timing_cobertura")
        # sin dato en cualquiera de los tres pilares, o cobertura insuficiente
        # en cualquiera de ellos, la fila no es fiable para preseleccionar
        if not (es_dato(calidad) and es_dato(upside) and es_dato(timing)):
            continue
        if not (es_dato(cob_cal) and es_dato(cob_fv) and es_dato(cob_t)):
            continue   # #39: si el dato de cobertura no llegó, no se asume fiable por defecto
        if min(cob_cal, cob_fv, cob_t) < cobertura_minima:
            continue   # #38: nota "completa" con poco dato real detrás no basta
        if calidad < minimo_calidad or upside < minimo_descuento_pct or timing < minimo_timing:
            continue
        candidatas.append({
            "ticker": ticker, "nombre": f.get("nombre"), "sector": f.get("sector"),
            "calidad": calidad, "upside_pct": upside, "timing": timing,
            "cobertura_minima_de_los_tres": min(cob_cal, cob_fv, cob_t),
            "score": (calidad + timing) / 2 + upside / 2,  # pondera también el margen de seguridad, no solo las notas 0-100
            "veredicto": f.get("veredicto"), "banda": f.get("banda"),
            "fecha_analisis": f.get("fecha"),
        })
    candidatas.sort(key=lambda c: c["score"], reverse=True)
    return candidatas[:max_candidatas]
```

**Paso 2 — integración en la sesión (orquestador, p. ej. `ui_vistas_rastreador.py` o un comando que el Supervisor ejecute al abrir la Fase 1):**

```python
desde = (hoy() - timedelta(days=SCREENER_MAX_DIAS)).isoformat()
filas = [core_rastreador.fila_desde_historico(h) for h in db_supabase.listar_screener(desde)]
ya_tratados = cargar_tickers_de_empresas_analizadas()  # hoy: parsear el .md; mañana: db_supabase.ya_analizada() (propuesta 2026-11-04)
preseleccion = core_screener.candidatas_automaticas(filas, ya_tratados)
# preseleccion es el PUNTO DE PARTIDA de la Fase 1, no el veredicto:
# cada analista sigue haciendo su propio análisis integral (foso, riesgos,
# fuentes) sobre 1-2 de estas, en vez de sobre un ticker elegido a ciegas.
```

Coste: **cero** llamadas a API adicionales — toda la información ya existe en `analisis_historico` gracias al cron que ya corre cada noche; es una consulta Supabase + un filtro en memoria, igual de barato que abrir hoy el modo Screener del Rastreador. El único trabajo de implementación real es el Paso 0 (exponer la cobertura que ya se calcula) y escribir `core_screener.py`.

**UI/UX asociada (continúa la propuesta del 2026-10-19):** vista nueva "Candidatas sugeridas" (o una pestaña dentro del Rastreador en modo Screener) que renderice `candidatas_automaticas()` con el mismo semáforo de `ui_metricas.py`, mostrando explícitamente la cobertura mínima de cada fila (para que el analista vea de un vistazo si la preselección es sobre dato sólido o sobre el mínimo aceptado) y un botón "Analizar" por fila que lleve a Análisis Individual, igual que ya hace `_columna_comparacion()` en el Rastreador.

## 3. Alertas para el debate de inversión

- **#38 (Alta, abierto):** la nota final de Timing y de Fair Value no exige cobertura mínima de datos (a diferencia de Calidad). Si alguna candidata de hoy muestra una nota de Timing o de Fair Value "completa" pero fue calculada con muy pocos componentes de dato disponibles, esa nota puede ser más ruido que señal — pedir a StockScanner/el analista el desglose de cobertura antes de ponderarla en el debate.
- **#39 (nuevo hoy, Media):** si algún analista consulta hoy el modo "Screener" del Rastreador para mirar de reojo qué índices tiene el cron ya analizados, tenga en cuenta que esa vista **no muestra ningún dato de cobertura** (ni lo tiene disponible internamente) — no es indicio de que el dato sea sólido, solo de que no se le pregunta.
- **#20 (Alta, abierto):** si alguna candidata de hoy cotiza en un mercado asiático/oceánico/latam con sufijo de ticker no occidental (`.T`, `.HK`, `.KS`, `.AX`, `.SA`, etc.), verificar a mano que el ticker usado en el análisis no esté corrompido por el bug de doble sufijo de `datos_indices.normalizar()`.
- **#26 (Alta, abierto):** si el plan de entrada de alguna candidata usa el stop "con ATR" y N3 cae más de ~25% bajo el coste medio, el riesgo real del stop puede superar el 25% anunciado — comprobar el riesgo_pct real antes de fijarlo como stop loss técnico en el Veredicto de Inversión.
- **#28 (Alta, abierto):** con solo dos valores disponibles en Fair Value (un método intrínseco + consenso de analistas), la banda de cordura nunca puede llegar a "excluir" un método por disparatado que sea, solo "recortarlo" — si el Fair Value de alguna candidata de hoy se apoya en poca cobertura de métodos, revisar manualmente que ninguno de los dos valores sea un outlier no detectado.
- **#37 (Alta, abierto):** si alguna candidata tiene patrimonio neto negativo, el ratio Deuda/Patrimonio resultante (negativo) se puntúa hoy como la mejor nota posible de ese tramo en Calidad en vez de señalar insolvencia — descartar manualmente cualquier candidata en esa situación, como ya se hizo con RNG/VIK el 2026-11-04.
