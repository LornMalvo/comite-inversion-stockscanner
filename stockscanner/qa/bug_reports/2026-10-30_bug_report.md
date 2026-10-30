# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-30
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2` (confirmado con `git log --oneline -- stockscanner/app/` → un solo commit; `git diff --stat 48be6c2 HEAD -- stockscanner/app/` → vacío) — **sin cambios de código desde 2026-09-19, novena confirmación consecutiva** (tras las de 2026-09-21 a 2026-10-28)

## 1. Bugs y discrepancias de datos

Ninguno nuevo. La auditoría de hoy revisó de nuevo los 36 hallazgos ya registrados sin encontrar ninguno que merezca reclasificación (no se ha aplicado ningún fix que confirmar, al no haber cambios de código) y releyó con atención los umbrales de `recomendar()` en `core_cartera.py` (`CARTERA_RECO_PERDIDA_VENDER = -20.0` / `REDUCIR = -8.0` / `GANANCIA_PARCIAL = 25.0` / `PROTEGER = 12.0` / `AMPLIAR_MAX = 10.0`, `config_settings.py:563-567`) para descartar una inconsistencia de orden entre umbrales: están correctamente escalonados, no hay bug. Se ejecutó además `core_cartera.libro()` en `python3.13 -I` con una secuencia sintética de compra/compra/venta parcial/venta total (dos tickers, comisiones, FIFO) — resultado (coste medio ponderado 110,10 €, realizado FIFO 347,50 € en la venta parcial y 196,00 € acumulado al cierre) cuadra exactamente con el cálculo manual esperado, sin discrepancias. Se releyeron también `core_rastreador.py` y `core_ponderar.py` buscando el patrón de coerción insegura/redistribución de pesos (mismo tipo que #26-#35): no se encontró ninguno nuevo.

## 2. Mejoras propuestas

### Arquitectura — Propuesta A (profundización de la propuesta del 2026-10-28): `exposicion_prospectiva()` en `core_cartera.py`

Pseudocódigo concreto construido sobre el `exposicion_sector()` real (líneas 276-293) y sobre las tablas reales de `db_supabase.py` (`analisis_historico`, `cartera_operaciones`):

```python
# core_cartera.py
def exposicion_prospectiva(
    posiciones: dict[str, dict],
    sectores: dict[str, str | None],
    candidatas_pendientes: list[dict],   # [{"ticker","sector","veredicto","fecha_analisis","importe_entrada_eur"}]
    dias_vigencia: int = 30,
) -> list[dict]:
    """Como exposicion_sector(), pero sumando el peso HIPOTÉTICO de
    candidatas con veredicto ENTRAR/ACUMULAR de los últimos `dias_vigencia`
    días que aún NO tienen una operación real en cartera_operaciones
    (p.ej. la decisión de la sesión de hoy del Comité, pendiente de
    ejecutarse en el bróker). Usa como proxy el importe de entrada E1
    del plan (core_plan_dca.plan()["entradas"][0]) — no inventa un tamaño."""
    base = exposicion_sector(posiciones, sectores)
    abiertas = {p["ticker"] for p in posiciones.values() if not p["cerrada"]}
    hoy, extra, detalle = date.today(), {}, {}
    for c in candidatas_pendientes:
        if c["ticker"] in abiertas or c.get("veredicto") not in ("ENTRAR", "ACUMULAR"):
            continue
        if (hoy - _fecha(c["fecha_analisis"])).days > dias_vigencia:
            continue
        importe = c.get("importe_entrada_eur")
        if not es_dato(importe):
            continue
        s = sectores.get(c["ticker"]) or c.get("sector") or "Sin sector"
        extra[s] = extra.get(s, 0.0) + float(importe)
        detalle.setdefault(s, []).append(c["ticker"])
    total_actual = sum(d["valor_eur"] for d in base)
    total_prosp = total_actual + sum(extra.values())
    if not total_prosp:
        return base
    acumulado = {d["sector"]: dict(d) for d in base}
    for s, importe in extra.items():
        d = acumulado.setdefault(s, {"sector": s, "valor_eur": 0.0, "tickers": []})
        d["valor_eur_prospectivo"] = d.get("valor_eur_prospectivo", d["valor_eur"]) + importe
        d["candidatas_pendientes"] = detalle[s]
    out = []
    for d in acumulado.values():
        vp = d.get("valor_eur_prospectivo", d["valor_eur"])
        d["peso_pct_prospectivo"] = vp / total_prosp * 100
        d["alerta_prospectiva"] = d["peso_pct_prospectivo"] / 100 >= CARTERA_SECTOR_ALERTA
        out.append(d)
    return sorted(out, key=lambda d: -d.get("valor_eur_prospectivo", d["valor_eur"]))
