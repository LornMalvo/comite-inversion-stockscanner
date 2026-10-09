# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-09
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2` (sin cambios de código desde el commit inicial del repo; confirmado con `git log --oneline -- stockscanner/app` y `git status --short`)

**Método:** se leyó primero [registro_hallazgos.md](../registro_hallazgos.md) (28 hallazgos) para no repetir nada — todos siguen "Abierto" en el mismo estado, sin necesidad de reescritura. Se eligieron 3 módulos de la lista "leídos pero no ejecutados todavía con datos sintéticos": `core_fundamentales.py`, `core_calidad.py` y `core_timing.py` (más una revisión ligera de `core_cartera.py` y `core_analistas.py` para confirmar que no hay bug nuevo aparte de los ya registrados #3 y #14). Se ejecutaron con `python3.13 -I` y datos sintéticos de casos límite (ceros, series vacías, cobertura parcial).

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 29 | Media | `core_fundamentales.py` (`ffo_por_accion()`, líneas 71-72) | `_fila()` devuelve `0.0` como valor válido cuando el campo existe y es cero (distinto de `None`). El `or` de Python trata `0.0` como falsy, así que si el flujo de caja reporta una Depreciación y Amortización real de exactamente `0.0`, el código IGNORA ese dato correcto y lo sustituye por `_fila(res, "Reconciled Depreciation")` (una fuente distinta, con semántica contable distinta), alterando el FFO/acción de forma silenciosa. El patrón correcto habría sido comprobar `is None`, como sí hace `roic()` dos líneas más abajo (línea 44). Patrón relacionado (mismo antipatrón, no reportado aparte por bajo impacto aislado): línea 97, `precio = _num(info.get("currentPrice")) or _num(info.get("regularMarketPrice"))`. | Reproducido en `python3.13 -I`: `cf.ffo_por_accion({'resultados': pd.DataFrame({'2025-12-31':[500.0,90.0]}, index=['Net Income','Reconciled Depreciation']), 'flujo_caja': pd.DataFrame({'2025-12-31':[0.0]}, index=['Depreciation And Amortization'])}, acciones=100.0)` → `5.9` (usa Reconciled Depreciation=90 en vez del D&A real de 0.0). Esperado si se respetara el dato real: `(500+0)/100 = 5.0`. | `ffo_por_accion` alimenta `fund["ffo_por_accion"]`/`fund["p_ffo"]` (líneas 112-113), que `core_fair_value.py` (líneas 157-159) usa para el método "P/FFO" de Fair Value de REITs — con **peso 0,40** en `config_settings.py` (línea 222), el método más pesado disponible para ese sector. Un REIT con D&A=0.0 real en un periodo (infrecuente pero posible si yfinance rellena un campo ausente con 0 en vez de `NaN`) vería su Fair Value distorsionado por una fuente de dato equivocada, sin ningún aviso en la interfaz. |

No se han encontrado más bugs nuevos reales (reproducibles) en el resto de módulos auditados hoy:
- `core_calidad.py` `calcular()`: probado con empresa financiera sin estados financieros completos (sector "Financial Services"). El bloque "III. Salud financiera" (peso 30/100) queda con cobertura 0 por diseño, se excluye correctamente vía `ponderar()`, y la nota total cae a `None` por debajo de `CALIDAD_COBERTURA_MINIMA` (0,50) — comportamiento intencionado, no es un bug.
- `core_timing.py` `calcular()`/`_puntos()`: revisada la rama especial de `volumen_relativo` (detección de distribución); el "fall-through" a `puntuar_tramos` por defecto es correcto.
- `core_cartera.py`: revisados `momentum()`, `recomendar()`, `resumen()`, `correlacion()` — `resumen()` reproduce exactamente el patrón ya registrado en #14, no es nuevo.
- `core_analistas.py`: confirmado que `resumen()` reproduce el patrón ya registrado en #3, no es nuevo.

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- **Helper centralizado `primero_valido()` en `core_ponderar.py`** (módulo "regla de oro" de datos ausentes), para sustituir el patrón `a or b` cada vez que se encadenan fuentes alternativas de un dato numérico que podría ser legítimamente `0.0`:
  ```python
  def primero_valido(*valores) -> float | None:
      """Primer valor de `valores` que sea un dato real (es_dato), sin que
      un 0.0 legítimo se confunda con 'ausente' como haría `a or b`."""
      for v in valores:
          if es_dato(v):
              return float(v)
      return None
  ```
  Corrige el hallazgo #29 de raíz (`core_fundamentales.py` líneas 72 y 97) y previene toda una clase de errores silenciosos de "dato cero tratado como ausente" en el resto del código.
- **Test de regresión concreto**: el bug es puramente aritmético y reproducible con datos sintéticos (sin red) — candidato ideal junto al hallazgo #28 para las primeras pruebas unitarias reales del proyecto (hallazgo #9, ya abierto): `test_ffo_por_accion_no_descarta_depreciacion_cero()`.

### Funcionalidad nueva de alto valor
- **Módulo de backtesting histórico (`core_backtest.py`).** Los propios docstrings de `core_calidad.py` (línea 10: "para que el backtesting pueda reconstruirla sobre estados de otra fecha") y `core_timing.py` (línea 10: "reconstruible para el backtesting a cualquier fecha") afirman que estos motores están diseñados como funciones puras evaluables contra el pasado, pero no existe ningún módulo de backtesting: la única validación de la metodología "triple condición" es hoy el debate cualitativo del comité, nunca una comprobación empírica de si esa combinación de pesos predijo rendimiento superior. Pseudocódigo de diseño:
  ```python
  def backtest_ticker(ticker: str, cortes: list[date], estados_por_fecha: dict,
                       precios: pd.Series, horizontes_meses=(1, 3, 6, 12)) -> list[dict]:
      """Para cada fecha de corte: reconstruye fund/estados a esa fecha, llama a
      core_calidad.calcular + core_timing.calcular + core_fair_value, registra la
      señal (nota_calidad, upside_fv, nota_timing) y el retorno real del precio a
      cada horizonte futuro."""
      filas = []
      for fecha in cortes:
          calidad = core_calidad.calcular(fund_en(fecha), estados_en(fecha))
          timing = core_timing.calcular(...)
          fv = core_fair_value.calcular(...)
          precio_en_fecha = precios.asof(fecha)
          fila = {"fecha": fecha, "nota_calidad": calidad["nota"],
                  "upside": fv.get("upside_pct"), "nota_timing": timing["nota"]}
          for h in horizontes_meses:
              precio_fwd = precios.asof(fecha + relativedelta(months=h))
              fila[f"retorno_{h}m"] = (precio_fwd / precio_en_fecha - 1) * 100 if es_dato(precio_fwd) else None
          filas.append(fila)
      return filas
  ```
  Uso: correr esto sobre el histórico de `empresas_analizadas.md` y la cartera cerrada, para responder con datos — no con intuición — si señales con Calidad/Timing altos batieron al mercado, y así validar o recalibrar los pesos de `CALIDAD_BLOQUES`/`TIMING_PESOS`.

## 3. Alertas para el debate de inversión

- Ninguna de las dos candidatas de hoy (BRK.B, LLY) es un REIT, por lo que el hallazgo #29 (P/FFO) no les afecta directamente.
- Ambas candidatas presentan, por admisión de su propio proponente, una dispersión alta entre fuentes de Fair Value (BRK.B: suma de partes ~$625 vs. P/B ~fair; LLY: rango $1.164-$1.755 según método). Esto es exactamente el escenario de "baja cobertura/alta divergencia entre métodos" que el hallazgo #28 (ya abierto, `core_fair_value._banda_cordura`) demuestra que la app no sabría manejar bien: un método muy alejado del resto quedaría como máximo "recortado", nunca "excluido". Si el comité recalculara el Fair Value de cualquiera de las dos candidatas con StockScanner tal cual está hoy, debería revisar el detalle método a método, no solo el % de upside agregado.
