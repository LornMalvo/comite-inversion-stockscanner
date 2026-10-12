# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-12
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2` (sin cambios de código en `stockscanner/app/` desde el commit inicial del repo; confirmado con `git log --oneline -20 -- stockscanner/app/`, que solo devuelve ese commit).

**Método:** se leyó primero [registro_hallazgos.md](../registro_hallazgos.md) (29 hallazgos, todos "Abierto") para no repetir nada. Como el código no ha cambiado, se priorizó explorar módulos de `stockscanner/app/` que el registro no mencionaba todavía: `core_ponderar.py`, `core_timing.py`, `core_rastreador.py`, `core_interpretar.py`, `core_referencias.py`, `core_paper.py` (completo), `datos_finnhub.py`, `datos_traduccion.py`, `config_secretos.py`, `db_supabase.py` (completo), `ui_graficos.py`, `ui_bloque_timing.py`, `ui_bloque_plan.py`, `ui_bloque_calidad_fv.py`, `ui_vistas_paper.py`, `ui_vistas_rastreador.py`, `core_indicadores.py` (adx/obv), y se repasó `core_fair_value.py`/`core_fundamentales.py`/`core_calidad.py` completos (no solo las líneas ya marcadas) para confirmar que no hay nada nuevo. Se ejecutó código real con `python3.13` (pandas 3.0.5 / numpy 2.5.3) y datos sintéticos para los motores puros.

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| N1 | **Alta** | `sql_migracion_001.sql` (tabla `tesis_invalidacion`, líneas 87-94) / `core_plan_dca.invalidacion()` (líneas 290-308) / `core_alertas.py` / `tarea_alertas.py` / `tarea_rastreo.py` | Funcionalidad de "stop loss fundamental" a medio construir y **nunca conectada**: `core_plan_dca.invalidacion()` calcula condiciones concretas y cuantificadas (`ruptura_soporte`, `calidad_degradada` con umbral numérico, `fair_value_caida` con umbral -15%, `earnings_debiles`) y `datos_analisis.py` las adjunta a cada análisis; `ui_bloque_plan.py` solo las pinta como texto estático dentro de un `st.toggle` (líneas 95-99). La tabla `tesis_invalidacion` existe en el esquema SQL con columnas pensadas exactamente para vigilarlas (`activa`, `disparada_en`), pero ninguna función de `db_supabase.py` lee ni escribe esa tabla, ningún cron las evalúa contra datos nocturnos, y `core_alertas.py` no tiene ningún tipo de evento para ellas (su docstring enumera 6 tipos — `nivel, cerca, movimiento, reco, resumen, screener` — ninguno es "invalidación de tesis"). | `grep -rn "tesis_invalidacion\|invalidacion" tarea_alertas.py tarea_rastreo.py core_alertas.py app.py db_supabase.py` → cero coincidencias en los 5 archivos. La tabla se crea en el `CREATE TABLE` pero no aparece en ningún `.table(...)` de `db_supabase.py` (44 funciones revisadas). | Esto es exactamente lo que CLAUDE.md exige redactar en el Veredicto de Inversión ("stop loss fundamental"). Hoy esa condición se calcula y se muestra una vez, en el momento del análisis, pero nadie la vuelve a comprobar: si meses después la Calidad cae bajo el umbral o el Fair Value se revisa un 15% a la baja, nadie se entera salvo relanzar el análisis a mano. Falsa sensación de "tesis vigilada" cuando solo está documentada una vez. |
| N2 | Baja | `core_indicadores.py` (`adx()`, líneas 60-77) | Patrón análogo al #24 (RSI→NaN en racha alcista pura) pero en ADX: si durante toda la ventana no hay ningún movimiento direccional neto (+DI y −DI ambos exactamente 0, precio perfectamente plano sesión tras sesión), `dx` da `NaN` en vez de un valor neutro, y `es_dato()` lo descarta. Requiere OHLC perfectamente invariable ≥14 sesiones — no observable en mercados reales con tick size > 0. | Reproducido en `python3.13`: histórico sintético con Close/High/Low constantes a 100.0 durante 60 sesiones → `adx()` da `NaN` en toda la cola; con tendencia alcista pura (misma prueba que #24) el ADX sale bien (49-50, sin NaN). | Solo afecta al componente "adx" del Timing en un supuesto (irreal) de cero volatilidad sostenida. Se incluye por completitud, no como acción urgente. |

**Ampliación de alcance del hallazgo #6 (ya abierto; no se abre nuevo id):** #6 solo nombraba `guardar_analisis` y `guardar_multiplos`. Al leer `db_supabase.py` completo (844 líneas): las **44 funciones** del módulo usan el mismo patrón `except Exception: return False/None/[]/0.0` sin loguear tipo ni mensaje de excepción. Incluye funciones que mueven dinero/estado de inversión real: `insertar_operacion` (altas de Cartera real), `ejecutar_nivel_paper` (ejecuciones de Paper Trading), `actualizar_plan_paper`, `eliminar_operacion`, `registrar_decision` (diario de decisiones del comité), `anadir_comparable`/`quitar_comparable` (universo que alimenta Fair Value). `grep -n "except Exception" db_supabase.py` → 44 coincidencias, todas con `pass`/retorno neutro inmediato, sin registrar nada. Si Supabase cambia una política RLS, expira la key o hay fallo de red puntual, toda la persistencia (cartera, paper trading, diario, comparables) falla en silencio y la UI no distingue "vacío real" de "error de escritura" — indistinguible de un bug durante una sesión programada autónoma donde nadie mira la consola. **Se recomienda reclasificar #6 a severidad Alta** dado este alcance real (dinero simulado y real, no solo caché de análisis).

No se encontraron más bugs nuevos reproducibles en el resto de módulos revisados hoy (`core_ponderar.py`, `core_timing.py`, `core_rastreador.py`, `core_interpretar.py`, `core_referencias.py`, `core_paper.py`, `datos_finnhub.py`, `datos_traduccion.py`, `config_secretos.py`, `ui_graficos.py`, `ui_bloque_timing.py`, `ui_bloque_calidad_fv.py`, `ui_vistas_paper.py`, `ui_vistas_rastreador.py`, relectura completa de `core_fair_value.py`/`core_fundamentales.py`/`core_calidad.py`).

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- **Vigilancia activa del stop fundamental (hallazgo N1).** Conectar `core_plan_dca.invalidacion()` a una comprobación real en el cron nocturno. Cada condición ya trae un `umbral` estructurado. Pseudocódigo para `tarea_rastreo.py`:
  ```python
  def comprobar_invalidaciones(plan, analisis_nuevo):
      rotas = []
      for cond in plan.get("invalidacion") or []:
          u = cond["umbral"]
          if "calidad_min" in u and analisis_nuevo["calidad"]["nota"] < u["calidad_min"]:
              rotas.append(cond)
          elif "fair_value_min" in u and analisis_nuevo["fair_value"]["fair_value"] < u["fair_value_min"]:
              rotas.append(cond)
              # ruptura_soporte ya la vigila core_alertas (nivel STOP); no duplicar aquí
      return rotas
  ```
  Cada condición rota se persiste en `tesis_invalidacion` (`activa=false`, `disparada_en=now()`) y genera un nuevo tipo de evento en `core_alertas.py` (p. ej. `tesis:{ticker}:{condicion}:{fecha}`, prioridad alta). Cierra el hueco entre lo que CLAUDE.md exige redactar y lo que la app vigila después.
- **Capital invertido promedio en `roic()`** (`core_fundamentales.py`, línea 57): usa un único balance de cierre, no el promedio inicio/fin de ejercicio. Una recompra o emisión de deuda grande cerca del cierre distorsiona el ROIC anual. Mejora barata: si hay ≥2 columnas de balance, usar la media de patrimonio+deuda-caja de las dos últimas.

### UI / UX
- **Distinguir "sin dato" de "error de escritura"** (ligado a #6 ampliado): añadir `logging.getLogger("stockscanner").warning(...)` dentro de cada `except Exception as e` de `db_supabase.py` (sin cambiar el valor de retorno) para dejar rastro diagnosticable en logs de Streamlit Cloud/GitHub Actions.
- **Badge de "tesis vigilada"** en Paper Trading y Cartera: con N1 resuelto, mostrar en cada ficha (`ui_vistas_paper._ficha`) qué condiciones de invalidación están activas y cuáles se han disparado, en vez de un toggle estático de solo lectura.

### Arquitectura / funcionalidades nuevas de alto valor
1. **Cerrar el círculo de `tesis_invalidacion`** (N1 + mejora de ROIC arriba): la funcionalidad nueva de mayor impacto posible hoy — convierte un campo que ya se calcula y se guarda en vigilancia real, sin nuevas fuentes de datos, solo cableado entre piezas existentes.
2. **Tests automatizados mínimos para `core_*.py`** (refuerza #9, ya abierto): las últimas auditorías han encontrado bugs reales (#26, #27, #28, #29) solo mediante ejecución manual ad-hoc. Formalizar como `tests/` con `pytest` (sin red ni Streamlit) evitaría regresiones. Empezar por `test_core_ponderar.py` y `test_core_fair_value.py` (caso de 2 valores de #28).
3. **Exportar la sesión del comité a PDF/Markdown descargable** desde la propia app: hoy el historial vive solo en el repo Git; un botón en Rastreador/Análisis Individual facilitaría compartir un análisis puntual.
4. **Alerta de concentración sectorial proactiva**: `ui_graficos.grafico_sectores` ya calcula y colorea en naranja los sectores que superan el umbral, pero es solo visual. Añadir un evento de `core_alertas.py` (`concentracion:{sector}:{fecha}`) que avise en el momento de decidir una compra, no solo al abrir el dashboard.

## 3. Alertas para el debate de inversión

> Aplicable a cualquier candidata que el comité analice hoy, de cualquier mercado:

- **#1** (stop sin ATR): si la candidata no tiene histórico suficiente para ATR, el stop técnico puede no respetar el mismo tope de caída máxima. Fijar el stop técnico a mano si el plan marca "sin ATR".
- **#26** (stop "con ATR" tampoco fiable si N3 cae mucho): verificar el riesgo real del plan a mano, no fiarse del `motivo_stop` mostrado, en candidatas con soportes muy dispersos.
- **#27** (pivotes agrupados por cadena, no por centro): revisar visualmente en el gráfico que las zonas de entrada/soporte propuestas tengan sentido, especialmente en small caps o mercados emergentes.
- **#28** (banda de cordura nunca excluye con baja cobertura): si la candidata (REIT, financiera, baja cobertura de analistas) tiene pocos métodos de valoración activos, cruzar el Fair Value con una fuente independiente.
- **#2 / #29** (P/FFO desalineado / antipatrón `or` con ceros): si la candidata es un REIT, verificar el FFO/acción contra una fuente externa antes de aceptar el Fair Value de la app.
- **#19 / #23** (Telegram sin truncar / cron sin credenciales): no afectan al cálculo de hoy, pero si esta es la sesión programada automática, verificar manualmente que la notificación de Telegram de esta sesión llegó.
- **N1 (nuevo):** aunque el Supervisor redacte hoy un stop fundamental correcto para una candidata futura, ningún mecanismo de la app lo volverá a comprobar automáticamente.
- **N3 / #6 ampliado (nuevo alcance):** si esta sesión es la programada autónoma, cualquier fallo de Supabase durante el guardado pasará desapercibido; el Supervisor debe confirmar al revisar el informe que `posiciones_actuales.md`/`historial_decisiones.md` quedaron realmente guardados, no asumirlo solo porque no se vio un error visible.

**Nota de esta sesión (2026-10-12):** ninguna de las tres candidatas investigadas por los analistas llegó a proponerse formalmente al comité (ver Fase 1 y Fase 3 de la sesión), por lo que ninguna de las alertas anteriores se aplicó a un cálculo de plan/stop real hoy. Quedan documentadas para la próxima sesión en que sí haya una candidata seleccionada.
