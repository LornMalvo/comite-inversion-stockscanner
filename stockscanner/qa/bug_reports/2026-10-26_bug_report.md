# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-26
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `bf858df` (stockscanner/app/ sin cambios desde `48be6c2` — confirmado por **séptima vez consecutiva** vía `git diff 48be6c2 HEAD -- stockscanner/app/`, diff vacío)

**Módulos cubiertos hoy con lectura línea a línea (primera vez explícita):** `ui_bloque_calidad_fv.py`, `ui_estilos.py`, `ui_graficos.py`, `sql_migracion_002.sql`, `sql_migracion_003.sql`, `sql_migracion_004.sql`, `CHANGELOG.md`, `requirements.txt`. Con esta auditoría, **todos** los módulos de `stockscanner/app/` han recibido ya una lectura detallada explícita en algún momento de las 17 auditorías del comité.

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 35 | Media | `core_indicadores.py` (`macd()`) / `ui_graficos.py` (`grafico_precio_macd`) | El MACD usa `Series.ewm(adjust=False)` con `ignore_na=False` (comportamiento por defecto de pandas): ante un hueco real de dato (`Close = NaN` un día concreto, habitual en tickers poco líquidos o recién salidos a bolsa — el perfil de muchas candidatas internacionales de este comité), el EWM **arrastra silenciosamente el valor anterior** en vez de propagar el hueco. La media móvil simple (`media_movil()`, base de MM50/100/200) sí se comporta correctamente: cualquier ventana que toque el hueco da `NaN`. Dos indicadores sobre el mismo precio, comportamiento inconsistente ante el mismo defecto de dato. | Reproducido en `python3.13 -I` con `Close = [100, 101, NaN, 103, 104, 105, 106, 107, 108, 109]`: `macd()` no produce ningún `NaN` en `hist` (arrastra el valor del día anterior en la fila con hueco); `media_movil(df, 3)` sobre el mismo `Close` da `NaN` en las 5 filas que tocan el hueco. | El histograma/línea MACD de Análisis Individual puede mostrarse en verde (momentum positivo) en una fecha sin cotización real, sin ningún aviso visual — misma familia de fondo que #11/#24/#31/#33 (dato ausente tratado como dato válido), pero con mecanismo distinto (arrastre de `ewm`, no `or 0`/`int(NaN)`), por lo que se registra como hallazgo nuevo. |
| 36 | Baja | `ui_estilos.py` (bloque `CSS`, reglas `.ss-var-neg` y `.ss-frescura-aviso`) | El docstring del módulo dice exponer "la paleta corporativa... como variables CSS", pero solo importa/expone como `--ss-*` 9 de los ~13 colores de `config_settings.py`. Los 4 colores de alerta (`C_ROJO`, `C_NARANJA`, `C_VERDE_OSCURO`, `C_ROJO_OSCURO`) no se importan ni se exponen como variable: se repiten como literales hexadecimales sueltos (`#dc2626`, `#f97316`) que hoy coinciden por casualidad con `config_settings.C_ROJO`/`C_NARANJA`, pero no están enlazados a ellos. | `config_settings.py`: `C_ROJO = "#dc2626"`, `C_NARANJA = "#f97316"`. `ui_estilos.py` repite esos valores como texto literal en `.ss-var-neg`/`.ss-frescura-aviso`, sin `{C_ROJO}`/`{C_NARANJA}` ni variable `--ss-rojo`/`--ss-naranja` en `:root`. | Si se cambia `C_ROJO`/`C_NARANJA` en `config_settings.py` (la fuente única de verdad que el módulo dice representar), el color de variación negativa o del aviso de frescura de dato en `ui_estilos.py` NO cambiaría, mientras que el resto de la app (que importa esas constantes directamente, p. ej. `core_plan_dca.PLAN_COLOR_STOP` o los badges de `ui_bloque_calidad_fv.py`) sí lo haría — dos tonos de "rojo de alerta" distintos convivirían en la misma pantalla sin que nadie lo note a simple vista. |

Sin hallazgos nuevos en `sql_migracion_002/003/004.sql`, `CHANGELOG.md` ni `requirements.txt`: las tres migraciones son idempotentes de forma correcta (`if not exists` consistente; el backfill de `anotaciones_entradas` usa `not exists` a nivel de ticker, sin duplicar en una segunda ejecución), y `requirements.txt`/`CHANGELOG.md` son coherentes con el código real observado.

## 2. Mejoras propuestas

### Cálculo / lógica financiera

