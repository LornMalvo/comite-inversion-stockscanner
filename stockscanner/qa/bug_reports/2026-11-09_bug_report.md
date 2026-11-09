# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-11-09
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2` (confirmado sin cambios desde 2026-09-19 vía `git log --oneline -- stockscanner/app/` — decimoquinta confirmación consecutiva)

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 37 | **Alta** | `config_settings.CALIDAD_TRAMOS["deuda_patrimonio"]` (consumido por `core_ponderar.puntuar_tramos()` desde `core_calidad.py`) y `core_interpretar.semaforo_absoluto()` | El ratio Deuda/Patrimonio solo puede ser negativo cuando el patrimonio neto (equity) es negativo — insolvencia contable, la peor señal posible de "balance saneado". El tramo de puntuación empieza en `(0.0, 100)` y `puntuar_tramos()` satura por debajo del primer punto devolviendo siempre el valor del extremo: cualquier Deuda/Patrimonio negativo obtiene **100/100**, la nota máxima posible en el bloque "III. Salud financiera" de Calidad. En paralelo, `semaforo_absoluto()` para sentido `"menor"` devuelve `None` (sin lectura, ni rojo ni verde) para todo valor `<=0`, ocultando la alerta también en el semáforo de la UI. No existe en todo el código ningún otro chequeo que lea `patrimonio_neto`/equity directamente | Reproducido con ejecución real (`python3 -I`): `puntuar_tramos(-2.5, CALIDAD_TRAMOS["deuda_patrimonio"])` → `100.0` (igual para -5.0, -0.5 y 0.0); mismo patrón en `CALIDAD_TRAMOS_APALANCADOS["deuda_patrimonio"]`; `semaforo_absoluto("deuda_patrimonio", -2.5)` → `None` (vs. `"mal"` para 2.0 y `"bien"` para 0.25) | Si una candidata futura con patrimonio neto negativo se analizara en StockScanner (el propio comité rechazó dos tickers reales por este motivo exacto el 2026-11-04: RNG y VIK, "incumple el criterio de balance saneado"), la app mostraría "Deuda/Equity: 100/100" sin ningún aviso rojo — contradiciendo directamente el criterio no negociable de CLAUDE.md |

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- **Corrección directa del hallazgo #37:** antes de puntuar `deuda_patrimonio`, comprobar su signo. Si `deuda_patrimonio < 0` (equivalente a patrimonio neto negativo), excluirlo de `ponderar()` como "sin dato" con motivo `"patrimonio neto negativo: ratio no es una métrica de apalancamiento válida"`, en vez de dejar que `puntuar_tramos()` lo sature a 100. Mismo tratamiento para `semaforo_absoluto`: cuando `sentido == "menor"` y `valor < 0`, devolver `"mal"` explícito (no `None`) — a diferencia de `deuda_neta_ebitda`, donde negativo sí es genuinamente bueno (caja neta), aquí negativo nunca es bueno.

### Arquitectura — gate de solvencia explícito
Propuesta de alto impacto, motivada directamente por #37 y por el precedente real de RNG/VIK (2026-11-04): añadir un campo `patrimonio_neto` a `core_fundamentales.extraer()` (ya disponible en el balance de yfinance/Finnhub, hoy sin usar) y un gate duro en `core_calidad.py`, independiente de los tramos de cualquier ratio derivado:

```python
# core_calidad.py
def calcular(fund, ...):
    ...
    if es_dato(fund.get("patrimonio_neto")) and fund["patrimonio_neto"] <= 0:
        # Insolvencia contable: no es "dato ausente", es la peor señal posible.
        bloques["III. Salud financiera"]["nota"] = 0.0
        alerta_critica = "Patrimonio neto negativo: balance no saneado (criterio no negociable)"
    ...
    return {..., "alerta_critica": alerta_critica}
```

y mostrarlo en `ui_bloque_calidad_fv.py` como un aviso rojo fijo (no un semáforo más), igual de visible que la propia nota de Calidad — para que ningún analista del comité pueda pasarlo por alto como pasó manualmente con RNG/VIK, esta vez con la app confirmándolo en vez de contradiciéndolo.

## 3. Alertas para el debate de inversión

Ninguna candidata fue propuesta por los analistas en esta sesión (las tres investigaciones independientes de Fase 1 descartaron todos sus tickers). Alerta operativa general para sesiones futuras: si alguna candidata propuesta tiene patrimonio neto negativo o Deuda/Patrimonio negativo según cualquier fuente citada, **no usar la nota de Calidad ni el semáforo de StockScanner como evidencia de balance saneado** — por el hallazgo #37, la app mostraría esa métrica como óptima (100/100, sin aviso) en vez de señalar el problema; verificar siempre el patrimonio neto con la fuente primaria (10-Q/10-K o informe oficial), igual que se hizo manualmente el 2026-11-04 con RNG y VIK.
