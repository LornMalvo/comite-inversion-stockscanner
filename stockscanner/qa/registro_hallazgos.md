# Registro de hallazgos QA

> Estado de cada hallazgo detectado por el Agente 4 en cualquier auditoría. Consultar antes de auditar — ver regla de no-repetición en [CLAUDE.md](../../CLAUDE.md). No reportar como nuevo un hallazgo ya listado con el mismo estado; revisar un módulo ya cubierto solo si el código cambió desde la fecha de detección o para confirmar un fix aplicado.

| ID | Detectado | Módulo | Severidad | Descripción breve | Estado | Última revisión |
|---|---|---|---|---|---|---|
| 1 | 2026-09-19 | `core_plan_dca.py` (`plan()`) | Alta | Stop sin ATR no aplica el tope frente a N3 que sí existe en la rama con ATR | Abierto | 2026-09-25 |
| 2 | 2026-09-19 | `core_fundamentales.py` (`ffo_por_accion`, `p_ffo`) | Alta | P/FFO usa FFO del último ejercicio anual (no TTM) mientras el resto de múltiplos son trailing TTM | Abierto | 2026-09-25 |
| 3 | 2026-09-19 | `core_analistas.py` (`resumen()`) | Media | `comparado` no aplica el mismo filtro de `ANALISTAS_MIN_REVISION` que `revision()`; el texto narrativo puede citar un mes distinto al que sustenta el índice | Abierto | 2026-09-25 |
| 4 | 2026-09-19 | `core_paper.py` (`auto_ejecuciones`) | Media | Fills de Paper Trading usan siempre `min(objetivo, precio_actual)`, sesgando a la baja las salidas en huecos alcistas favorables | Abierto | 2026-09-25 |
| 5 | 2026-09-19 | `datos_cache.py` (`cubo_mercado`) | Media | El calendario de sesión de mercado usado para la caché asume siempre NYSE, pese a soportar 10 mercados/divisas distintos | Abierto | 2026-09-25 |
| 6 | 2026-09-19 | `db_supabase.py` / `datos_analisis.py` | Media | Fallos de escritura en Supabase (`guardar_analisis`, `guardar_multiplos`) se descartan silenciosamente sin avisar | Abierto | 2026-09-25 |
| 7 | 2026-09-19 | `ui_vistas_cartera.py` / `config_settings.CARTERA_DIVISAS_CONVERTIBLES` | Media | El formulario de alta de operaciones en Cartera solo admite EUR/USD pese a que el análisis soporta 10 divisas. Alcance ampliado 2026-09-21: la misma limitación afecta a `fmt_precio`/`fmt_importe` en toda la UI y a la columna "Cap. (M€)" de Comparables — ver hallazgo #10 | Abierto | 2026-09-25 |
| 8 | 2026-09-19 | `datos_yfinance.py` | Baja | Ninguna llamada a yfinance fija `timeout` explícito, a diferencia de Finnhub/Indices | Abierto | 2026-09-25 |
| 9 | 2026-09-19 | Todo el repo `stockscanner/` | Baja (estructural) | Cero tests automatizados pese a que los motores `core_*.py` son funciones puras diseñadas para testearse | Abierto | 2026-09-25 |
| 10 | 2026-09-21 | `ui_componentes.py` (`fmt_precio`, `fmt_importe`) / `ui_bloque_peers.py` (`_tabla`) / `datos_yfinance.convertir_a_eur` | Media | El equivalente en EUR mostrado junto a cada precio/importe de la app (no solo Cartera) depende de `CARTERA_DIVISAS_CONVERTIBLES = ("EUR","USD")`; cualquier divisa distinta muestra "EUR n/d" en Análisis Individual, Rastreador, Paper Trading, Favoritos y Comparables | Abierto | 2026-09-25 |
| 11 | 2026-09-21 | `ui_vistas_favoritos.py` (`render`) | Baja | La variación diaria ausente (`var is None`) se pinta en verde (`ss-var-pos`) en vez de quedar neutra, por `(var or 0) >= 0` evaluando `True` con `None` | Abierto | 2026-09-25 |
| 12 | 2026-09-23 | `.github/workflows/` (raíz) / `stockscanner/app/.github/workflows/` | **Alta** | Workflows de cron (rastreo nocturno, alertas) viven fuera de la única ruta que GitHub Actions reconoce en la raíz del repo; posible cron nunca ejecutado en producción | Abierto | 2026-09-25 |
| 13 | 2026-09-23 | `core_calidad.py` (`metricas()`, `calidad_beneficio`) | Media | `calidad_beneficio` compara FCF TTM con beneficio neto anual (no TTM), mismo patrón que el hallazgo #2 | Abierto | 2026-09-25 |
| 14 | 2026-09-23 | `core_cartera.py` (`resumen()`) | Media | `retorno_total_pct` mezcla `invertido` de todas las posiciones con `latente` solo de las que tienen precio disponible | Abierto | 2026-09-25 |
| 15 | 2026-09-23 | `ui_vistas_analisis.py` (`_earnings()`) | Baja | Ingresos real (con divisa, convertido a EUR) vs. estimado (sin divisa) mostrados con formato inconsistente | Abierto | 2026-09-25 |
| 16 | 2026-09-23 | `ui_metricas.py` (fila "Máximo 52 semanas") | Baja | Semáforo por signo simple pinta en rojo casi siempre esa fila, sin aportar señal real | Abierto | 2026-09-25 |
| 17 | 2026-09-23 | `config_sectores.py` (comentario `INDUSTRIAS_REIT`) | Baja | Comentario dice que P/FFO no está implementado; sí lo está en `core_fair_value.py` | Abierto | 2026-09-25 |
| 18 | 2026-09-23 | `config_settings.py` (`TIMING_PESOS`, comentario) | Baja | Línea de comentario residual/duplicada e inconsistente con la suma real de pesos | Abierto | 2026-09-25 |
| 19 | 2026-09-25 | `core_alertas.py` (`componer()`) / `tarea_alertas.py` (`main()`) / `tarea_rastreo.py` (`alertar()`) | Alta | Mensajes de Telegram no se truncan/parten; si superan 4096 caracteres el envío falla en silencio y los eventos no se marcan como avisados (se reintentan y agravan) | Abierto | 2026-09-25 |
| 20 | 2026-09-25 | `datos_indices.py` (`normalizar()`) | Media | La lista blanca de sufijos bursátiles no cubre mercados asiáticos/oceánicos/latam (`.T`, `.HK`, `.KS`, `.AX`, `.SA`, etc.); latente hoy, pero bloquea ampliar `INDICES` a mercados no occidentales | Abierto | 2026-09-25 |
| 21 | 2026-09-25 | `core_alertas.py` (`alertas_planes()`) | Baja | `es_dato(0.0)` es `True`; un nivel de plan con precio 0 provocaría `ZeroDivisionError` no capturado en `alertas_planes()` | Abierto | 2026-09-25 |
| 22 | 2026-09-25 | `tarea_rastreo.py` (`main()`) | Baja | Mensaje de log "ningún índice activo" se evalúa antes de añadir el índice virtual de Comparables, pudiendo aparecer aunque sí se rastree ese índice | Abierto | 2026-09-25 |

**Informes fuente:** [2026-09-19_bug_report.md](bug_reports/2026-09-19_bug_report.md), [2026-09-21_bug_report.md](bug_reports/2026-09-21_bug_report.md), [2026-09-23_bug_report.md](bug_reports/2026-09-23_bug_report.md), [2026-09-25_bug_report.md](bug_reports/2026-09-25_bug_report.md)
