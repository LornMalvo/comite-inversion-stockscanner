# Bug Report & Feature Request — StockScanner

**Fecha:** 2026-11-04
**Agente:** Beta Tester & QA (Agente 4)
**Versión/commit de la app auditada:** `48be6c2` (sin cambios en `stockscanner/app/` desde 2026-09-19, confirmado vía `git log --oneline -- stockscanner/app/`; HEAD actual del repositorio: `3e6f4c0`, que no toca ese directorio) — **undécima confirmación consecutiva**

## 1. Bugs y discrepancias de datos

| # | Severidad | Módulo | Descripción | Evidencia | Impacto |
|---|---|---|---|---|---|
| — | — | — | Ninguno nuevo. Los 36 hallazgos del registro (`registro_hallazgos.md`) siguen en el mismo estado ("Abierto") que en la última revisión; no se reporta ninguno como nuevo, conforme a la regla de no-repetición de CLAUDE.md. | `git log --oneline -- stockscanner/app/` → un único commit (`48be6c2`) en toda la historia de ese directorio | N/A |

## 2. Mejoras propuestas

### Cálculo / lógica financiera
- Ninguna nueva hoy (sin cambios de código desde la última pasada).

### UI / UX
- Ninguna nueva hoy.

### Arquitectura

**Migrar `comite_inversion/cartera/empresas_analizadas.md` a una tabla Supabase consultable.**

Problema verificado hoy mismo, en vivo, durante la propia sesión: el registro ya tiene 295 líneas (~52.000 tokens) y sigue creciendo una fila por ticker cada sesión. Al aplicar la regla de no-repetición de hoy, la lectura completa del archivo excedió el límite de una sola llamada de herramienta y tuvo que partirse en tres lecturas sucesivas. No es un caso límite teórico: es el cuello de botella operativo real del propio comité, y empeora de forma monótona sesión tras sesión, sin fin de vida del archivo (a diferencia de `registro_hallazgos.md`, que crece mucho más despacio por tener una granularidad de hallazgo, no de ticker).

**Propuesta:** aprovechar que `db_supabase.py` ya tiene la infraestructura (tablas como `diario_decisiones`/`analisis_historico` y el patrón de upsert/consulta puntual que usa el resto del módulo) para añadir:

```python
# db_supabase.py

def ya_analizada(ticker: str) -> dict | None:
    """Devuelve la última fila de exclusión para `ticker`, o None si nunca se trató."""
    cli = _cliente()
    if cli is None:
        return None
    try:
        r = (cli.table("empresas_analizadas")
                .select("*").eq("ticker", ticker)
                .order("fecha_sesion", desc=True).limit(1).execute())
        return r.data[0] if r.data else None
    except Exception:
        return None


def registrar_empresa_analizada(ticker: str, fecha_sesion: str, resultado: str,
                                 tesis_resumida: str, que_cambiaria: str,
                                 sesion_url: str) -> bool:
    cli = _cliente()
    if cli is None:
        return False
    try:
        cli.table("empresas_analizadas").upsert({
            "ticker": ticker, "fecha_sesion": fecha_sesion, "resultado": resultado,
            "tesis_resumida": tesis_resumida, "que_cambiaria": que_cambiaria,
            "sesion_url": sesion_url,
        }, on_conflict="ticker,fecha_sesion").execute()
        return True
    except Exception:
        return False
```

Migración nueva (`sql_migracion_005.sql`), con índice en `ticker` para que la consulta de no-repetición sea O(1) por candidata en vez de leer ~300 líneas de Markdown:

```sql
create table if not exists empresas_analizadas (
    id bigint generated always as identity primary key,
    ticker text not null,
    fecha_sesion date not null,
    resultado text not null,
    tesis_resumida text,
    que_cambiaria text,
    sesion_url text,
    created_at timestamptz default now()
);
create index if not exists idx_empresas_analizadas_ticker on empresas_analizadas (ticker);
```

El Supervisor consultaría `ya_analizada(ticker)` por cada candidata propuesta en la Fase 1 (una fila, no 295 líneas de Markdown) antes de aceptarla. El Markdown se conserva como vista legible para humanos — generada/exportada desde la tabla al cierre de sesión — pero deja de ser la fuente que el propio comité tiene que leer íntegra para cumplir su propia regla de no-repetición. Nota de rigor: estas dos funciones heredan el patrón `except Exception: return None/False` ya usado en las 44 funciones de `db_supabase.py`, por lo que siguen sujetas al hallazgo #6 (fallos de Supabase descartados silenciosamente sin loguear tipo ni mensaje) — su corrección no es parte de esta propuesta, pero conviene recordarlo antes de implementar.

Mismo patrón de crecimiento lineal sin fin de vida aplicaría a futuro a `registro_hallazgos.md` si su ritmo de crecimiento se acelerase, aunque hoy ese archivo sigue siendo manejable en una sola lectura.

## 3. Alertas para el debate de inversión

- Ninguna candidata se propuso en la Fase 1 de hoy (17 tickers investigados y descartados por los tres analistas, sin llegar a Fase 3), por lo que no hay ninguna alerta de calidad de datos que integrar en el debate.
- Recordatorio de vigencia: los hallazgos #26 (stop ATR puede superar el riesgo anunciado), #28 (banda de cordura nunca excluye un método disparatado con solo 2 valores de cobertura) y #30 (el stop fundamental no se re-comprueba tras la decisión inicial) siguen abiertos y aplicarían a cualquier candidata futura que llegue a comprarse.
