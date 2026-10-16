# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-16
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `9711d6751357543395dcffc2b0f8d1220f0ba3ba` (HEAD). Confirmado con `git diff --stat fa2f0ed24bc6558d8bfe7f0d0f9e9bc00673b8ff -- stockscanner/app/` → sin salida: el código de `stockscanner/app/` sigue siendo exactamente el de `48be6c2`, sin cambios desde las auditorías de 2026-10-12/2026-10-14.

**Cobertura de esta auditoría:** los 41 `.py` de `stockscanner/app/` ya estaban al menos referenciados en `registro_hallazgos.md`. Se auditaron a fondo los tres módulos hasta ahora solo cubiertos de forma somera (`datos_telegram.py`, `ui_estilos.py`, `app.py` completos) y se ejecutó código real con datos sintéticos sobre el pipeline de recomendaciones de analistas (`datos_yfinance.py` + `datos_finnhub.py`), módulo poco ejecutado en auditorías anteriores (en 2026-10-05 solo se leyó, no se ejecutó). Esa ejecución encontró el hallazgo #33.

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 33 | **Alta** | `datos_yfinance.py` (`obtener_recomendaciones_yf()`, línea 443) / `datos_finnhub.py` (`_recomendaciones()`, línea 250, mismo patrón) / `ui_vistas_analisis.py` (línea 196) | El parseo de la serie mensual de recomendaciones usa `int(fila.get(k) or 0)` para cada categoría (`strongBuy`, `buy`, `hold`, `sell`, `strongSell`). Si esa celda es `NaN` (algo que `yf.Ticker(ticker).recommendations` produce cuando Yahoo no informa una categoría para un mes concreto, especialmente en tickers internacionales menos cubiertos), `NaN or 0` se evalúa a `NaN` (un `float('nan')` es *truthy*, el `or` nunca cae al `0`), y `int(NaN)` lanza `ValueError: cannot convert float NaN to integer`. La excepción no está capturada en el bucle de parseo y se propaga hasta el `except Exception as e: return Dato(None, str(e) or "sin recomendaciones", _ahora())` genérico, que deja el texto crudo de la excepción Python en el campo `fuente` del `Dato`, mostrado directamente al usuario por `ui_vistas_analisis.py:196`. | Reproducido en `python3 -I`: con un diccionario que simula la fila (`{'strongBuy': float('nan'), 'buy': 2, ...}`), `mes.get('strongBuy') or 0` da `nan` y `int(nan)` lanza `ValueError: cannot convert float NaN to integer` — comportamiento idéntico al de una `pandas.Series.get()` sobre esa misma celda (la truthiness de `float('nan')` es independiente del contenedor). | Cualquier ticker cuya serie de recomendaciones de Yahoo tenga un hueco de categoría en un mes pierde el bloque de analistas **completo** (no solo ese mes), y el usuario ve `Sin recomendaciones de analistas (cannot convert float NaN to integer)` — mensaje de implementación interna filtrado a la UI, no un "sin cobertura" entendible. Apaga además silenciosamente el componente `revisiones_analistas` del Timing (peso 3 pts, se redistribuye) sin que el motivo mostrado tenga relación con la causa real. |

**Hallazgos previos reconfirmados sin cambios** (código idéntico a `48be6c2`): se revisaron explícitamente `core_calidad.py`, `core_fair_value.py` completo, `core_timing.py`, `core_ponderar.py`, `core_rastreador.py`, `core_interpretar.py` y `core_referencias.py` — ningún hallazgo de cálculo nuevo distinto de #33; los patrones ya conocidos (#2, #3, #13, #26-#29) siguen intactos.

## 2. Mejoras propuestas

### Cálculo / lógica financiera

- **Corregir el parseo NaN-inseguro de recomendaciones (hallazgo #33)** en `datos_yfinance.py:443` y `datos_finnhub.py:250`:
```python
def _entero_seguro(v) -> int:
    try:
        f = float(v)
    except (TypeError, ValueError):
        return 0
    return int(f) if f == f else 0   # f == f es False solo para NaN
```
  y sustituir `int(fila.get(k) or 0)` por `_entero_seguro(fila.get(k))` en ambos sitios, tratando una categoría sin dato como 0 en vez de tirar toda la serie.

### UI / UX

- No mostrar nunca el `str(e)` crudo de una excepción inesperada como `fuente` al usuario (afecta a las 4 funciones `obtener_*` de `datos_finnhub.py` con el patrón `except Exception as e: return Dato(None, str(e) or "...", _ahora())` — noticias, calendario de resultados, recomendaciones y peers). Separar `_SinDatos` (mensaje ya pensado para el usuario) de cualquier otra excepción (texto genérico + detalle técnico solo en logs).

### Arquitectura

- **Punto único de responsabilidad para errores de datos de mercado**: un decorador/wrapper `_dato_seguro(fn, motivo_generico)` en un módulo común (p. ej. `datos_comun.py`) que capture `_SinDatos` tal cual y cualquier otra `Exception` con log + mensaje genérico, aplicado a las funciones públicas `obtener_*` de `datos_finnhub.py`/`datos_yfinance.py`. Mismo antipatrón de fondo que el hallazgo #6 (fallos silenciosos sin loguear en `db_supabase.py`), ahora confirmado también en la capa de datos de mercado.
- **Suite de tests unitarios para los motores `core_*.py`** (ligado al hallazgo #9): los hallazgos #26 a #33 de las últimas seis auditorías se han encontrado todos ejecutando los motores puros con datos sintéticos a mano. Formalizar `tests/test_core_*.py` con `pytest` y fixtures con huecos deliberados (NaN/None/0/negativo en cada frontera) habría detectado #33 automáticamente.

## 3. Alertas para el debate de inversión

- El hallazgo #33 es relevante para cualquier candidata con poca cobertura de analistas o cotización fuera de EE. UU.: si StockScanner mostrara "Sin recomendaciones de analistas (cannot convert float NaN to integer)" para un ticker internacional, no es ausencia real de cobertura — es este bug. No aplica a ninguna candidata concreta hoy porque ninguna llegó a proponerse formalmente.
- Siguen abiertos y potencialmente relevantes para futuras candidatas no estadounidenses: #7/#10 (conversión a EUR limitada a EUR/USD) y #20 (sufijos bursátiles no cubren mercados asiáticos/latam) — varios de los tickers investigados hoy por los tres analistas (WEGE3.SA, CSL.AX, BAP, KNEBV, B3SA3, 7309.T, ADM.L, ANTO.L, INVE-B.ST) habrían disparado exactamente estas limitaciones si se hubieran introducido en la app.
