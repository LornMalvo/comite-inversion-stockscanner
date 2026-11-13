# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-11-13
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2` (confirmado sin cambios vía `git log --oneline -- stockscanner/app/`; **decimoséptima confirmación consecutiva** desde 2026-09-19)

## 1. Bugs y discrepancias de datos

Ninguno nuevo. Los 38 hallazgos del registro ([registro_hallazgos.md](../registro_hallazgos.md)) mantienen el mismo estado que en la última revisión (2026-11-11); no se reportan de nuevo.

## 2. Mejoras propuestas

### Arquitectura — `core_screener.candidatas_automaticas()` (nueva, alto impacto)

**Motivación verificada en vivo durante la sesión de hoy:** los tres analistas investigaron 16 tickers nuevos (NVR, BRO, CTAS, ROL, SHW, GGG, APD, C, COP, PFE, GM, VZ, ORCL, GE, UBER, NFLX) de forma puramente manual — elegidos por intuición/memoria del analista, uno a uno, con WebSearch genérico — y los 16 fallaron por el mismo patrón estructural de siempre: ROIC/DCF irreconciliable entre proveedores o ausencia de descuento real. Esta es la vigésimo quinta sesión del comité y la selección manual de candidatas sigue siendo un cuello de botella puramente artesanal: StockScanner ya calcula, para cualquier ticker que se le dé, una nota de Calidad, una de Fair Value y una de Timing (los tres pilares exactos de la triple condición de CLAUDE.md) a través de `datos_analisis.analizar()` — pero hoy nada en la app recorre un universo de tickers para proponer candidatas; cada analista debe adivinar qué ticker probar antes de poder usar la app.

**Propuesta:** un módulo nuevo `core_screener.py` con una función pura que reutiliza el pipeline de análisis ya existente sobre una lista de tickers, en vez de depender de que el analista adivine cada candidata a mano:

```python
def candidatas_automaticas(
    universo: list[str],
    resultados_analisis: dict[str, dict],  # {ticker: resultado de datos_analisis.analizar()}
    ya_tratados: set[str],                  # tickers de empresas_analizadas.md (ver #6/#10 propuesto 2026-11-04)
    minimo_calidad: float = 70.0,
    minimo_descuento_pct: float = 15.0,
    minimo_timing: float = 60.0,
    cobertura_minima: float = COBERTURA_MINIMA_FIABLE,  # reutiliza la constante propuesta el 2026-11-11 (#38)
) -> list[dict]:
    """Filtra `universo` a los tickers cuyo resultado ya calculado por
    datos_analisis.analizar() supera los tres umbrales simultáneamente
    (triple condición), con cobertura de datos suficiente en cada nota,
    y que no estén ya en el registro de no-repetición. Devuelve
    ordenado por score compuesto (media de las tres notas), con el
    desglose de calidad/valor/timing de cada uno para que el analista
    valide manualmente antes de proponerlo — no decide por sí sola,
    solo preselecciona candidatas a investigar con profundidad.
    """
    candidatas = []
    for ticker in universo:
        if ticker in ya_tratados:
            continue
        r = resultados_analisis.get(ticker)
        if r is None:
            continue
        calidad, fv, timing = r["calidad"], r["fair_value"], r["timing"]
        if (calidad["cobertura"] < cobertura_minima
                or fv["cobertura"] < cobertura_minima
                or timing["cobertura"] < cobertura_minima):
            continue  # nota no fiable, no preseleccionar con dato insuficiente
        descuento_pct = fv["descuento_pct"]
        if (calidad["nota"] >= minimo_calidad
                and descuento_pct >= minimo_descuento_pct
                and timing["nota"] >= minimo_timing):
            candidatas.append({
                "ticker": ticker,
                "score": (calidad["nota"] + fv["nota"] + timing["nota"]) / 3,
                "calidad": calidad["nota"], "descuento_pct": descuento_pct,
                "timing": timing["nota"],
            })
    return sorted(candidatas, key=lambda c: c["score"], reverse=True)
```

El `universo` inicial podría ser una lista configurable en `config_settings.py` (p. ej. S&P 500 + Stoxx 600 + Nikkei 225 + un puñado de mercados emergentes ya visitados por el comité), ejecutada una vez por el cron nocturno (`tarea_rastreo.py`, que ya recorre índices) para no disparar cientos de llamadas a APIs en el momento de la sesión. Esto no sustituye el juicio cualitativo del analista (foso, value trap, riesgos) — solo reemplaza la fase de "adivinar qué ticker probar" por una preselección cuantitativa reproducible, reduciendo drásticamente el número de tickers investigados en vano cada sesión (16 de 16 fallidos hoy, patrón repetido en 24 de las 25 sesiones).

### UI/UX
- Si se implementa el screener anterior, añadir una vista nueva en Streamlit ("Candidatas sugeridas") que liste el resultado de `candidatas_automaticas()` con el mismo semáforo de colores que ya usa `ui_metricas.py`, para que el comité la consulte al inicio de la Fase 1 en vez de partir de una lista en blanco.

## 3. Alertas para el debate de inversión

- Ninguna candidata llegó a proponerse formalmente hoy, por lo que no hay alerta de datos que aplicar a un análisis concreto de esta sesión.
