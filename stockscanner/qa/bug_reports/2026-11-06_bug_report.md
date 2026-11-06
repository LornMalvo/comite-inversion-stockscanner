# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-11-06
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** 1836f41ea577104845c5d25ac7fda48b429d3181 (HEAD)

## 1. Bugs y discrepancias de datos

Ninguno nuevo. Confirmado con `git log 48be6c2..HEAD -- stockscanner/app/` (salida vacía) que el código no ha cambiado desde el commit `48be6c2` — **decimotercera confirmación consecutiva**. Los 36 hallazgos de [registro_hallazgos.md](../registro_hallazgos.md) siguen con el mismo estado que en su última revisión; no se reporta ninguno de ellos de nuevo.

**Precisión adicional sobre el hallazgo #30** (ya Abierto/Alta, no se registra como nuevo): al verificar su alcance con detalle, el problema es aún más completo de lo descrito originalmente. `core_plan_dca.invalidacion()` (líneas 290-312) calcula las condiciones de invalidación de tesis y `datos_analisis.analizar()` las mete en el dict `a["invalidacion"]` (línea 151), pero `fila_historico()` (`datos_analisis.py`, líneas 172-197+) construye la fila para `analisis_historico` a mano y **no incluye la clave `invalidacion`** — el dato ni siquiera llega a persistirse una vez; se usa solo para el texto de la tarjeta de narrativa y se descarta en memoria. Confirmado también por búsqueda en el código que ninguna de las 44 funciones de `db_supabase.py` referencia la tabla `tesis_invalidacion` (que sí existe, con columnas `activa`/`disparada_en` ya pensadas para esto, en `sql_migracion_001.sql`, líneas 87-95). Esto sustenta directamente la propuesta de mejora de esta sesión.

## 2. Mejoras propuestas

### Arquitectura — Vigilancia activa de invalidación de tesis (cierra el hallazgo #30)

**Problema concreto:** CLAUDE.md exige que cada Veredicto de Inversión incluya "stop loss fundamental (condición de negocio que invalida la tesis)". La app ya calcula esas condiciones (`core_plan_dca.invalidacion()`) y ya tiene la tabla SQL lista (`tesis_invalidacion`), pero nada las persiste ni las vuelve a comprobar con datos nuevos. El comité depende hoy de que el Supervisor recuerde releer la tesis de cada posición abierta en cada sesión — exactamente el tipo de control que el cron de rastreo (`tarea_rastreo.py`) ya hace para el screener.

**Módulos afectados:** `db_supabase.py` (nuevas funciones), `ui_vistas_paper.py` (registrar al crear plan), `core_alertas.py` (nueva regla pura), `tarea_rastreo.py` (wiring del cron).

**Pseudocódigo:**

```python
# db_supabase.py — nuevas funciones, mismo patrón que registrar_ejecucion_paper()
def registrar_invalidaciones(ticker: str, plan_id: int | None, condiciones: list[dict]) -> bool:
    """Guarda en tesis_invalidacion una fila por condición de invalidacion()
    recién calculada (llamar justo después de guardar_plan_paper)."""
    cli = _cliente()
    if cli is None:
        return False
    try:
        filas = [{"ticker": ticker, "plan_id": plan_id, "condicion": c["condicion"],
                  "umbral": c["umbral"], "activa": True} for c in condiciones]
        cli.table("tesis_invalidacion").insert(filas).execute()
        return True
    except Exception:
        return False


def invalidaciones_activas(tickers: tuple[str, ...]) -> dict[str, list[dict]]:
    """{ticker: [fila,...]} de condiciones aun activas, para los tickers dados."""
    cli = _cliente()
    if cli is None or not tickers:
        return {}
    try:
        filas = cli.table("tesis_invalidacion").select("*") \
                   .in_("ticker", list(tickers)).eq("activa", True).execute().data or []
    except Exception:
        return {}
    out: dict[str, list[dict]] = {}
    for f in filas:
        out.setdefault(f["ticker"], []).append(f)
    return out


def disparar_invalidacion(inv_id: int) -> bool:
    """Marca una condición como disparada (activa=False) para no repetir el aviso."""
    cli = _cliente()
    if cli is None:
        return False
    try:
        cli.table("tesis_invalidacion").update(
            {"activa": False, "disparada_en": datetime.now(timezone.utc).isoformat()}
        ).eq("id", inv_id).execute()
        return True
    except Exception:
        return False
```

