# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-14
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `fa2f0ed24bc6558d8bfe7f0d0f9e9bc00673b8ff` (sin cambios de código en `stockscanner/app/` desde `48be6c2`, confirmado con `git diff --stat 48be6c2 HEAD -- stockscanner/app`)

## 0. Alcance de esta auditoría

Cobertura ampliada a los módulos que el registro aún no cubría explícitamente: `core_ponderar.py`, `core_timing.py`, `core_rastreador.py`, `core_interpretar.py`, `core_referencias.py`, `datos_finnhub.py`, `datos_traduccion.py`, `config_secretos.py`, `ui_bloque_calidad_fv.py`, `ui_bloque_plan.py`, `ui_bloque_timing.py`, `ui_graficos.py`, `ui_vistas_rastreador.py`, `ui_vistas_paper.py`, `ui_interfaz.py`, `datos_analisis.py`, `ui_vistas_analisis.py` (funciones distintas a `_earnings`), y las funciones restantes de `core_calidad.py`, `core_fair_value.py`, `core_fundamentales.py`, `core_paper.py` y `core_confluencia.py` no cubiertas antes. Se verificaron también las sumas de pesos (`CALIDAD_PESOS`, `FV_PESOS*`, `TIMING_PESOS`, `RASTREADOR_PESOS`) en `config_settings.py` — todas consistentes.

No se encontró ningún bug nuevo de cálculo financiero reproducible. Con el código sin cambios desde la auditoría anterior (2026-10-12) y esta ampliación de cobertura, la confianza en que no quedan bugs de cálculo obvios sin detectar en los motores `core_*` aumenta, aunque **sigue sin haber tests automatizados** (hallazgo #9, abierto) que lo garanticen de forma permanente.

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 32 | Baja | `ui_vistas_analisis.py` (`_formulario()`) | El botón "Analizar" no tiene `disabled=not ticker.strip()`. Si el usuario lo pulsa con el campo de ticker vacío, `datos_analisis.analizar("")` devuelve `None` y la vista muestra `st.error("No se ha podido obtener histórico para «». Revisa el ticker.")` — mensaje confuso con comillas vacías en vez de pedir que se introduzca un ticker | Trazado directo del código (lógica de cadena determinista): `"".strip().upper() == ""` → rama `if not ticker: return None`; otros puntos de entrada de texto libre de la app (ej. "Añadir" en Rastreador) sí aplican `disabled=not nuevos.strip()` | Papercut de UX menor; no afecta a ningún cálculo ni a la integridad de datos |

**Ningún hallazgo previo (1-31) cambia de estado:** cero código modificado desde su fecha de detección.

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- Nada nuevo esta sesión; las prioridades de cálculo siguen siendo #1, #2, #13, #26, #28, #29 (todas abiertas).

### UI / UX
- Fix directo del #32: añadir `disabled=not (pendiente or ticker.strip())` al `st.button("Analizar", ...)` de `ui_vistas_analisis._formulario()`, igual que ya hace el campo de universo del Rastreador.
- Generalizar el patrón de botón deshabilitado por input vacío como norma de estilo del proyecto: es el único punto de entrada de texto libre de toda la app que no lo aplicaba.

### Arquitectura
- Sin hallazgos nuevos esta sesión; prioridades siguen siendo #9 (tests) y #6 (logging de excepciones en `db_supabase.py`).

### Funcionalidades nuevas de alto valor

1. **Ficha de candidata lista para el Comité, generada directamente desde el análisis.** CLAUDE.md exige para cada candidata el formato exacto de `plantilla_analisis_candidata.md` (ROIC/margen/CAGR con periodo y fuente, Fair Value con método y % de descuento, momentum con RS/volumen/niveles). Hoy ese texto lo redacta cada analista a mano a partir de lo que ve en la app, con riesgo de citar una cifra distinta a la que el motor realmente calculó. Propuesta: función `generar_ficha_comite(a: dict) -> str` (en `ui_bloque_plan.py` o una vista nueva) que mapee 1:1 el dict de `datos_analisis.analizar()` a la plantilla, con un `st.text_area` de solo lectura para copiar. Elimina la transcripción manual y garantiza trazabilidad 1:1 entre lo citado en sesión y lo que calculó el motor.

2. **Panel de "Fiabilidad de datos" agregado** (nueva sección en Rastreador/Screener). `fair_value.anomalia`, `fair_value.cobertura`, `calidad.cobertura` y los métodos `"excluido"`/`"recortado"` ya se calculan y persisten por análisis, pero nunca se agregan a nivel de universo. Un panel que consulte `analisis_historico` de los últimos N días y muestre, p. ej., "% de análisis con cobertura de Fair Value < 70%" o "tickers con `anomalia=true` recurrente" daría al comité una foto objetiva de cuánto fiarse de los datos del universo que se debate hoy.

3. **Re-chequeo automático de rechazos previos.** CLAUDE.md exige no reproponer un ticker salvo cambio sustancial documentado, pero hoy ese chequeo depende de que el analista recuerde y relea sesiones anteriores — y de hecho, en esta misma sesión, el Analista 3 volvió a investigar Nintendo (7974.T, ya descartada el 2026-10-12) sin detectarlo hasta el cierre. Con `analisis_historico` ya guardando cada análisis pasado por ticker, se puede construir una función que compare el último análisis guardado contra uno nuevo en vivo y marque automáticamente si ROIC/descuento/nivel técnico cambiaron lo suficiente para justificar una revisión — convirtiendo una regla hoy solo documental en una comprobación mecánica.

4. **Conteo de racha de sorpresas de earnings como señal de Timing.** `_racha()` en `ui_vistas_analisis.py` ya pinta en verde/rojo cada trimestre pasado, pero una racha de ≥3 sorpresas positivas consecutivas (dato ya disponible, solo falta contarlo) es justo el tipo de catalizador que CLAUDE.md pide para el pilar de Timing y hoy no se puntúa en `core_timing.TIMING_PESOS`. Mejora de bajo coste con alto encaje metodológico.

## 3. Alertas para el debate de inversión

No hay ninguna alerta nueva de calidad de datos que afecte a las candidatas de hoy (no hubo candidatas propuestas). Siguen vigentes y aplicables a cualquier candidata futura: el stop-loss "con ATR" puede superar el 25% de riesgo anunciado cuando N3 cae mucho (#26, Alta); la banda de cordura del Fair Value nunca excluye un método disparatado cuando solo hay 2 valores disponibles (#28, Alta); el stop-loss fundamental (invalidación de tesis) se calcula una vez pero nunca se re-comprueba automáticamente (#30, Alta).

---

**Archivos consultados:** `stockscanner/qa/registro_hallazgos.md`, `stockscanner/app/ui_vistas_analisis.py`, `stockscanner/app/core_fair_value.py`, `stockscanner/app/core_plan_dca.py`, módulos listados en la sección 0, `comite_inversion/plantillas/plantilla_analisis_candidata.md`, `comite_inversion/plantillas/plantilla_bug_report.md`.
