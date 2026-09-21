# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-09-21
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** 5bd2749

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 10 | Media | `ui_componentes.py` (`fmt_precio`, `fmt_importe`) / `ui_bloque_peers.py` (`_tabla`) / `datos_yfinance.convertir_a_eur` | El equivalente en EUR mostrado junto a CADA precio/importe de la app (no solo en el alta de operaciones de Cartera, alcance del hallazgo #7) depende de `convertir_a_eur`, que solo convierte divisas en `CARTERA_DIVISAS_CONVERTIBLES = ("EUR", "USD")` (`config_settings.py:510`). | `fmt_precio` (`ui_componentes.py:90-100`): si `divisa` no es EUR y `convertir_a_eur` devuelve `None` (cualquier divisa que no sea USD), la función devuelve `"{base} (EUR n/d)"`. Igual en `fmt_importe` (líneas 77-87) y en la columna "Cap. (M€)" de `ui_bloque_peers._tabla` (línea 116). | Un candidato cotizado en GBP, JPY, CHF, SEK, HKD, etc. (mercados que CLAUDE.md permite explícitamente) muestra "EUR n/d" en Análisis Individual, Rastreador, Paper Trading, Favoritos y Comparables — es transversal a toda la interfaz de valoración, no solo el formulario de Cartera. |
| 11 | Baja | `ui_vistas_favoritos.py` (`render`, línea 30) | La variación diaria ausente se pinta con la clase CSS positiva (verde) en vez de quedar neutra. | `clase = "ss-var-pos" if (var or 0) >= 0 else "ss-var-neg"`: cuando `var` es `None`, `(var or 0) >= 0` es `True`, así que el badge se pinta verde con texto "n/d". Viola el principio del proyecto de no colorear datos ausentes, sí respetado en `core_interpretar.semaforo_signo` y `core_cartera._sem`. | Cosmético pero engañoso: un favorito sin cotización disponible aparece con el mismo color que una subida real. |

No se han encontrado discrepancias nuevas de severidad alta o crítica en los módulos aún no cubiertos por el registro (`core_indicadores.py`, `core_confluencia.py`, `core_timing.py`, `core_ponderar.py`, `core_cartera.py`, `core_referencias.py`, `core_rastreador.py`, `core_alertas.py`, `core_interpretar.py`, `datos_finnhub.py`, `datos_indices.py`, `datos_telegram.py`, `datos_traduccion.py`, `tarea_alertas.py`, `tarea_rastreo.py`, `app.py` y los `ui_*.py` restantes): el manejo de datos ausentes (`es_dato`, `ponderar()`, exclusión con motivo) es consistente en todos ellos.

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- Corregir #1 (stop sin ATR) y #2 (P/FFO no-TTM), ambos siguen abiertos — ver sección 4.
- Extender `CARTERA_DIVISAS_CONVERTIBLES` más allá de EUR/USD (GBP, JPY, CHF, SEK, HKD como mínimo) usando el mismo mecanismo de `fx_en_fecha`/`obtener_fx` ya construido para pares `XXXEUR=X`: un único punto de cambio en `config_settings.py` resuelve a la vez #7 y #10.
- El aviso de "divisa de los estados financieros distinta de la de cotización" en `core_fair_value.calcular` (línea 289) solo compara strings; podría reforzarse con una validación cruzada real de precio (contrastar yfinance con Finnhub o Stooq cuando estén disponibles) para detectar desalineaciones de fuente, no solo de divisa declarada.

### UI / UX
- En `fmt_precio`/`fmt_importe`, cuando la conversión a EUR no es posible, mostrar "sin conversión disponible" en vez de "(EUR n/d)", que puede leerse como "dato no disponible" en general.
- Corregir el badge de variación en Favoritos (#11) para que quede neutro sin dato, igual que en Cartera/Paper Trading.

### Arquitectura
- Sigue sin haber tests automatizados (#9). La mayoría de motores puros revisados hoy (`core_ponderar`, `core_indicadores`, `core_confluencia`, `core_timing`, `core_referencias`, `core_cartera`) son candidatas de bajo esfuerzo/alto valor para un primer paquete de tests de regresión (`ponderar()`, `puntuar_tramos()`, `atr()`/`rsi()`/`adx()` contra valores de referencia conocidos, `libro()`/`_vender()` de `core_cartera` para el FIFO).

### Funcionalidades nuevas de alto impacto
- **Validación cruzada de precio y fundamentales entre fuentes**: hoy el precio y los estados financieros dependen en exclusiva de yfinance; un chequeo cruzado ligero contra Finnhub `/quote` cuando la diferencia supere un umbral daría una alerta de calidad de dato antes de una decisión de inversión.
- **Insignia de "cobertura de divisa"** en Análisis Individual y Rastreador: cuando la divisa de cotización no es EUR/USD (afectada por #7/#10), señalar explícitamente que el equivalente EUR y el sizing en Paper Trading no son fiables.
- **Registro de reconciliación de estados financieros**: guardar en el análisis qué alias de fila de yfinance usó `roic()`/`ffo_por_accion()` (`_fila` prueba varios), para auditar comparabilidad entre empresas del mismo sector.

## 3. Alertas para el debate de inversión

- **Directamente relevante hoy:** de las 3 candidatas de la sesión, 2 cotizan en divisas distintas de USD/EUR (BATS.L/BTI en GBP, 8001.T/ITOCHU en JPY). El hallazgo #10 implica que si el comité usara StockScanner para dimensionar posiciones o comparar capitalización con comparables sobre estos dos tickers, el equivalente en EUR de precio, capitalización, caja y deuda aparecería como "n/d" — cualquier sizing o comparación en euros debe hacerse manualmente, no fiarse de la app tal cual está hoy.
- Si alguna candidata futura es un REIT, el P/FFO (hallazgo #2, confirmado abierto) mezcla FFO anual con el resto de métodos TTM — tratarlo como referencia de menor precisión temporal.
- Si alguna candidata futura no tiene ATR calculable, el stop del plan DCA queda sin el tope de seguridad frente a N3 (hallazgo #1, confirmado abierto) — no aplica a las 3 candidatas de hoy (historial largo en los tres casos), pero se mantiene como bloqueante documentado.
- El calendario de sesión de la caché asume siempre NYSE (hallazgo #5, confirmado abierto): para BATS.L (LSE) y 8001.T (TSE), el sello de frescura del precio en la app podría no reflejar correctamente los festivos locales si se consultaran ahí.

## 4. Estado de hallazgos previos (confirmación)

- **#1** (stop sin ATR, `core_plan_dca.plan()`): sigue abierto, no se aplicó fix. Código actual (líneas 141-149): la rama con ATR acota el stop dos veces frente a N3; la rama sin ATR no compara con N3.
- **#2** (P/FFO no-TTM, `core_fundamentales.py`): sigue abierto. `ffo_por_accion()` sigue usando `columna=0` (ejercicio anual), mientras el resto de múltiplos son TTM.
- **#3** (`core_analistas.resumen()` sin filtro `ANALISTAS_MIN_REVISION`): sigue abierto, idéntico comportamiento.
- **#4** (`core_paper.auto_ejecuciones` con `min(objetivo, precio_actual)`): sigue abierto, línea 200 idéntica.
- **#5** (`datos_cache.cubo_mercado` asume NYSE): sigue abierto, `_festivos_nyse()` sigue siendo el único calendario.
- **#6** (fallos de escritura en Supabase silenciados): sigue abierto, `except Exception: return False` sin logging.
- **#7** (formulario de Cartera solo EUR/USD): sigue abierto; su alcance real es más amplio de lo documentado — ver hallazgo #10 de hoy (afecta a toda la UI, no solo al alta de operaciones).
- **#8** (sin timeout en yfinance): confirmado, sigue abierto (no revisado en profundidad hoy, fuera del foco Alta/Media).
- **#9** (cero tests): sigue abierto.

**Informes fuente:** [2026-09-19_bug_report.md](2026-09-19_bug_report.md)
