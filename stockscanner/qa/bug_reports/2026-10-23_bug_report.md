# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-23
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `9a3cf9f` (código de `stockscanner/app/` sin cambios desde `48be6c2` — `git log --oneline -- stockscanner/app/` solo muestra ese commit; confirmado por sexta vez consecutiva tras 2026-09-30, 10-14, 10-16, 10-19 y 10-21)

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| — | — | — | Ninguno nuevo. Cobertura ampliada hoy a módulos que ningún hallazgo previo mencionaba explícitamente: `core_ponderar.py`, `core_timing.py`, `core_rastreador.py`, `core_referencias.py`, `core_interpretar.py`, `ui_vistas_paper.py`, `datos_traduccion.py`, `ui_bloque_timing.py`, `ui_bloque_plan.py`, `config_secretos.py` | Lectura íntegra de los 10 módulos; sin bug reproducible nuevo. Estos módulos usan `core_ponderar.es_dato()` de forma consistente, evitando el patrón "dato ausente tratado como cero/falsy" que motivó varios hallazgos anteriores (#21, #24, #29, #31, #33) | Ninguno — confirma robustez, no introduce hallazgo |

`ui_bloque_calidad_fv.py`, `ui_estilos.py`, `ui_graficos.py` y `sql_migracion_002/003/004.sql` quedan pendientes de una lectura línea a línea en una próxima sesión (todavía no cubiertos explícitamente en ninguna auditoría).

## 2. Mejoras propuestas

### Arquitectura — cerrar el bucle entre el Comité y el motor de backtest ya existente

`core_rastreador.py` ya implementa un motor de backtest completo (`evaluar_senales()`, `filas_backtest()`, `resumen_backtest()`) pensado para los análisis que guarda la propia app (`analisis_historico`). Pero las decisiones reales del Comité (`comite_inversion/cartera/historial_decisiones.md`) viven en Markdown libre, fuera de ese circuito — el motor no puede evaluarlas tal cual.

Propuesta concreta: un importador que traduzca cada entrada de `historial_decisiones.md` a una fila de `analisis_historico`/`backtest_resultados` y reutilice el motor sin tocarlo:

```python
# tarea_importar_comite.py (nuevo, fuera de core_* para no romper la pureza)
def importar_decision(ticker, fecha_entrada, precio_entrada, motor_version="comite-v1"):
    fila = {"ticker": ticker, "fecha_analisis": fecha_entrada, "precio": precio_entrada,
            "veredicto": "ENTRAR", "senal_timing": None, "motor_version": motor_version}
    db_supabase.insertar_analisis_historico(fila)  # requiere nueva función en db_supabase.py

# en tarea_rastreo.py o un comando aparte, una vez por sesión del comité:
detalle, resumenes = core_rastreador.evaluar_senales(
    db_supabase.listar_analisis_historico(motor_version="comite-v1"), cierres, benchmark)
```

Esto da al Supervisor, por primera vez, un track record cuantitativo verificable de sus propias decisiones (MU, V, AEM, ACN) frente al benchmark, con el motor que ya existe — sin inventar nada nuevo. (Primera propuesta en el registro de hallazgos del 2026-10-21; aquí se entrega con pseudocódigo concreto para su implementación.)

### Cálculo — coerción numérica segura centralizada (profundización del hallazgo #33)

`core_ponderar.es_dato()` ya resuelve el problema de "ausencia" en todos los motores `core_*`, pero el patrón inseguro que causó el hallazgo #33 (`int(x or 0)` con `NaN` truthy) vive en el parseo de fuentes externas (`datos_yfinance.py`, `datos_finnhub.py`), no en los motores. Propuesta: una función hermana de `es_dato()` en `core_ponderar.py` para ese punto de entrada:

```python
def entero_seguro(valor, defecto: int = 0) -> int:
    """int() que nunca lanza con NaN/None/str no numérica; usar en TODO
    parseo de respuesta externa antes de guardar o sumar."""
    return int(float(valor)) if es_dato(valor) else defecto
```

y sustituir cada `int(fila.get(k) or 0)` de `datos_yfinance.obtener_recomendaciones_yf()` / `datos_finnhub._recomendaciones()` por `entero_seguro(fila.get(k))`.

### UI / UX
- Sin propuesta nueva hoy.

## 3. Alertas para el debate de inversión

Ninguna candidata se propuso formalmente hoy (los tres analistas descartaron todas las que investigaron), por lo que no hay alerta de datos específica de esta sesión. Alerta estructural ya conocida (hallazgo #30, Abierto desde 2026-10-12) que el Supervisor debe seguir teniendo presente para las posiciones vivas de cartera (MU, V, AEM, ACN): el stop loss fundamental de cada una se calculó una vez y nunca se vuelve a comprobar contra datos nuevos — ningún cron lo revisa.