- **Aviso explícito de "hueco en el histórico" antes de dibujar MACD/RSI/ADX.** Añadir una función pura en `core_indicadores.py` que detecte huecos reales en `Close` (no festivos/fin de semana) y los exponga como metadato, en vez de que cada indicador los trate de forma distinta (hallazgo #35, misma familia que #24/#31):

  ```python
  def huecos_cotizacion(df: pd.DataFrame) -> list[pd.Timestamp]:
      """Fechas de sesión bursátil (días hábiles dentro del rango del propio
      índice) donde Close es NaN: huecos reales de dato, no festivos."""
      return df.index[df["Close"].isna()].tolist()
  ```

  Reutilizar el patrón de aviso `⚠` ya existente en `ui_bloque_calidad_fv._fair_value` (estilo `.ss-anotacion`, color `C_NARANJA`): si `huecos_cotizacion(df)` no está vacío, mostrar junto al gráfico `f"⚠ {n} día(s) sin cotización en el histórico: MACD puede estar suavizado artificialmente ese tramo"`.

### UI / UX

- **Exponer en `ui_estilos.py` los 4 colores de alerta que faltan como variables CSS** (cierra el hallazgo #36): añadir `C_VERDE_OSCURO`, `C_NARANJA`, `C_ROJO`, `C_ROJO_OSCURO` al `import` y al bloque `:root`, y sustituir los literales de `.ss-var-neg`/`.ss-frescura-aviso` por `var(--ss-rojo)`/`var(--ss-naranja)`. Cambio mecánico y de bajo riesgo que cierra la brecha de "fuente única de verdad" que el propio módulo declara tener.

### Arquitectura

- **Flag de "fuente única, sin verificación cruzada" en Calidad/Fair Value — y, a medio plazo, verificación cruzada real con un segundo proveedor.** `roic()` y el resto de métricas de Calidad/Fair Value de StockScanner se calculan hoy a partir de una única fuente (estados financieros de yfinance); Finnhub en este repo solo aporta noticias, earnings, recomendaciones de analistas y peers — ningún dato fundamental propio con el que cruzar ROIC o múltiplos. Repasando `empresas_analizadas.md`, el motivo de descarte más repetido de las últimas sesiones (más de 15 tickers solo en las últimas tres sesiones: 4063.T, ICE, PRU, TTE, 005380.KS, CSL.AX, JHX, POOL, WSO, HEI, SOON.SW, RELX, EXPN.L, IFC.TO, DHR...) es "ROIC/DCF/P-E irreconciliable o en disputa de signo entre fuentes" — un trabajo de verificación manual que los tres analistas repiten cada sesión fuera de la app, porque la app misma no tiene con qué contrastarse. Propuesta en dos fases:

  1. **Fase honesta (bajo coste):** marcar en la UI, junto a cada métrica de Calidad/Fair Value dependiente de un único proveedor, una etiqueta discreta `fuente única · no verificada de forma independiente`, reutilizando el patrón `referencia`/`ref.` que ya existe en `ui_bloque_calidad_fv._fair_value` para los métodos de Fair Value.
  2. **Fase de verificación cruzada real:** usar el endpoint `/stock/metric?metric=all` de Finnhub (no utilizado hoy en este repo) como segunda fuente de ROIC/márgenes/múltiplos, con una función de reconciliación:

     ```python
     def reconciliar(clave: str, valor_yf: float | None, valor_finnhub: float | None,
                      umbral_pct: float = 0.25) -> dict:
         """Compara el mismo dato de dos proveedores; si ambos existen y
         divergen más de umbral_pct, lo marca 'en_disputa' en vez de elegir
         uno silenciosamente."""
         if not es_dato(valor_yf) or not es_dato(valor_finnhub):
             return {"valor": valor_yf if es_dato(valor_yf) else valor_finnhub,
                     "estado": "fuente_unica"}
         divergencia = abs(valor_yf - valor_finnhub) / max(abs(valor_yf), abs(valor_finnhub), 1e-9)
         if divergencia > umbral_pct:
             return {"valor": None, "estado": "en_disputa",
                     "motivo": f"yfinance {valor_yf:.2f} vs Finnhub {valor_finnhub:.2f}"}
         return {"valor": (valor_yf + valor_finnhub) / 2, "estado": "reconciliado"}
     ```

     Esto convertiría en una comprobación automática de un segundo algo que hoy el comité descubre manualmente leyendo 3-4 webs externas por candidata — y habría marcado "en disputa" directamente en la app varios casos reales ya vividos por este comité (p. ej. el ROIC de BTI 9,68% vs. GF Value en sentido contrario a los DCF citados, o el P/E de Novonesis 42,9x-88,5x según fuente).

## 3. Alertas para el debate de inversión

- Ninguna de las dos mejoras ni los dos hallazgos nuevos de hoy (#35 MACD, #36 CSS) afecta a ningún cálculo de valoración ya guardado en la cartera actual (MU, V, AEM, ACN) ni condiciona datos de candidatas de esta sesión — no hay ninguna candidata propuesta hoy (ver Fase 1), y ninguno de los dos hallazgos toca ROIC, Fair Value ni niveles de plan.
- Recordatorio operativo (no es alerta nueva): el hallazgo #30 (Abierto desde 2026-10-12) sigue sin resolver — ninguna de las cuatro posiciones vivas de la cartera (MU, V, AEM, ACN) tiene su stop loss fundamental revisado automáticamente contra datos nuevos.