```python
# core_alertas.py — nueva regla pura, mismo contrato que alertas_planes()/alertas_cartera()
def alertas_invalidacion(activas: dict[str, list[dict]], analisis_hoy: dict[str, dict], hoy: date) -> tuple[list[dict], list[int]]:
    """Compara cada condición activa contra el análisis fresco de hoy.
    analisis_hoy: {ticker: fila de core_rastreador.fila()/analisis_historico}
    con al menos calidad, fair_value, precio.
    Devuelve (eventos tipo 'invalidacion', ids de condiciones a marcar como disparadas).
    Prioridad -1, por delante de STOP (prioridad 0)."""
    out, disparadas = [], []
    for ticker, condiciones in activas.items():
        hoy_f = analisis_hoy.get(ticker)
        if not hoy_f:
            continue
        for c in condiciones:
            cond, umbral = c["condicion"], c["umbral"]
            rota = False
            if cond == "calidad_degradada" and es_dato(hoy_f.get("calidad")):
                rota = hoy_f["calidad"] < umbral["calidad_min"]
            elif cond == "fair_value_caida" and es_dato(hoy_f.get("fair_value")):
                rota = hoy_f["fair_value"] < umbral["fair_value_min"]
            elif cond == "ruptura_soporte" and es_dato(hoy_f.get("precio")):
                rota = hoy_f["precio"] < umbral["precio"]
            elif cond == "earnings_debiles":
                # requiere datos_finnhub.obtener_earnings, resuelto antes de llamar
                # a esta función pura y pasado ya evaluado en umbral["disparada"]
                rota = umbral.get("disparada", False)
            if rota:
                out.append({"tipo": "invalidacion", "ticker": ticker,
                            "clave": f"inv:{ticker}:{cond}:{hoy.isoformat()}",
                            "prioridad": -1,
                            "texto": f"<b>{ticker}</b> ⚠️ TESIS INVALIDADA ({cond}): {c['texto']}"})
                disparadas.append(c["id"])
    return out, disparadas
```

```python
# tarea_rastreo.py main() — tras calcular filas_hoy (ya existe en el cron)
tickers_vigilados = tuple(sorted({f["ticker"] for f in filas_hoy}))
activas = db_supabase.invalidaciones_activas(tickers_vigilados)
if activas:
    analisis_hoy_por_ticker = {f["ticker"]: f for f in filas_hoy}
    eventos_inv, disparadas = core_alertas.alertas_invalidacion(activas, analisis_hoy_por_ticker, hoy)
    eventos += eventos_inv          # se suman al lote de alertas_screener ya existente
    for inv_id in disparadas:
        db_supabase.disparar_invalidacion(inv_id)

# ui_vistas_paper.py — justo después de un guardar_plan_paper(fila) exitoso
plan_id = db_supabase.guardar_plan_paper(fila)
if plan_id and plan_id > 0:
    condiciones = core_plan_dca.invalidacion(calidad, fair_value, plan, earnings.valor)
    db_supabase.registrar_invalidaciones(ticker, plan_id, condiciones)
```

Añadir `-1: "\U0001f480 TESIS INVALIDADA"` al diccionario `titulos` de `core_alertas.componer()` para que el bloque aparezca primero, por delante de STOP (prioridad 0).

**Justificación:** convierte en funcional una pieza del esquema de datos que ya existe pero está completamente muerta (`tesis_invalidacion` nunca se lee ni se escribe desde Python), cierra directamente el hallazgo #30, y automatiza el único control de riesgo que el propio CLAUDE.md exige por escrito en cada Veredicto de Inversión (stop fundamental) pero que hoy depende de que un humano lo recuerde sesión a sesión. Reutiliza el 100% de la lógica de condiciones que `core_plan_dca.invalidacion()` ya calcula — no se duplica ninguna regla de negocio, solo se persiste y se vuelve a comprobar con datos frescos.

## 3. Alertas para el debate de inversión

Ninguna candidata de esta sesión llegó a usar StockScanner (ninguna de las 16 investigadas en Fase 1 fue propuesta formalmente a Fase 3), así que los hallazgos abiertos de severidad alta (#26, #28, #30) no aplican a ninguna candidata concreta de hoy. Siguen vigentes como riesgo para la próxima candidata que sí se proponga y, en particular, para las cuatro posiciones ya en cartera (MU, V, AEM, ACN), que es justamente lo que motiva la propuesta de este informe.
