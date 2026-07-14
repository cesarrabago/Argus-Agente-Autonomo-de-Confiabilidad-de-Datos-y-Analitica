<div align="center">

# 🛰️ Argus — Agente Autónomo de Confiabilidad de Datos y Analítica

### Plataforma de datos agéntica que monitorea pipelines, se autodiagnostica ante incidentes y responde preguntas de negocio en lenguaje natural — construida sobre dbt, BigQuery y OpenClaw/Claude.

<br>

![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=for-the-badge&logo=python&logoColor=white)
![dbt](https://img.shields.io/badge/dbt-Transform%20%2B%20Tests-FF694B?style=for-the-badge&logo=dbt&logoColor=white)
![BigQuery](https://img.shields.io/badge/BigQuery-Warehouse-669DF6?style=for-the-badge&logo=googlebigquery&logoColor=white)
![OpenClaw](https://img.shields.io/badge/OpenClaw-Agent%20Gateway-E4572E?style=for-the-badge)
![Claude](https://img.shields.io/badge/Claude-claude--cli-D97757?style=for-the-badge&logo=anthropic&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-Dashboards-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/CI%2FCD-Eval%20Gated-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)

<br>

> **Proof of Concept** — Argus simula el stack de confiabilidad de datos que una empresa data-driven necesita para pasar de apagar incendios de forma *reactiva* ("¿por qué cambió este número del dashboard?") a pipelines *proactivos* que se autodiagnostican, más una capa de analítica en lenguaje natural gobernada. Todo corre sobre datos sintéticos; no se usa información real de ninguna empresa.

<br>

[🎯 Problema](#-el-problema) · [💡 Solución](#-la-solución) · [📐 Arquitectura](#-arquitectura) · [🤖 El Agente](#-la-capa-de-agente) · [🧱 Plataforma de Datos](#-plataforma-de-datos-dbt--bigquery) · [🧪 Evals](#-harness-de-evaluación) · [🛡️ Guardrails](#️-guardrails--seguridad) · [📈 Observabilidad](#-observabilidad--costo) · [🚀 Quick Start](#-quick-start) · [🩺 Decisiones de Diseño](#-decisiones-de-diseño--tradeoffs) · [📁 Estructura](#-estructura-del-proyecto)

</div>

---

## 🎯 El Problema

Toda empresa que opera sobre datos choca con los mismos dos modos de falla a medida que escala — y ninguno se resuelve agregando más dashboards:

| Síntoma | Impacto real |
|---|---|
| Los pipelines se rompen **en silencio** | Un KPI cambia y nadie sabe por qué hasta que un stakeholder se queja |
| El análisis de causa raíz es **manual** | Los analistas de guardia queman horas escribiendo queries ad-hoc para rastrear un número malo |
| Los checks de calidad viven **dentro de un script** | Sin lineage, sin historial, sin gate — el dato malo llega al warehouse |
| Los analistas son un **cuello de botella** para preguntas | "¿Por qué bajaron las ventas en MX la semana pasada?" espera en una fila |
| Los demos de LLM-sobre-SQL **alucinan joins** | Impresionan en la demo, fallan en producción, nadie confía en ellos |
| Sin visibilidad de **costo / tokens** en la capa de IA | Un agente que responde preguntas también puede quemar el presupuesto en silencio |

**La consecuencia:** respuesta lenta a incidentes, un equipo de analítica saturado, y una capa de IA en la que nadie confía porque no está gobernada, medida ni observada.

---

## 💡 La Solución

Una plataforma de datos gobernada con un agente autónomo encima. Los datos fluyen en una sola dirección; el agente lee la capa *gobernada*, nunca tablas crudas:

```
Datos sintéticos multicanal  →  dbt (staging → marts + tests + lineage)  →  BigQuery
                                          │
                                 Capa semántica (métricas gobernadas)
                                          │
                        ┌─────────────────┴─────────────────┐
                        │        OpenClaw + Claude           │
                        │         (claude-cli, WSL)          │
                        └─────────────────┬─────────────────┘
                 ┌───────────────┬────────┴────────┬────────────────┐
        Monitoreo autónomo       Text-to-SQL         Digest ejecutivo
        autodiagnóstico → Jira   en Slack            brief programado
             + alerta Slack      (anclado a semántica) → Slack
```

1. **Modela** el warehouse con dbt — staging, marts, tests, freshness y lineage como ciudadanos de primera clase.
2. **Gobierna** las métricas en una capa semántica para que el agente responda desde métricas *definidas*, no SQL adivinado.
3. **Vigila** los pipelines de forma programada; cuando un check falla, el agente investiga, redacta un diagnóstico en lenguaje llano, abre un ticket en Jira y avisa en Slack.
4. **Responde** preguntas de negocio en Slack vía text-to-SQL, anclado a la capa semántica y cercado por los guardrails de SQL.
5. **Resume** el día/semana en un brief ejecutivo, de forma automática.
6. **Mide todo** — la precisión del text-to-SQL vía un harness de evals, y tokens/costo/latencia vía OpenTelemetry.

---

## 📐 Arquitectura

![Arquitectura de Argus](./Img_Arq.png)

Tres capas, cada una con una sola responsabilidad:

**Plataforma de datos (la fuente de verdad).** Generadores sintéticos emiten eventos multicanal (órdenes, tickets, interacciones) hacia BigQuery. dbt transforma `raw → staging → marts`, y publica `tests`, `source freshness` y `exposures` que apuntan a los dashboards de Power BI. Esta capa es determinista y totalmente testeable — el agente nunca inventa datos, lee tablas modeladas.

**Capa de agente (OpenClaw + Claude).** El gateway de OpenClaw corre Claude vía `claude-cli` y expone tres capacidades. Alcanza el warehouse y las herramientas externas (Jira, Slack) a través de servidores MCP, lee métricas gobernadas de la capa semántica, y corre sus monitores en las tareas programadas de OpenClaw.

**Ingeniería transversal.** Los guardrails, un harness de evals que actúa como gate del CI/CD, y la observabilidad de costo/traces envuelven ambas capas. Esta es la parte que convierte un demo en un sistema.

---

## 🤖 La Capa de Agente

### Capacidad 1 — Observabilidad Autónoma de Datos

Corre de forma programada (cron de OpenClaw). Cada ciclo ejecuta la suite de calidad de datos y, ante una falla, el agente **investiga por su cuenta** — lanza queries de seguimiento para localizar la causa (¿qué fuente? ¿qué día? ¿qué segmento?), redacta un diagnóstico legible, abre un ticket en Jira y publica una alerta en Slack con el resumen y un enlace a la corrida.

```
Detectado:  breach de freshness en `mart_orders` — 0 filas cargadas para 2026-07-14
Diagnóstico: `raw_orders_web` aterrizó por última vez a las 03:00; falta el batch de las 04:00.
             El volumen del canal web es 0 vs. una mediana de 7 días de ~1,400.
Acción:     Jira DATA-142 abierto (P2) · Slack #data-alerts notificado
```

Checks implementados: freshness, anomalías de volumen de filas, drift de esquema, picos de tasa de nulos, y detección de anomalías en KPIs (estadística + basada en modelo, reutilizando el tooling de anomalías de trabajo previo).

### Capacidad 2 — Analítica Conversacional (Text-to-SQL en Slack)

Cualquiera hace una pregunta en lenguaje natural; Argus genera el SQL, lo ejecuta y devuelve un gráfico más un insight escrito. Clave: la generación está **anclada a la capa semántica** — al agente se le entregan las definiciones gobernadas de métricas y dimensiones, así compone a partir de métricas conocidas en vez de alucinar joins.

```
@argus ¿por qué bajaron las órdenes en MX la semana pasada?
→ Las órdenes de MX cayeron 12.3% WoW (18,204 → 15,960). La caída se concentra
  en el canal "web" (-31%), mientras app y POS quedaron planos. Causa upstream
  probable: el hueco de ingesta web reportado en DATA-142. [gráfico adjunto]
```

Cada query generada pasa por los [guardrails de SQL](#️-guardrails--seguridad) antes de ejecutarse, y cada respuesta es evaluada por el [harness de evals](#-harness-de-evaluación).

### Capacidad 3 — Digest Ejecutivo

Un brief programado (diario/semanal) que convierte KPIs y cambios relevantes en narrativa orientada a decisiones, con enlaces a los dashboards de Power BI. Mismo motor que la Capacidad 2, corrido headless en un temporizador.

### Cómo mapea a OpenClaw

| Concern | Primitiva de OpenClaw |
|---|---|
| Motor de razonamiento | Claude vía `claude-cli` (`agents.defaults.cliBackends.claude-cli`) |
| Monitores programados | Cron / tareas programadas |
| Interfaz | Canal de Slack |
| Acceso a warehouse / Jira / Slack | Servidores MCP |
| Acciones reutilizables | Skills (run-quality-suite, text-to-sql, chart) |
| Memoria de incidentes y baselines | Memoria del agente |
| Traces y costo | Export de OpenTelemetry (nativo) |

---

## 🧱 Plataforma de Datos (dbt + BigQuery)

El warehouse está modelado, no volcado. El layering sigue las convenciones de dbt:

```
models/
├── staging/      un modelo por fuente, tipado/renombrado ligero, 1:1 con raw
├── intermediate/ lógica de negocio reutilizable, no expuesta
└── marts/        hechos y dims listos para análisis, documentados, testeados, expuestos
```

La calidad de datos vive **en los modelos**, versionada y con gate en CI — no en un script imperativo:

```yaml
# models/marts/_marts.yml
models:
  - name: mart_orders
    description: "Una fila por orden, enriquecida con canal y campos de SLA."
    columns:
      - name: order_id
        tests: [unique, not_null]
      - name: channel
        tests:
          - accepted_values:
              values: ['web', 'app', 'pos', 'email', 'chat', 'social']
      - name: revenue
        tests:
          - dbt_utils.accepted_range: { min_value: 0 }

sources:
  - name: raw
    freshness:
      warn_after:  { count: 6,  period: hour }
      error_after: { count: 12, period: hour }
    tables:
      - name: raw_orders_web
        loaded_at_field: _ingested_at
```

`dbt build` corre modelos **y** tests en orden de dependencias; `dbt source freshness` alimenta el monitor de frescura; los `exposures` conectan el DAG con los reportes de Power BI para que el lineage sea de punta a punta.

---

## 🧮 Capa Semántica

Las métricas se definen una sola vez, en control de versiones, y el agente compone a partir de ellas — esto es lo que evita que el text-to-SQL alucine. Las definiciones se le exponen al agente como su contexto de anclaje.

```yaml
# semantic/metrics.yml
metrics:
  - name: orders
    label: "Órdenes"
    calculation: "count(order_id)"
    grain: [date, channel, country]
  - name: revenue
    label: "Ingresos"
    calculation: "sum(revenue)"
    grain: [date, channel, country]
  - name: cancellation_rate
    label: "Tasa de cancelación"
    calculation: "countif(status = 'cancelled') / count(order_id)"
    grain: [date, channel, country]
```

Cuando un usuario hace una pregunta, Argus recibe estas definiciones y las dimensiones disponibles, y genera SQL que referencia solo métricas gobernadas — no aritmética arbitraria de columnas.

---

## 🧪 Harness de Evaluación

Un feature de LLM que no puedes medir es un pasivo. Argus incluye una suite de evals de text-to-SQL evaluada por **precisión de ejecución** (¿coinciden los result sets?) — no por coincidencia de strings del SQL, que penalizaría reescrituras válidas.

```yaml
# evals/cases.yml
- id: mx_orders_wow
  question: "¿Cuántas órdenes en México la semana pasada vs la semana previa?"
  expected_sql_result:
    reference: "sql/references/mx_orders_wow.sql"   # query de referencia confiable
  assert:
    - type: result_set_equals    # corre el SQL generado, compara filas contra la referencia
    - type: row_count_gt: 0
- id: cancellation_by_channel
  question: "¿Qué canal tuvo la mayor tasa de cancelación en junio?"
  expected_answer_contains: ["web"]
  assert:
    - type: valid_sql            # parsea + es un único SELECT
    - type: result_set_equals
```

El runner ejecuta cada query generada contra un warehouse de fixtures semilla, compara el result set contra la referencia confiable, y reporta una **precisión** por suite más un desglose de fallas (join malo, filtro incorrecto, columna alucinada). Esta suite es un **gate obligatorio de CI** (abajo).

> **Precisión actual de evals:** `_TBD — se llena tras la primera corrida de evals_` · Target: **≥ 90%** de precisión de ejecución en la suite semilla.

---

## 🛡️ Guardrails & Seguridad

El agente habla con el warehouse a través de una cuenta de servicio **solo lectura**, y cada query generada se parsea y valida *antes* de ejecutarse:

```python
# guardrails/sql.py  — rechaza cualquier cosa que no sea un SELECT acotado
import sqlglot
from sqlglot import exp

BLOCKED = (exp.Insert, exp.Update, exp.Delete, exp.Drop, exp.Create, exp.Alter, exp.Merge)
PII_COLUMNS = {"email", "phone", "full_name", "customer_name"}

def validate(sql: str, max_rows: int = 10_000) -> str:
    tree = sqlglot.parse_one(sql, read="bigquery")
    if any(isinstance(n, BLOCKED) for n in tree.walk()):
        raise ValueError("Solo se permiten sentencias SELECT de solo lectura.")
    if tree.find(exp.Star):
        raise ValueError("SELECT * no está permitido — nombra tus columnas.")
    if any(c.name.lower() in PII_COLUMNS for c in tree.find_all(exp.Column)):
        raise ValueError("La query toca columnas con PII.")
    if not tree.args.get("limit"):
        tree = tree.limit(max_rows)               # fuerza un límite
    return tree.sql(dialect="bigquery")
```

Defensa en profundidad:

- **IAM de solo lectura** — la cuenta de servicio del agente tiene `roles/bigquery.dataViewer` + `jobUser`, nada que pueda escribir.
- **Cap de bytes facturados** — cada job fija `maximum_bytes_billed` para que una query mala no pueda escanear (ni facturar) todo el warehouse.
- **Redacción de PII** — la config de redacción de OpenClaw enmascara campos sensibles en logs, transcripts y payloads de herramientas.
- **Postura anti prompt-injection** — el texto no confiable (cuerpos de tickets, comentarios) se trata como dato, nunca como instrucciones para el agente.

---

## 📈 Observabilidad & Costo

El agente está instrumentado como un servicio de producción, no como una caja negra:

- **Tracing** — el export de OpenTelemetry (nativo en OpenClaw) envía spans por cada corrida del agente: prompt, tool calls, SQL ejecutado, latencia.
- **Contabilidad de tokens y costo** — cada corrida registra tokens de entrada/salida y costo estimado; un reporte acumulado señala patrones de preguntas caras.
- **Historial de corridas** — cada monitor y pregunta se persiste con su resultado, para que incidentes y respuestas sean auditables después.

> **Costo por pregunta respondida (p50):** `_TBD — medido tras las primeras corridas_`

---

## 🚀 Quick Start

### Prerrequisitos

```bash
Python 3.11+
Docker & Docker Compose
Un proyecto de BigQuery (el tier sandbox sirve) + una llave de service-account solo lectura
Gateway de OpenClaw corriendo con claude-cli configurado (ver docs/openclaw-setup.md)
Power BI Desktop (opcional — para construir los dashboards .pbix)
```

### Setup

```bash
# 1. Clonar
git clone https://github.com/cesarrabago/argus-data-agent
cd argus-data-agent

# 2. Variables de entorno
cp .env.example .env          # proyecto BigQuery, dataset, ruta de la SA key, tokens de Slack/Jira

# 3. Infra local (warehouse de fixtures para evals + servicios de soporte)
docker-compose up -d

# 4. Dependencias de Python
pip install -r requirements.txt
```

### Sembrar datos y construir el warehouse

```bash
# Generar datos sintéticos multicanal → BigQuery raw
python data/synthetic/generate_data.py --days 90

# Construir modelos + correr todos los tests (esto es lo que corre CI)
dbt build
dbt source freshness
```

### Apuntar OpenClaw al agente

```bash
# Backend de Claude CLI (ruta absoluta para evitar problemas de PATH bajo systemd/WSL)
openclaw config set agents.defaults.cliBackends \
  '{"claude-cli":{"command":"/home/crabago/.local/bin/claude"}}' --strict-json --merge

# Registrar los skills + servidores MCP de Argus, luego reiniciar el gateway
openclaw config set skills.allow '["run-quality-suite","text-to-sql","chart"]' --strict-json --merge
systemctl --user restart openclaw-gateway.service
```

### Correr las piezas

```bash
# Ciclo de monitoreo autónomo (también corre en el cron de OpenClaw)
python -m argus.monitors.run

# Text-to-SQL: preguntar desde la CLI (replica lo que llama el canal de Slack)
python -m argus.ask "¿por qué bajaron las órdenes en MX la semana pasada?"

# Suite de evaluación
python -m argus.evals.run          # imprime la precisión de ejecución por suite
```

---

## 🩺 Decisiones de Diseño & Tradeoffs

*Lo senior no es la lista de features — es por qué cada pieza está construida como está.*

- **`claude-cli` por ruta absoluta, no por resolución de PATH.** OpenClaw spawnea `claude` desde un servicio de usuario systemd cuyo entorno está recortado. Depender del `PATH` es frágil (y, en WSL, activamente peligroso — un `claude.exe` de Windows puede colarse vía `appendWindowsPath` y fallar al ejecutarse bajo systemd). Argus fija el backend a la ruta absoluta del binario nativo de Linux, eliminando el PATH de la superficie de falla por completo.

- **Capa semántica en vez de "que el LLM escriba SQL libremente".** El text-to-SQL sin anclaje se ve hermoso en demo y falla en producción — inventa joins y aritmética de columnas. Entregarle al modelo *definiciones gobernadas de métricas* cambia un poco de flexibilidad por corrección y confianza. El harness de evals existe para probar que ese tradeoff valió la pena, no para asumirlo.

- **Precisión de ejecución, no coincidencia de string del SQL, como métrica de eval.** Dos queries distintas pueden ser igualmente correctas. Evaluar según si los *result sets* coinciden premia la corrección en vez de penalizar reescrituras válidas — y es la métrica que de verdad mapea a "¿el usuario recibió la respuesta correcta?".

- **Solo lectura por construcción, no por política.** La capa de guardrails es real, pero el control primario es que el rol IAM del agente *no puede* escribir. La seguridad que depende de que el modelo se porte bien no es seguridad.

- **Tests de dbt como gate de CI, no como reporte nocturno.** Atrapar dato malo después de que se envió es respuesta a incidentes; atraparlo en el PR es ingeniería. Los tests corren en cada cambio y bloquean el merge.

- **"No ayudó" es un resultado válido y documentado.** Donde una feature candidata o un ajuste de modelo no muestra mejora medible, se queda fuera — y el resultado negativo se escribe. (Principio heredado de trabajo previo, donde adiciones de features que sonaban razonables movieron una métrica menos que el ruido.)

---

## ⚠️ Limitaciones Conocidas

- Corre completamente sobre **datos sintéticos**; las distribuciones son plausibles pero no reales, así que los baselines de anomalías son ilustrativos.
- El **tier sandbox de BigQuery** tiene límites de cuota/retención; un despliegue productivo usaría un proyecto estándar.
- La suite de evals cubre el set de preguntas semilla — ampliar la cobertura es continuo, y la precisión reportada es tan representativa como ese set.
- Las métricas marcadas `_TBD_` se llenan de corridas reales; los targets se declaran por separado y **no** son resultados medidos.

---

## 📦 Tech Stack

| Capa | Tecnología | Propósito |
|---|---|---|
| Lenguaje | Python 3.11 | Generadores, glue del agente, evals, guardrails |
| Warehouse | Google BigQuery | Warehouse en la nube (amigable con sandbox) |
| Transformación | dbt (core) | Modelos, tests, freshness, lineage, exposures |
| Capa semántica | dbt metrics / registro YAML | Definiciones gobernadas de métricas para anclaje |
| Gateway de agente | OpenClaw | Scheduling, canales, MCP, memoria |
| Razonamiento | Anthropic Claude (`claude-cli`) | Diagnóstico, text-to-SQL, narrativa |
| Interfaz | Slack | Hacer preguntas, recibir alertas y digests |
| Ticketing | Jira (MCP) | Tickets de incidente abiertos automáticamente |
| Guardrails de SQL | sqlglot | Parsea/valida cada query generada |
| Detección de anomalías | scikit-learn / XGBoost | Monitores de KPI estadísticos + basados en modelo |
| BI | Power BI Desktop | Dashboards ejecutivos (exposures) |
| Contenedores | Docker Compose | Entorno local reproducible |
| CI/CD | GitHub Actions | dbt build + gate de evals + lint en cada PR |
| Observabilidad | OpenTelemetry | Traces, contabilidad de tokens/costo |
| Lint/format | ruff + sqlfluff | Gates de estilo de Python + SQL |

---

## 🔁 CI/CD

Cada pull request tiene que pasar, o no se mergea:

```yaml
# .github/workflows/ci.yml
name: ci
on: [pull_request]
jobs:
  build-and-eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -r requirements.txt
      - run: ruff check . && sqlfluff lint models/
      - run: dbt build --target ci          # modelos + todos los tests de datos
      - run: python -m argus.evals.run --min-accuracy 0.90   # gate en precisión de text-to-SQL
```

El paso de evals **falla el build** si la precisión del text-to-SQL cae por debajo del umbral — el feature de IA se trata con el mismo rigor que los modelos de datos.

---

## 📁 Estructura del Proyecto

```
argus-data-agent/
│
├── 📄 README.md
├── 📄 Img_Arq.png                    ← diagrama de arquitectura referenciado arriba
├── 📄 docker-compose.yaml
├── 📄 requirements.txt
├── 📄 .env.example
├── 📄 dbt_project.yml
│
├── 📂 docs/
│   └── openclaw-setup.md             ← setup del backend claude-cli + MCP + cron
│
├── 📂 data/
│   └── synthetic/
│       └── generate_data.py          ← eventos sintéticos multicanal → BigQuery raw
│
├── 📂 models/                        ← dbt
│   ├── staging/
│   ├── intermediate/
│   └── marts/
│       ├── mart_orders.sql
│       └── _marts.yml                ← tests, freshness, exposures
│
├── 📂 semantic/
│   └── metrics.yml                   ← definiciones gobernadas de métricas (anclaje del agente)
│
├── 📂 argus/                         ← la aplicación del agente
│   ├── ask.py                        ← entry point de text-to-SQL
│   ├── monitors/
│   │   └── run.py                    ← suite de calidad + autodiagnóstico → Jira/Slack
│   ├── guardrails/
│   │   └── sql.py                    ← validación con sqlglot (solo lectura, sin PII, acotado)
│   ├── skills/                       ← skills de OpenClaw (run-quality-suite, text-to-sql, chart)
│   └── evals/
│       ├── cases.yml
│       ├── references/               ← queries de referencia confiables
│       └── run.py                    ← scorer de precisión de ejecución
│
├── 📂 powerbi/                       ← dashboards .pbix (exposures)
│
└── 📂 .github/workflows/
    └── ci.yml                        ← dbt build + gate de evals + lint
```

---

## 🗺️ Roadmap

```
Semana 1  ▸  Generadores sintéticos → BigQuery raw
          ▸  dbt: staging → marts + tests + freshness + exposures
          ▸  Power BI conectado (flujo de datos de punta a punta funcionando)

Semana 2  ▸  Agente OpenClaw: monitoreo autónomo → Jira + Slack
          ▸  Text-to-SQL en Slack, anclado a la capa semántica
          ▸  Guardrails + harness de evals + gate de GitHub Actions
          ▸  Traces de OpenTelemetry + contabilidad de tokens/costo
          ▸  README, diagrama de arquitectura, video demo de 90 segundos

Futuro    ▸  Ampliar cobertura de evals; tracking de regresión de la precisión en el tiempo
          ▸  Soporte multi-warehouse (paridad de fixtures Snowflake / DuckDB)
          ▸  Memoria de hilo en Slack para que las preguntas de seguimiento tengan contexto
          ▸  Sugerencias de auto-remediación adjuntas a cada ticket de incidente
```

---

## 📄 Licencia

MIT License — libre para uso educativo y de portafolio.

---

<div align="center">

**Desarrollado como Proof of Concept de confiabilidad de datos autónoma y analítica self-service.**

*Utiliza datos completamente sintéticos generados para fines demostrativos. No contiene información real de ninguna empresa.*

<br>

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Conectar-0077B5?style=for-the-badge&logo=linkedin)](https://linkedin.com/in/cesar-rabago-perez)
[![GitHub](https://img.shields.io/badge/GitHub-Más%20proyectos-181717?style=for-the-badge&logo=github)](https://github.com/cesarrabago)

</div>
