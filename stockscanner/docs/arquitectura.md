# StockScanner — Arquitectura

> Documento vivo. Actualizar cuando el Veredicto de Desarrollo (Fase 4) introduzca cambios estructurales.

## Stack

- **Lenguaje:** Python
- **Framework UI:** Streamlit
- **Estado actual:** [pendiente de primera versión de código en `stockscanner/app/`]

## Capas propuestas

- `data/` — obtención y normalización de datos de mercado y fundamentales (fuente, caché, manejo de errores de API).
- `calc/` — funciones de cálculo financiero puras (DCF, múltiplos, ROIC, CAGR, WACC), testeables de forma aislada.
- `ui/` — componentes Streamlit, separados de la lógica de cálculo.
- `qa/` — tests automatizados de las funciones de `calc/` (valores de referencia conocidos).

## Convenciones de rigor de datos

- Toda función de `calc/` debe documentar la fórmula exacta y sus supuestos.
- Todo dato mostrado en `ui/` que sea una estimación (no un hecho reportado) debe marcarse visualmente como tal.
- Los periodos fiscales (TTM, FY, forward) deben quedar siempre explícitos en la UI, nunca implícitos.
