# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-11-02
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2d26b5934428245cd77a6c284f2a8f6a06` (verificado con `git log -1 --format=%H -- stockscanner/app/`)

## Confirmación de cambios de código

**Cero cambios de código desde 48be6c2 — décima confirmación consecutiva** (2026-10-14, 16, 19, 21, 23, 26, 28, 30, y hoy 2026-11-02 — más la cobertura original del 2026-09-30). El hash coincide exactamente con el citado en todas las auditorías previas. En consecuencia, siguiendo la regla de no-repetición de CLAUDE.md, no se ha vuelto a hacer caza de bugs línea a línea sobre cálculos ya auditados (9 pasadas exhaustivas previas cubren el 100% de `stockscanner/app/`).

## 1. Bugs y discrepancias de datos

Ninguno nuevo. Los 36 hallazgos del registro ([registro_hallazgos.md](../registro_hallazgos.md)) permanecen "Abierto" con el mismo estado de la última revisión; no se reporta ninguno de ellos de nuevo. Siguen vigentes como riesgo de mayor severidad para cualquier candidata futura: #26 (stop ATR puede superar el 25% de riesgo anunciado), #28 (banda de cordura del Fair Value nunca excluye un método disparatado con solo 2 valores), #30 (stop-loss fundamental se calcula una vez pero nunca se re-comprueba), #33 (parseo de recomendaciones de analistas puede apagarse entero por un NaN).

## 2. Mejoras propuestas

### Arquitectura / Funcionalidad nueva — Alerta de noticias materiales para posiciones abiertas

**Justificación.** StockScanner ya descarga titulares (`datos_finnhub.obtener_noticias()`, con respaldo en `datos_yfinance.obtener_noticias_yf()`), pero **solo se muestran una vez**, dentro de la vista de Análisis Individual (`ui_vistas_analisis._noticias()`), en el momento puntual en que alguien abre esa ficha. El motor de alertas (`core_alertas.py`, ejecutado por el cron `tarea_alertas.py`) vigila niveles de plan, movimientos de precio y recomendaciones de analistas — pero **nunca vuelve a mirar noticias** para las posiciones que ya están en cartera. Esto es un gap distinto del hallazgo #30 (invalidación de tesis, que son condiciones cuantitativas ya calculadas por `core_plan_dca.invalidacion()`): aquí se trata de una fuente de información cualitativa completamente distinta — titulares de prensa — que hoy existe en el código pero no se monitoriza nunca entre sesiones del comité. Dado el principio no negociable de CLAUDE.md sobre rigor y trazabilidad, y que un evento cualitativo (fraude contable, dimisión del CEO, recorte de guidance, investigación regulatoria) suele mover el precio *antes* de que lo capture cualquier señal técnica (ATR, RSI) o cuantitativa (ROIC, Fair Value), cerrar este hueco da al comité una alerta temprana real sin necesidad de nuevas fuentes de datos: solo cablear lo que ya existe.

**Diseño.** Nuevo tipo de evento `"noticia"` en `core_alertas.py`, filtrado por palabras clave de evento material (para evitar saturar el mensaje de Telegram con ruido rutinario — 5 noticias × N posiciones por pase), con deduplicación por URL del titular (no por fecha, porque un titular no se repite):

```python
# config_settings.py — nueva constante
ALERTA_NOTICIAS_PALABRAS_CLAVE = (
    "guidance", "recorte de previsiones", "profit warning", "rebaja de previsiones",
    "downgrade", "rebaja de recomendación",
    "investigation", "investigación", "sec probe", "fraude", "fraud", "accounting",
    "resignation", "dimisión", "renuncia", "cese",
    "restructuring", "reestructuración", "layoffs", "despidos masivos",
    "bankruptcy", "quiebra", "default", "impago",
    "delisting", "exclusión de cotización",
    "recall", "retirada de producto",
    "lawsuit", "demanda", "litigio",
    "dividend cut", "recorte de dividendo", "suspende el dividendo",
)

# core_alertas.py
import hashlib
from config_settings import ALERTA_NOTICIAS_PALABRAS_CLAVE

def _es_material(titular: str) -> bool:
    t = titular.lower()
    return any(p in t for p in ALERTA_NOTICIAS_PALABRAS_CLAVE)

def alertas_noticias(posiciones: dict[str, dict], noticias_por_ticker: dict[str, list[dict]], hoy: date) -> list[dict]:
    """Titular con palabra clave de evento material en una posición abierta,
    una vez por titular (clave por URL: un mismo titular no vuelve a avisar
    aunque se reciba en pases sucesivos del cron)."""
    out = []
    for t, p in posiciones.items():
        if p.get("cerrada"):
            continue
        for n in noticias_por_ticker.get(t) or []:
            if not _es_material(n["titular"]):
                continue
            clave_url = hashlib.sha1(n["url"].encode()).hexdigest()[:12]
            out.append({
                "tipo": "noticia", "ticker": t, "clave": f"not:{t}:{clave_url}",
                "prioridad": 1,
                "texto": f"<b>{t}</b> noticia con posible impacto en la tesis ({n['fecha']:%d/%m}): "
                         f"<a href=\"{n['url']}\">{n['titular']}</a> ({n['fuente']})."
            })
    return out
```

Cableado en `tarea_alertas.py::main()`, justo después de construir `posiciones` (reutiliza `datos_finnhub.obtener_noticias`, ya cacheado con `TTL_NOTICIAS=900s`, adecuado para un cron):

```python
from datos_finnhub import obtener_noticias

noticias_por_ticker = {}
for t in (t for t, p in posiciones.items() if not p["cerrada"]):
    d = obtener_noticias(t)
    if d.ok:
        noticias_por_ticker[t] = d.valor
eventos += core_alertas.alertas_noticias(posiciones, noticias_por_ticker, hoy)
```

La deduplicación reutiliza la infraestructura ya existente (`db_supabase.alertas_ya_enviadas` / `registrar_alerta`, tabla `alertas_enviadas`), sin esquema nuevo. En `componer()`, el bucket de prioridad 1 ("⚡ Niveles y avisos") ya admite este tipo sin tocar los títulos.

**Alcance limitado a propósito:** solo tickers con posición abierta (`not p["cerrada"]`), no todo el universo del Rastreador, para no multiplicar llamadas a Finnhub innecesariamente y mantener el enfoque en proteger la tesis de inversión ya ejecutada — que es el principio rector del Veredicto de Inversión en CLAUDE.md.

## 3. Alertas para el debate de inversión

No hubo candidatas propuestas formalmente en Fase 1 de esta sesión, por lo que no aplica ninguna alerta de datos concreta. Siguen vigentes y aplicables a cualquier candidata futura: el stop-loss "con ATR" puede superar el 25% de riesgo anunciado cuando N3 cae mucho (#26, Alta); la banda de cordura del Fair Value nunca excluye un método disparatado cuando solo hay 2 valores disponibles (#28, Alta); el stop-loss fundamental (invalidación de tesis) se calcula una vez pero nunca se re-comprueba automáticamente (#30, Alta).
