# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-10-05
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2` (sin cambios desde la estructura inicial del repo; confirmado con `git log --oneline -- stockscanner/app` y `git status --short`)

**Módulos revisados en esta auditoría:** `core_confluencia.py`, `core_interpretar.py`, `core_ponderar.py`, `core_rastreador.py`, `core_referencias.py`, `core_timing.py`, `core_fair_value.py`, `core_alertas.py` (recontraste), `core_cartera.py` (skim), `datos_finnhub.py`, `datos_traduccion.py`, `config_secretos.py`, `ui_bloque_calidad_fv.py`, `ui_bloque_plan.py`, `ui_bloque_timing.py`, `ui_graficos.py`, `ui_vistas_paper.py`, `ui_vistas_rastreador.py`, `ui_estilos.py`/`ui_interfaz.py` (skim).

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| 27 | Alta | `core_confluencia.py` (`_pivotes_agrupados()`, líneas 69-91) | El agrupador de pivotes compara cada nuevo pivote solo contra el **último** elemento ya añadido al grupo (`abs(precio - grupos[-1][-1][1]) <= tolerancia`), no contra el primero ni contra la media del grupo — "chain clustering". Una secuencia de pivotes cada uno a menos de `tolerancia` del anterior se fusiona en un único grupo aunque el primero y el último disten mucho más que `tolerancia`, violando el principio de diseño que el propio docstring del módulo enuncia ("dos niveles a 0,3 ATR se funden solos, a 3 ATR no"). | Ejecución real con 15 pivotes sintéticos espaciados 0,9 (tolerancia=1,0): todos se fusionan en un solo grupo con centro en 106,3, pese a que el primer pivote (100,0) y el último (112,6) distan 12,6 — 12,6x la tolerancia nominal. El grupo recibe además un peso reforzado por "15 toques" (`1 + 0,35·ln(15) ≈ 1,95x` el peso base de un pivote diario), cuando en realidad son 15 niveles técnicos distintos agrupados por error. | `candidatos()`/`zonas()` alimentan directamente `core_plan_dca.plan()`: las entradas E1/E2/E3 y salidas S1/S2/S3 mostradas al usuario en el Bloque 6 (`ui_bloque_plan.py`) pueden anclarse a un centro de masa ficticio que no corresponde a ningún soporte/resistencia real, especialmente en históricos largos (10 años de pivotes semanales) con tendencias sostenidas. |

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- Corregir `_pivotes_agrupados()` (hallazgo #27) comparando cada candidato contra el **centro/media del grupo** (o contra su primer elemento) en vez de contra el último, para que la anchura máxima de cualquier grupo quede acotada por `tolerancia` sin importar cuántos pivotes se acumulen en cadena.

### UI / UX
- En `ui_bloque_plan.py` / `ui_bloque_timing.py`, cuando una zona de confluencia agrega muchos "toques" (`n >= 8-10`), mostrar junto al motivo el rango de precios real de los componentes agrupados (hoy solo se ve el precio central y el conteo), para detectar a simple vista un caso como el del hallazgo #27 antes de operarlo.

### Arquitectura / Funcionalidad nueva de alto impacto
- **Señal de ruptura de zona de confluencia (breakout/breakdown) como componente de Timing.** CLAUDE.md exige para el pilar de Momentum "ruptura de resistencias... o zona técnica idónea de entrada", pero ningún componente de `core_timing.TIMING_PESOS` lo puntúa hoy. Pseudocódigo:
  ```python
  def ruptura(df, zonas_conf, ind):
      precio_ayer, precio_hoy = df["Close"].iloc[-2], df["Close"].iloc[-1]
      for z in zonas_conf:
          if not z["fuerte"]:
              continue
          cruzo_arriba = precio_ayer <= z["precio"] < precio_hoy
          cruzo_abajo  = precio_ayer >= z["precio"] > precio_hoy
          if (cruzo_arriba or cruzo_abajo) and es_dato(ind.get("volumen_relativo")) and ind["volumen_relativo"] >= 1.3:
              return {"direccion": "alcista" if cruzo_arriba else "bajista",
                      "zona": z["precio"], "motivos": z["motivos"]}
      return None
  ```
  Se añadiría a `TIMING_PESOS`/`TIMING_TRAMOS` como componente nuevo (5-8 puntos redistribuidos desde componentes afines). Debe implementarse después de corregir el hallazgo #27, para no construir la señal sobre zonas con agrupación defectuosa.
- **Señal de compra institucional/insider** (`datos_finnhub.obtener_transacciones_insider`, endpoint gratuito `/stock/insider-transactions`). El Momentum del comité exige explícitamente "volumen inusual de compra institucional", y hoy no existe ningún módulo que lo consulte. Alimentaría un nuevo componente de Timing (compra neta de insiders en N meses, normalizada por capitalización) con la misma mecánica de `ponderar()`/exclusión por falta de dato que ya usa el resto del motor.

## 3. Alertas para el debate de inversión

- El hallazgo #27 (agrupación en cadena de pivotes) afecta a cualquier candidata de hoy cuyo plan DCA dependa de zonas construidas con muchos pivotes semanales/diarios encadenados en tendencias largas (p. ej. acciones con varios años de historial en un canal regular): los niveles E1/E2/E3 o S1/S2/S3 que mostraría la app podrían anclarse a un precio sin soporte/resistencia real individual. Si el Supervisor usa los niveles de la app para fijar entrada/stop de una candidata de hoy, debe cruzarlos visualmente contra el gráfico de velas.
- Hallazgos ya abiertos directamente relevantes para las candidatas de hoy: **#7/#10** (conversión de divisas limitada a EUR/USD en toda la UI) afecta a 105560.KS (KB Financial Group), que cotiza en KRW — mostraría "EUR n/d". **#20** (la lista blanca de sufijos bursátiles no cubre mercados asiáticos, cita explícitamente `.KS`) afecta directamente al mismo ticker: `105560.KS` podría no normalizarse correctamente en StockScanner.
- El hallazgo #26 (stop con ATR puede violar el tope del 25%) sigue abierto y sin corregir; aplica a cualquier plan DCA que se genere hoy para las posiciones ya en cartera.

## 4. Hallazgos previos a reconfirmar o cerrar

- Ninguno: código sin cambios desde `48be6c2`; los 26 hallazgos previos siguen abiertos tal como están registrados.