```

Requiere un helper nuevo en `db_supabase.py` — `listar_candidatas_pendientes(dias=30)` — que lea `analisis_historico` filtrando `veredicto in ("ENTRAR","ACUMULAR")` y excluya tickers ya presentes en `cartera_operaciones` (origen "real"). Expuesto en `ui_vistas_cartera.py` junto a la tabla de `exposicion_sector()` ya existente, resolvería en producto el gap verificado el 2026-10-28 (ninguna posición "pendiente de ejecución" se ve hoy en `exposicion_sector()`).

### Cálculo / arquitectura — Propuesta B (nueva, fundamentada en código real): soporte de divisas realmente internacional

Hoy `CARTERA_DIVISAS_CONVERTIBLES = ("EUR", "USD")` (`config_settings.py:510`) es una tupla fija de solo 2 divisas, usada como gate binario en `datos_yfinance.py` (líneas 149, 217 y 228: `if divisa not in CARTERA_DIVISAS_CONVERTIBLES: return None`). Dado que el CLAUDE.md permite expresamente candidatas de cualquier mercado del mundo, cualquier candidata cotizada en GBP, JPY, HKD, KRW, AUD, BRL, etc. se queda hoy sin ningún importe en EUR en Análisis Individual, Rastreador, Paper Trading, Favoritos, Comparables y Cartera (hallazgo #7/#10 ya registrado; esta es su materialización como propuesta de producto):

```python
# config_settings.py — ampliar la lista y dejar de tratarla como "sí/no"
CARTERA_DIVISAS_CONVERTIBLES = ("EUR", "USD", "GBP", "JPY", "CHF", "AUD",
                                 "CAD", "HKD", "SEK", "NOK", "KRW", "BRL", "MXN")

# datos_yfinance.py — mecanismo de conversión real en vez de solo gate
def convertir_a_eur(importe, divisa):
    if not es_dato(importe) or not divisa:
        return None
    if divisa == CARTERA_DIVISA_BASE:
        return importe
    if divisa not in CARTERA_DIVISAS_CONVERTIBLES:
        return None                      # explícito: nunca se inventa, igual que hoy
    tasa = _tasa_fx_cacheada(divisa)      # yf.Ticker(f"{divisa}EUR=X"), con caché TTL (mismo patrón que datos_cache.py)
    return importe * tasa if es_dato(tasa) else None
```

No requiere tocar el whitelist de sufijos de `datos_indices.normalizar()` (hallazgo #20), porque ese whitelist solo afecta al scraper de constituyentes de índices del cron nocturno, no al Análisis Individual manual (un ticker tecleado como `7203.T` va directo a yfinance sin pasar por `normalizar()`). Si en el futuro se ampliara `config_settings.INDICES` a mercados no occidentales para el rastreo nocturno, el whitelist de `datos_indices.py:39` (`.MC .DE .PA .AS .MI .BR .HE .L .SW .F .IR .LS .VI` — solo Europa) sí tendría que ampliarse con sufijos asiáticos/oceánicos/latam (`.T .HK .KS .AX .SA .SS .SZ .TW .SI .NZ .TO .MX`), tal como ya señala el hallazgo #20.

### UI / UX

En las vistas afectadas por #7/#10 (Análisis Individual, Rastreador, Cartera, Favoritos, Comparables), cuando `convertir_a_eur` devuelve `None` por divisa no soportada, mostrar explícitamente "divisa no soportada (añadir a config)" en vez de "EUR n/d" — hoy ambos casos (sin dato vs. divisa fuera de la lista) se ven idénticos en pantalla, lo que oculta al usuario que el problema es una limitación de configuración y no una ausencia real de dato de mercado.

## 3. Alertas para el debate de inversión

Ningún analista llegó a proponer formalmente ninguna candidata en la Fase 1 de hoy, así que no hay una candidata concreta a la que aplicar estas alertas, pero quedan documentadas para sesiones futuras con candidatas internacionales:

- **#7 / #10 (Media, Abierto):** cualquier candidata cotizada en una divisa distinta de EUR/USD (GBP, JPY, HKD, KRW, AUD, BRL, CHF, etc.) muestra "EUR n/d" en precio, importes y capitalización en toda la app.
- **#20 (Media, Abierto):** el rastreo nocturno por índices solo normaliza sufijos de tickers europeos; no cubre Asia/Oceanía/Latam. El Análisis Individual manual no se ve afectado (no pasa por ese whitelist), solo el universo que el cron rastrea automáticamente.
- **#33 (Alta, Abierto):** el parseo de recomendaciones de analistas puede apagarse por completo (`ValueError` no capturado al convertir `NaN` a entero) si algún mes de la serie trae un hueco — relevante sobre todo para candidatas con poca cobertura de analistas (internacionales o de baja capitalización).

Dos de los tickers investigados hoy por los analistas (0388.HK/HKEX y ADP) fueron rechazados en parte por discrepancias de ROIC/DCF entre proveedores de datos externos a StockScanner — no por un fallo de la app, pero es el mismo patrón de "dato en disputa entre fuentes" que motivó la propuesta de arquitectura de verificación cruzada con Finnhub lanzada el 2026-10-26.
