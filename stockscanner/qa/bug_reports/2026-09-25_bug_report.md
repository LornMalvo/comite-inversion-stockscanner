# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-09-25
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2` (sin cambios en `stockscanner/app/` desde 2026-09-19, verificado con `git log`)

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 19 | Alta | `core_alertas.py` (`componer()`) + `tarea_alertas.py` (`main()`) + `tarea_rastreo.py` (`alertar()`) | No existe ningún límite ni troceo del mensaje antes de enviarlo por Telegram. `componer()` concatena todos los eventos sin comprobar longitud; Telegram rechaza mensajes de más de 4096 caracteres. | `componer()`: `return "StockScanner\n\n" + "\n\n".join(partes)` sin truncar. `datos_telegram.enviar()` hace `requests.post(...)` y devuelve `r.ok` (False si Telegram rechaza por longitud). En `tarea_alertas.main()`: si `enviar()` devuelve `False`, ninguna clave de alerta se registra en `alertas_enviadas`, así que en el siguiente pase se reintenta el mismo lote MÁS los nuevos eventos, agravando el problema. | Con una cartera de tamaño medio o una noche de alta volatilidad/muchas alertas de screener, el mensaje puede superar 4096 caracteres y el envío falla en silencio (solo visible en logs de GitHub Actions). El fallo es persistente y creciente porque los eventos no enviados nunca se marcan como avisados. Alertas críticas de STOP pueden quedar bloqueadas detrás de eventos de menor prioridad dentro del mismo mensaje fallido. |
| 20 | Media | `datos_indices.py` (`normalizar()`) | La lista blanca de sufijos bursátiles preservados (`.MC, .DE, .PA, .AS, .MI, .BR, .HE, .L, .SW, .F, .IR, .LS, .VI`) solo cubre EE.UU./Europa. Faltan sufijos asiáticos, oceánicos y latinoamericanos (`.T`, `.HK`, `.SS`/`.SZ`, `.KS`, `.AX`, `.NS`/`.BO`, `.SA`, `.TO`, `.SI`, `.TA`). | `if "." in t and not any(t.endswith(s) for s in (...)): t = t.replace(".", "-")`. Para un ticker japonés como `"7203.T"` (Toyota), el sufijo no está en la lista y el resultado es `"7203-T"`, inválido para Yahoo Finance. | Hoy `config_settings.INDICES` solo define S&P 500, Nasdaq 100, Dow Jones, IBEX 35 y Euro Stoxx 50 (sufijos ya cubiertos), así que el bug está latente. Pero CLAUDE.md exige candidatas de "cualquier mercado del mundo": en cuanto se active el rastreo de un índice asiático/latam/oceánico, sus tickers se normalizarán mal y el rastreo/Fair Value fallará silenciosamente para ellos. |
| 21 | Baja | `core_alertas.py` (`alertas_planes()`) | `objetivo = precio_nivel(p, nivel)` se valida con `es_dato(objetivo)`, pero `es_dato(0.0)` es `True`. Si un nivel del plan quedara guardado con precio `0` (dato corrupto), la siguiente línea `dist = (precio / objetivo - 1) * 100` lanza `ZeroDivisionError` sin capturar. | `if not es_dato(objetivo): continue` seguido de división sin comprobar `objetivo != 0`. | Muy improbable en condiciones normales, pero de darse, abortaría todo `tarea_alertas.py` (cron horario) para TODOS los planes esa hora, no solo el afectado. |
| 22 | Baja | `tarea_rastreo.py` (`main()`) | El mensaje `"Ningún índice activo..."` se imprime evaluando `objetivo` ANTES de añadir el índice virtual "Comparables" unas líneas después. | `if not objetivo: print(...)` ocurre antes del `append` del índice de comparables, en un bloque `if` independiente. | Solo cosmético/logging: puede aparecer el aviso aunque sí se rastree el índice virtual de comparables. No afecta al resultado, solo puede confundir al revisar logs de GitHub Actions. |

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- Centralizar la tabla de sufijos bursátiles por país (hoy embebida solo en `datos_indices.normalizar()`) en `config_sectores.py` o `config_settings.py`, ampliándola a Asia/Oceanía/Latam, con un test que verifique la normalización de una muestra de tickers por índice antes de activarlo.
- Añadir un guard explícito `objetivo > 0` (no solo `es_dato(objetivo)`) en `core_alertas.alertas_planes()` antes de dividir, coherente con el patrón ya usado en `core_indicadores.distancia_pct`.
- En `core_confluencia._pivotes_agrupados()`, el agrupado por tolerancia encadena contra el ÚLTIMO punto añadido al grupo, no contra el centro o el primer punto — con una cadena de toques puede formarse una "zona" artificialmente ancha. Revisar si conviene comparar contra la media del grupo.

### UI / UX
- Cuando `datos_telegram.enviar()` devuelve `False` por longitud excesiva (bug #19), no hay traza visible fuera del log de GitHub Actions. Añadir aviso o partir el mensaje en varios envíos evitaría la pérdida silenciosa de alertas.
- En `ui_vistas_rastreador.py`, indicar el coste aproximado en tiempo (nº de tickers × pausa entre peticiones) del análisis en vivo antes de que el usuario pulse el botón.

### Arquitectura
- Extraer un helper único `enviar_alertas(eventos, simular)` compartido entre `tarea_alertas.py` y `tarea_rastreo.py` que centralice troceo por longitud, registro de claves y manejo de errores de Telegram — hoy la lógica "componer → enviar → registrar solo si tuvo éxito" está duplicada casi literal en ambos scripts.
- Añadir tests unitarios ligeros (sin red) para `datos_indices.normalizar()` con una tabla de casos por mercado, dado que es una función pura que condiciona la corrección de todo el rastreo nocturno de índices no estadounidenses.

### Funcionalidad nueva de alto impacto
- **Backtesting realista del plan DCA:** `core_rastreador.evaluar_senales()` mide el retorno del precio desde la fecha de análisis, pero no simula haber ejecutado el plan de entradas escalonadas con el tamaño de posición real. Encadenar `core_paper` + el histórico de `analisis_historico` permitiría un backtest de rendimiento real de la metodología, no solo del ticker.
- **Métricas de riesgo de cartera** (Sharpe, volatilidad anualizada, máximo drawdown, beta vs. `BENCHMARK`): `core_cartera` y `ui_graficos.grafico_rendimiento` ya calculan la serie diaria de valor de cartera vs. benchmark; extensión natural con los datos ya existentes.
- **Señal de insider trading** como componente adicional de Timing/Confluencia: Finnhub ofrece `/stock/insider-transactions` en el plan gratuito (ya integrado en `datos_finnhub.py`); encajaría en `TIMING_PESOS`/`TIMING_FAMILIAS` siguiendo el mismo patrón de exclusión por dato faltante que el resto de `core_timing.py`.
- **Exportación CSV/Excel** de los resultados del Rastreador/Screener y de Cartera vía `st.download_button` sobre los `pd.DataFrame` ya construidos en `ui_vistas_rastreador._tabla()` y `ui_vistas_cartera.py` — bajo coste, alto valor para informes al comité.
- **Alerta de "mejora de fiabilidad de referencia":** cuando la fuente dominante de Fair Value de un ticker pasa de "sector (semilla)" a "sector real" o "comparables" (más fiable), un aviso puntual ayudaría a saber cuándo confiar más en una tesis — encaja con el patrón de claves de deduplicación ya usado en `core_alertas.py`.
- **Ampliar `INDICES`** a mercados no estadounidenses/europeos (Nikkei 225, Hang Seng, ASX 200, Sensex, Bovespa), una vez corregido el bug #20, en línea directa con el mandato de CLAUDE.md de analizar candidatas de cualquier mercado del mundo.

## 3. Alertas para el debate de inversión

- Ninguno de los hallazgos de hoy afecta directamente a los datos usados para valorar las tres candidatas de la sesión (V, MC.PA, GOOGL) — los tres cotizan en mercados/divisas ya cubiertos por la lista blanca de sufijos (`.PA` de LVMH está incluido).
- Sí es relevante de forma transversal: el bug de truncamiento de Telegram (#19) es un riesgo operativo para el seguimiento de CUALQUIER posición en cartera (incluida MU pendiente y la que se decida hoy) — una alerta crítica de stop podría no llegar si el mensaje agregado supera el límite de Telegram.

## 4. Notas sobre hallazgos previos

Los 18 hallazgos previos (registro completo en [registro_hallazgos.md](../registro_hallazgos.md)) siguen abiertos y sin cambios: el código de esos módulos no se ha modificado desde 2026-09-19 (verificado con `git log`). Módulos auditados hoy en detalle (todos sin hallazgos previos registrados): `app.py`, `config_secretos.py`, `core_alertas.py`, `core_confluencia.py`, `core_fair_value.py`, `core_indicadores.py`, `core_interpretar.py`, `core_ponderar.py`, `core_rastreador.py`, `core_referencias.py`, `core_timing.py`, `datos_finnhub.py`, `datos_indices.py`, `datos_telegram.py`, `datos_traduccion.py`, `tarea_alertas.py`, `tarea_rastreo.py`, `ui_bloque_calidad_fv.py`, `ui_bloque_plan.py`, `ui_bloque_timing.py`, `ui_estilos.py`, `ui_graficos.py`, `ui_interfaz.py`, `ui_vistas_paper.py`, `ui_vistas_rastreador.py`.
