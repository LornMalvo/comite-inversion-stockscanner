# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-28
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2` (confirmado con `git log -1 --format=%H -- stockscanner/app/`) — **sin cambios de código desde 2026-09-19, octava confirmación consecutiva** (tras las de 2026-09-21 a 2026-10-26)

## 1. Bugs y discrepancias de datos

Ninguno nuevo. La auditoría de hoy revisó con lectura de código y ejecución mental de casos borde: `core_ponderar.ponderar()` / `puntuar_tramos()` (componentes vacíos, pesos cero), `core_timing.calcular()` (se verificó que el gate de calidad —`CALIDAD_MINIMA_TIMING` / `TIMING_TOPE_SIN_SALUD`— se aplica tal y como describe su propio docstring, sin inconsistencia), `_obv_normalizado()` (borde de longitud mínima de histórico), `datos_indices.normalizar()` (se confirmó que `BRK.B`→`BRK-B` ya se gestiona correctamente; el hueco de sufijos no-occidentales sigue siendo el ya registrado hallazgo #20, sin caso nuevo) y `core_rastreador.py` completo (filas, puntuación, filtros, backtest). No se encontró ninguna ruta de fallo reproducible que no esté ya en [registro_hallazgos.md](../registro_hallazgos.md).

## 2. Mejoras propuestas

### Arquitectura (propuesta nueva de alto impacto)

**El control de concentración/exposición de la app no ve nada antes de ejecutar la operación.** `core_cartera.exposicion_sector()` solo agrega posiciones que ya están registradas como abiertas (`cerrada == False`) y con `valor_eur` calculable (depende de `valorar()`, que a su vez depende de tener un precio en EUR). Las 4 posiciones actuales del propio Comité (MU, V, AEM, ACN) están TODAS en estado "pendiente de ejecución" — nunca se registró una operación real en la tabla `cartera` de Supabase —, así que ni siquiera entrarían en el `dict` de posiciones que recibe esta función, y mucho menos en su agregación por sector. Resultado: la única herramienta de la app pensada para detectar concentración sectorial/temática está ciega justo en el momento en que el Comité más la necesita (antes de comprar), y por eso el Supervisor ha tenido que detectar a mano, sesión tras sesión, solapamientos de exposición que la app podría automatizar — por ejemplo: oro (AEM vs. FNV/WPM, descartadas el 2026-10-19 en parte por duplicar exposición) o memoria/HBM (MU vs. Samsung 005930.KS, descartada el 2026-09-23 por el mismo motivo).

Propuesta: una función nueva, `exposicion_prospectiva()`, que combine (a) las posiciones ya abiertas/ejecutadas (vía `exposicion_sector()` existente) y (b) una lista de posiciones "pendientes de ejecución" + watchlist que el propio Comité ya mantiene en Markdown (`posiciones_actuales.md`), agregando por **tema declarado** (no solo por el campo `sector` de yfinance, que no distingue "oro" de "minería diversificada" ni "memoria" de "semiconductores" en general), y alertando ANTES de que el Supervisor emita veredicto si una candidata de hoy se solaparía con algo ya en cartera o pendiente:

```python
def exposicion_prospectiva(abiertas: list[dict], pendientes: list[dict], candidatas: list[dict]) -> list[dict]:
    """abiertas: salida de exposicion_sector() (ya ejecutadas).
    pendientes: [{'ticker':..,'sector':..,'tema':..,'peso_pct_objetivo':..}] desde posiciones_actuales.md.
    candidatas: mismas claves, para las propuestas de HOY antes de aceptarlas.
    Devuelve alerta por tema/sector si una candidata se solaparía con algo
    ya en cartera o pendiente, ANTES de que el Supervisor emita veredicto."""
    acum = {d["sector"]: d["peso_pct"] for d in abiertas}
    for p in pendientes:
        clave = p.get("tema") or p.get("sector")
        acum[clave] = acum.get(clave, 0) + (p.get("peso_pct_objetivo") or 0)
    alertas = []
    for c in candidatas:
        clave = c.get("tema") or c.get("sector")
        if acum.get(clave, 0) > 0:
            alertas.append({"ticker": c["ticker"], "tema": clave, "exposicion_previa_pct": acum[clave]})
    return alertas
```

Esto convertiría una verificación manual y propensa a error en una comprobación automática y trazable, justo en la Fase 1/3 donde se decide si proponer o aceptar una candidata.

### Cálculo / lógica financiera
Sin novedad desde el 2026-10-26 (el patrón de "dato ausente tratado como válido" ya está cubierto por los hallazgos #11/#24/#31/#33/#35).

### UI / UX
Sin novedad.

## 3. Alertas para el debate de inversión

Ninguna alerta específica de datos de StockScanner aplica a las candidatas de hoy, porque ningún analista llegó a proponer formalmente ninguna en la Fase 1 (ver sesión completa). Alerta general para futuras sesiones: si alguna candidata comparte sector/tema con MU, V, AEM o ACN (ya en cartera, pendientes de ejecución) o con alguna entrada de la watchlist, la app **no lo detecta automáticamente hoy** (ver hallazgo de arquitectura arriba) — debe verificarse a mano, como se ha hecho en sesiones anteriores.
