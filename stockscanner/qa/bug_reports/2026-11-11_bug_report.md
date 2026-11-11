# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-11-11
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** 48be6c2 (sin cambios desde 2026-09-19 — confirmado de nuevo hoy con `git log -1 --format=%h -- stockscanner/app/`, decimosexta vez consecutiva)

## 0. Metodología de esta auditoría

Como en las últimas sesiones, toda la lectura línea a línea de `stockscanner/app/` ya está cubierta en auditorías previas (ver [registro_hallazgos.md](../registro_hallazgos.md)). Hoy se continuó la metodología de **ejecución real** (`python3.13 -I`, datos sintéticos) sobre módulos que aún no habían sido ejecutados explícitamente: `core_timing.py` (`calcular()`, `valores()`) y `datos_indices.py` (`normalizar()`). También se revisaron por lectura cruzada `core_analistas.py`, `core_referencias.py`, `core_rastreador.py`, `core_indicadores.py` y `datos_traduccion.py` buscando patrones nuevos; no se reproducen aquí los hallazgos que ya figuran en el registro con los mismos síntomas (p. ej. `core_analistas.resumen()` → hallazgo #3, confirmado por lectura pero no re-reportado).

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 1 (nuevo, #38) | **Alta** | `core_timing.py` (`calcular()`, línea 146) / `core_fair_value.py` (línea 260) | La nota final de **Timing** y de **Fair Value** se calcula con `ponderar(componentes, motivos=motivos)` **sin** pasar `cobertura_minima`, a diferencia de la nota final de **Calidad** (`core_calidad.py` línea 213, `cobertura_minima=CALIDAD_COBERTURA_MINIMA=0.50`) y de `core_rastreador.puntuacion()` (línea 126, `cobertura_minima=0.5`). Con `ponderar()` por defecto (`cobertura_minima=0.0`), basta **un solo componente con dato** para que `Ponderado.valor` deje de ser `None`: el "gate" de calidad sigue aplicándose, pero sobre una nota que puede sostenerse en una cobertura real ínfima. | Ejecución real (`python3.13 -I`) de `core_timing.calcular()` con un DataFrame OHLCV sintético de 300 sesiones pero `ind={}`, `fund={}`, `calidad=None`, `fair_value=None`, `earnings=None`, `plan=None`, `analistas=None`: el único componente con dato resultó ser OBV (calculado directamente del DataFrame, peso 4/100), y la función devolvió `nota=59.0`, `nota_bruta=60.66`, **`cobertura=0.04`** (4%) — una nota de Timing numéricamente "completa" sostenida en el 4% de los 15 componentes documentados. El mismo patrón de diseño (ponderar sin `cobertura_minima`) está presente, por lectura de código, en el `ponderar()` final de `core_fair_value.py` (línea 260), que calcula el upside/Fair Value mostrado al comité. | Dos de los tres pilares que CLAUDE.md exige en "triple condición" (Valor y Timing) pueden mostrar una nota y un veredicto de señal (`senal_timing`) con una cobertura de datos crítica sin que el usuario lo note salvo que mire explícitamente el campo `cobertura`/`cobertura_pct` en la interfaz; solo Calidad está protegida por un umbral mínimo (50%). Esto es relevante para el comité: una candidata con datos pobres de mercado (histórico corto, sin Fair Value, sin analistas) podría recibir igualmente una señal de Timing de apariencia fiable. |
| 2 (ampliación de #20, no nuevo) | Media → **se propone Alta** | `datos_indices.py` (`normalizar()`, líneas 29-43) | Confirmado por ejecución real que la lista blanca de sufijos occidentales no solo "bloquea ampliar `INDICES`" (como dice el hallazgo #20 original) sino que **corrompe el ticker en dos formas simultáneas** cuando la tabla de Wikipedia ya trae un sufijo no occidental: el punto se sustituye por guion (tratándolo como "clase de acción" al estilo `BRK.B`) y **además** se le vuelve a añadir el sufijo del índice, porque tras el `replace` ya no queda ningún punto en el string. | Ejecución real (`python3.13 -I`): `normalizar("0700.HK", ".HK") -> "0700-HK.HK"`; `normalizar("005930.KS", ".KS") -> "005930-KS.KS"`; `normalizar("BHP.AX", ".AX") -> "BHP-AX.AX"`; `normalizar("PETR4.SA", ".SA") -> "PETR4-SA.SA"`. Ninguno de los cuatro es un ticker válido en Yahoo Finance. | Si se amplía `INDICES` a mercados no occidentales (Hang Seng, KOSPI, ASX, Bovespa...) tal y como pide CLAUDE.md ("las candidatas pueden ser de cualquier mercado del mundo"), el rastreo nocturno no fallaría visiblemente — generaría tickers con apariencia plausible pero inválidos, que `datos_yfinance` simplemente no encontraría (dato ausente silencioso), no un error que salte a la vista. Se recomienda reclasificar #20 de Media a Alta dado que el riesgo real es corrupción silenciosa de datos, no solo una limitación de alcance. |

*(No se listan como nuevos los hallazgos #1-#37 del registro: todos siguen "Abierto" y el código no ha cambiado. Se confirmó por lectura cruzada que `core_analistas.resumen()` sigue reproduciendo exactamente el mecanismo de #3, y que `core_rastreador.py`/`core_referencias.py` no presentan excepciones ni comportamiento anómalo con datos sintéticos de borde — ver §4.)*

## 2. Mejoras propuestas

### Cálculo / lógica financiera

- **Gate de cobertura mínima simétrico entre los tres pilares** (cierra directamente el hallazgo #38 de hoy). Centralizar el umbral hoy exclusivo de Calidad en una sola constante de `config_settings.py` y aplicarla también a Timing y a Fair Value:

  ```python
  # config_settings.py
  COBERTURA_MINIMA_FIABLE = 0.50   # sustituye a CALIDAD_COBERTURA_MINIMA; mismo valor,
                                     # reutilizado por los tres motores + core_rastreador.puntuacion()

  # core_timing.py — calcular()
  p = ponderar(componentes, motivos=motivos, cobertura_minima=COBERTURA_MINIMA_FIABLE)
  # si p.valor es None (cobertura < 50%), la vista debe mostrar
  # "Timing no disponible: cobertura de datos insuficiente (<50%)"
  # en vez de forzar una nota con el gate de salud encima.

  # core_fair_value.py — ponderar() final (línea 260)
  p = ponderar(componentes, motivos={...}, cobertura_minima=COBERTURA_MINIMA_FIABLE)
  ```

  Esto no es solo un parche puntual: hoy el mismo patrón (`ponderar()` sin `cobertura_minima`) puede reaparecer en cualquier motor nuevo que se añada si no se documenta como convención obligatoria. Se propone además una regla de lint/convención explícita en el docstring de `core_ponderar.ponderar()`: "toda nota que alimente un veredicto final debe pasar `cobertura_minima` explícito, nunca el valor por defecto 0.0".

- **Corrección del sufijo bursátil en `datos_indices.normalizar()`** (cierra la ampliación de #20). Antes de tratar un punto como separador de clase de acción, comprobar si el texto ya termina en un sufijo bursátil conocido (occidental o no):

  ```python
  # datos_indices.py
  _SUFIJOS_CONOCIDOS = (".MC", ".DE", ".PA", ".AS", ".MI", ".BR", ".HE", ".L", ".SW", ".F",
                        ".IR", ".LS", ".VI", ".T", ".HK", ".KS", ".AX", ".SA", ".TO", ".SI",
                        ".BO", ".NS", ".SS", ".SZ")  # ampliar según mercados que se añadan a INDICES

  def normalizar(bruto: str, sufijo: str) -> str | None:
      ...
      t = m.group(0)
      if any(t.endswith(s) for s in _SUFIJOS_CONOCIDOS):
          return t                      # ya viene con sufijo válido: no tocar
      if "." in t:
          t = t.replace(".", "-")       # clase de acción (BRK.B)
      if sufijo and "." not in t:
          t += sufijo
      return t
  ```

### UI / UX

- **Badge explícito de "nota de baja fiabilidad"** independiente del gate de arriba: cuando `cobertura_pct` de Timing, Fair Value o Calidad caiga por debajo del umbral (incluso si no llega a devolver `None`, p. ej. justo en el límite), mostrar una etiqueta visible junto a la nota (no solo el número de cobertura en letra pequeña) — reduce el riesgo de que el comité o el usuario lean "Timing: 59 (ESPERAR)" como una señal tan fiable como un "Timing: 59" con cobertura del 95%.

### Arquitectura

- Ninguna nueva en esta sesión más allá de la centralización de `COBERTURA_MINIMA_FIABLE` arriba. Las propuestas de arquitectura de alto impacto ya pendientes de sesiones anteriores (verificación cruzada con segundo proveedor, `exposicion_prospectiva()`, divisas internacionales, migración de `empresas_analizadas.md` a Supabase, vigilancia activa de invalidación de tesis, gate de solvencia por patrimonio neto negativo) siguen todas sin implementar — el comité solo audita/diseña, nunca implementa en `stockscanner/app/`.

## 3. Alertas para el debate de inversión

> Cualquier problema de datos que pueda afectar la valoración de una candidata de esta sesión debe listarse aquí para que los analistas lo integren en la Fase 3.

- No se dispone en esta auditoría de las candidatas concretas que los Analistas 1-3 van a proponer hoy (la Fase 2 de QA se ejecuta en paralelo e independiente de la Fase 1), por lo que no se puede confirmar si el hallazgo #38 (notas de Timing/Fair Value con cobertura crítica) afecta a alguna candidata específica de esta sesión.
- **Alerta genérica aplicable a cualquier candidata de hoy:** si alguna candidata propuesta tiene histórico de precio corto (p. ej. una salida a bolsa reciente), sin Fair Value calculable, sin cobertura de analistas o sin plan de entrada (DCA) calculado, su nota de **Timing** mostrada en pantalla puede estar sostenida en una cobertura de datos muy baja (hallazgo #38) sin que la interfaz lo destaque con la misma fuerza que el número de la nota. Los analistas deben comprobar explícitamente el campo "cobertura" de Timing y de Fair Value de su candidata antes de citar esas notas como respaldo de la tesis, y declarar expresamente si la cobertura es baja (<50%), tal como exige el principio no negociable de CLAUDE.md de trazabilidad y declaración de estimaciones.
- Si alguna candidata de hoy cotiza en un mercado no occidental (Asia, Oceanía, Latam) y su ticker fue tomado de un listado de índice en vez de introducido manualmente, verificar manualmente que el ticker no esté corrompido por el mecanismo descrito en la ampliación de #20 (aunque hoy `INDICES` no incluye esos mercados, por lo que el riesgo práctico actual es bajo).
