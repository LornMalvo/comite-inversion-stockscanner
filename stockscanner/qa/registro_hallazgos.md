# Registro de hallazgos QA

> Estado de cada hallazgo detectado por el Agente 4 en cualquier auditoría. Consultar antes de auditar — ver regla de no-repetición en [CLAUDE.md](../../CLAUDE.md). No reportar como nuevo un hallazgo ya listado con el mismo estado; revisar un módulo ya cubierto solo si el código cambió desde la fecha de detección o para confirmar un fix aplicado.

| ID | Detectado | Módulo | Severidad | Descripción breve | Estado | Última revisión |
|---|---|---|---|---|---|---|
| 1 | 2026-09-19 | `core_plan_dca.py` (`plan()`) | Alta | Stop sin ATR no aplica el tope frente a N3 que sí existe en la rama con ATR | Abierto | 2026-09-21 |
| 2 | 2026-09-19 | `core_fundamentales.py` (`ffo_por_accion`, `p_ffo`) | Alta | P/FFO usa FFO del último ejercicio anual (no TTM) mientras el resto de múltiplos son trailing TTM | Abierto | 2026-09-21 |
| 3 | 2026-09-19 | `core_analistas.py` (`resumen()`) | Media | `comparado` no aplica el mismo filtro de `ANALISTAS_MIN_REVISION` que `revision()`; el texto narrativo puede citar un mes distinto al que sustenta el índice | Abierto | 2026-09-21 |
| 4 | 2026-09-19 | `core_paper.py` (`auto_ejecuciones`) | Media | Fills de Paper Trading usan siempre `min(objetivo, precio_actual)`, sesgando a la baja las salidas en huecos alcistas favorables | Abierto | 2026-09-21 |
| 5 | 2026-09-19 | `datos_cache.py` (`cubo_mercado`) | Media | El calendario de sesión de mercado usado para la caché asume siempre NYSE, pese a soportar 10 mercados/divisas distintos | Abierto | 2026-09-21 |
| 6 | 2026-09-19 | `db_supabase.py` / `datos_analisis.py` | Media | Fallos de escritura en Supabase (`guardar_analisis`, `guardar_multiplos`) se descartan silenciosamente sin avisar | Abierto | 2026-09-21 |
| 7 | 2026-09-19 | `ui_vistas_cartera.py` / `config_settings.CARTERA_DIVISAS_CONVERTIBLES` | Media | El formulario de alta de operaciones en Cartera solo admite EUR/USD pese a que el análisis soporta 10 divisas. Alcance ampliado 2026-09-21: la misma limitación afecta a `fmt_precio`/`fmt_importe` en toda la UI y a la columna "Cap. (M€)" de Comparables — ver hallazgo #10 | Abierto | 2026-09-21 |
| 8 | 2026-09-19 | `datos_yfinance.py` | Baja | Ninguna llamada a yfinance fija `timeout` explícito, a diferencia de Finnhub/Indices | Abierto | 2026-09-19 |
| 9 | 2026-09-19 | Todo el repo `stockscanner/` | Baja (estructural) | Cero tests automatizados pese a que los motores `core_*.py` son funciones puras diseñadas para testearse | Abierto | 2026-09-21 |
| 10 | 2026-09-21 | `ui_componentes.py` (`fmt_precio`, `fmt_importe`) / `ui_bloque_peers.py` (`_tabla`) / `datos_yfinance.convertir_a_eur` | Media | El equivalente en EUR mostrado junto a cada precio/importe de la app (no solo Cartera) depende de `CARTERA_DIVISAS_CONVERTIBLES = ("EUR","USD")`; cualquier divisa distinta muestra "EUR n/d" en Análisis Individual, Rastreador, Paper Trading, Favoritos y Comparables | Abierto | 2026-09-21 |
| 11 | 2026-09-21 | `ui_vistas_favoritos.py` (`render`) | Baja | La variación diaria ausente (`var is None`) se pinta en verde (`ss-var-pos`) en vez de quedar neutra, por `(var or 0) >= 0` evaluando `True` con `None` | Abierto | 2026-09-21 |

**Informes fuente:** [2026-09-19_bug_report.md](bug_reports/2026-09-19_bug_report.md), [2026-09-21_bug_report.md](bug_reports/2026-09-21_bug_report.md)
