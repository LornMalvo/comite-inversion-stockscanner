# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-07
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `0bbfac9` (sin cambios de código desde el commit inicial del repo `48be6c2`; confirmado con `git log --oneline -- stockscanner/app` y `git status --short`)

**Módulos revisados en esta auditoría (no cubiertos todavía en el registro, salvo confirmación puntual):** `core_fair_value.py`, `core_indicadores.py` (funciones distintas de `rsi()`, ya registrada como hallazgo #24), `core_interpretar.py`, `core_ponderar.py`, `core_rastreador.py`, `core_referencias.py`, `core_timing.py`, `datos_finnhub.py`, `datos_traduccion.py`, `ui_bloque_calidad_fv.py`, `ui_bloque_plan.py`, `ui_bloque_timing.py`, `ui_estilos.py`, `ui_graficos.py`, `ui_interfaz.py`, `ui_vistas_paper.py`, `ui_vistas_rastreador.py`, `config_secretos.py`.

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 28 | Alta | `core_fair_value.py` (`_banda_cordura()`, líneas 219-243) | El "ancla mixta" de la banda de cordura es la mediana de **todos** los valores disponibles, incluido el propio método que se está juzgando. Cuando solo hay 2 valores (el escenario de menor cobertura: un único método intrínseco + el consenso de analistas — justo el caso de mayor riesgo), la mediana de 2 números es su media, por lo que el ratio `método/ancla = 2·método/(método+consenso)` tiende asintóticamente a 2,0 por disparatado que sea el método, y **nunca puede alcanzar `FV_EXCLUSION_TECHO` (2,50)**. Un método corrupto o mal calculado en ese escenario queda como máximo "recortado" al 160% del ancla, nunca "excluido", y sigue pesando en el Fair Value final. | Reproducido en `python3.13 -I`: `_banda_cordura({'consenso': 10.0, 'per_forward': 10000.0})` → `{'consenso': 10.0, 'per_forward': 8008.0}`, estado `{'per_forward': 'recortado'}` (nunca "excluido"). Repetido con `per_forward = 100000.0` (10.000x el consenso): mismo resultado, solo "recortado" a `80008.0`. | El objetivo declarado de `_banda_cordura` ("excluye lo que se sale de la banda de cordura") queda matemáticamente inalcanzable justo en el caso de menor cobertura de métodos (empresas pre-rentables, financieras o con pocos comparables), que es cuando más falta hace el filtro de sanidad. Un dato corrupto o un múltiplo sectorial mal calculado se cuela ponderado en el Fair Value sin ninguna señal de alarma más allá de la cobertura baja ya mostrada. |

No se han encontrado más bugs nuevos reales (reproducibles) en el resto de módulos auditados hoy: se leyeron íntegros y se ejercitaron mentalmente/con sintéticos los casos límite (divisiones por cero, series vacías, `NaN`/`None`, coberturas parciales) sin encontrar otra discrepancia reproducible.

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- **Rediseñar el ancla de `_banda_cordura`** para que cada método se juzgue contra la mediana de los **otros** valores disponibles (excluyéndose a sí mismo), no contra una mediana que él mismo contamina; y/o exigir un mínimo de 3 métodos no-consenso disponibles antes de permitir la ponderación de un método "recortado" con peso pleno — si no se alcanza ese mínimo, degradar automáticamente su peso en vez de solo recortar el precio. Esto cierra el hueco del hallazgo #28 sin perder la filosofía de "nunca inventar, solo advertir".
- **Test de regresión concreto**: el bug es puramente aritmético y reproducible con datos sintéticos (sin red), por lo que es el candidato ideal para la primera prueba unitaria real del proyecto (ver hallazgo #9, ya abierto): `test_banda_cordura_excluye_outlier_con_solo_dos_metodos()`, fijando el caso exacto reproducido arriba como regresión permanente.

### UI / UX
- **`ui_vistas_rastreador.py` (`_rastrear`)**: las excepciones de `datos_analisis.analizar()` se descartan con `except Exception: a = None` y el ticker solo aparece en la lista `errores` sin el motivo. Para un analista que lanza un rastreo de 40-50 tickers de mercados no estadounidenses (el comité puede proponer candidatas de cualquier mercado), distinguir "ticker no encontrado en Yahoo" de "Finnhub sin key" o "rate limit" es clave para no descartar un valor por un error transitorio. Propuesta: capturar `str(e)` por ticker y mostrarlo en un expander junto a la lista de errores, en vez de solo el símbolo.
- **Comparación en vivo del Screener** (`ui_vistas_rastreador._resultados`, rama `screener`): si al pulsar "Comparar en vivo" uno de los tickers seleccionados falla el análisis, la comprobación `{tickers de vivo} == {tickers seleccionados}` nunca coincide y la función devuelve `[]` sin ningún mensaje — el usuario ve la comparación desaparecer sin saber por qué. Propuesta: si `vivo["errores"]` no está vacío, mostrar explícitamente qué ticker falló en vez de devolver una lista vacía silenciosa.

### Arquitectura
- **Contratos de datos tipados entre motores**: `fund`, `ind`, `calidad`, `fair_value`, `plan`, etc. viajan como `dict` sin esquema entre `core_fair_value.py`, `core_timing.py`, `core_rastreador.py` y las vistas `ui_*`. Una clave mal escrita en cualquiera de ellos se traduce silenciosamente en un `None` vía `.get()` (coherente con la "regla de oro" del proyecto de nunca convertir ausencia en cero, pero también oculta errores de programación, no solo de datos). Definir `TypedDict`/`dataclass` por motor (p. ej. `FundamentalesDict`, `FairValueResult`) y pasar `mypy`/`pyright` en CI haría visibles en revisión los mismos typos que hoy solo se detectan leyendo el código a mano.
- **Observabilidad de "anomalías" de Fair Value a nivel de cartera**: hoy `fv["anomalia"]` y `fv["avisos"]` solo se muestran por ticker individual (`ui_bloque_calidad_fv.py`). Con el Rastreador/Screener ya analizando decenas de tickers de golpe, añadir un contador agregado ("X de Y candidatas con upside fuera de rango razonable o con métodos recortados/excluidos") en la vista de resultados del Rastreador daría al comité una señal temprana de cuándo confiar menos en el ranking por puntuación antes de entrar al debate de la Fase 3.

## 3. Alertas para el debate de inversión

- **Fair Value con baja cobertura de métodos = mayor riesgo del que muestra el upside.** Por el hallazgo #28, cualquier candidata de hoy cuya tabla de "Métodos y banda de cordura" muestre cobertura baja (un solo método intrínseco + consenso) puede estar arrastrando un método "recortado" en vez de "excluido" aunque su valor bruto sea absurdo frente al resto. Los analistas deben revisar manualmente el detalle método-a-método (no solo el % de upside agregado) antes de citar el Fair Value de una candidata con cobertura de métodos por debajo del ~50%, especialmente farmacéuticas con guías recortadas por cargos únicos (caso de hoy: MRK, cuyo propio proponente reporta un rango de Fair Value muy disperso, 0-20% de descuento según método) o perfiles financieros/REIT.
- **RSI en racha alcista pura (hallazgo #24, ya abierto, no nuevo pero relevante hoy):** si alguna candidata lleva una racha de velas sin ninguna sesión bajista en los últimos 14 periodos, el RSI(14) sale como `NaN`/sin dato y el componente de Timing se excluye en vez de puntuar sobrecompra extrema — relevante para UNH y MRK, ambas en fuerte recuperación de precio en 2026.

## 4. Hallazgos previos a reconfirmar o cerrar

- Ninguno: código sin cambios desde `48be6c2`; los 27 hallazgos previos siguen abiertos tal como están registrados.
