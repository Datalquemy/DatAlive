# Proyecto DatAlive (DA) — Plan General

Este documento describe todas las fases del proyecto **DatAlive (DA)**.  
El objetivo es construir una arquitectura analítica multinube (AWS y Azure, con posibilidad de extender a GCP) que permita:  
- Consultar reportes operativos diarios de producción.  
- Generar pronósticos de producción a corto y mediano plazo.  
- Explicar en **lenguaje natural** las variaciones en producción y pronósticos mediante un sistema RAG (Retrieval Augmented Generation).  

Cada fase incluye una explicación en lenguaje sencillo y las tareas específicas que se deben realizar.

---

## Fase 1 — Gobierno y seguridad base  
En esta fase aseguramos las cuentas raíz (root en AWS y Global Admin en Azure) y establecemos los cimientos de seguridad.  
El objetivo es que la infraestructura no arranque en un estado inseguro. Se aplican medidas mínimas como habilitar MFA, bloquear el uso operativo de la cuenta root, crear usuarios y roles administrados, definir políticas de contraseñas, activar llaves de cifrado y habilitar auditorías de acciones.

### Tareas
1. **Asegurar cuenta root (AWS)**
   - Activar MFA en la cuenta root.
   - Deshabilitar cualquier Access Key.
   - Root se usará solo para facturación y emergencias.
2. **Crear grupos y usuarios IAM (AWS)**
   - Grupos: Admin, PowerUser, ReadOnly.
   - Crear usuarios IAM (ej. `dq-admin`) y asignarlos.
3. **Definir roles IAM (AWS)**
   - Crear roles `dqalq-admin` y `dqalq-readonly`.
   - Probar permisos con AssumeRole.
4. **Política de contraseñas (AWS)**
   - Definir longitud mínima, complejidad y rotación.
   - Activar la política en IAM.
5. **Cifrado con KMS (AWS)**
   - Crear una CMK para **DEV**.
   - Crear una CMK para **QA**.
   - Crear una CMK para **PROD**.
   - Asociar cada CMK a sus buckets y servicios (S3, RDS, etc.).
   - Definir quién administra y quién usa las llaves.
6. **Auditoría con CloudTrail (AWS)**
   - Activar CloudTrail en todas las regiones.
   - Enviar logs a un bucket centralizado.
7. **Seguridad en Azure**
   - MFA en Global Admin.
   - Crear grupos RBAC base (Owner, Contributor, Reader).
   - Configurar Key Vault con Soft delete y Purge protection.
   - Exportar Activity Logs a Log Analytics.
8. **Documentación**
   - Guardar evidencias.
   - Registrar decisiones.
   - Resumen ejecutivo.

---

## Fase 2 — Estructura de cuentas y ambientes  
En esta fase organizamos las cuentas en AWS y las suscripciones en Azure para separar los ambientes de trabajo: Desarrollo (DEV), Pruebas (QA) y Producción (PROD).  
Esto asegura aislamiento y control de acceso en cada entorno.

### Tareas
**Ámbitos por ambiente (DEV/QA/PROD):** las cuentas (AWS) y suscripciones (Azure) se crean y administran por ambiente para aislar permisos, costos y riesgos. QA puede fusionarse o deshabilitarse más adelante si la estrategia se simplifica a *DEV/PROD*.

1. **AWS Organizations**
   - Crear OUs: Platform, DEV, QA, PROD.
   - Crear cuentas hijas y aplicar SCPs (políticas de control de servicio).
   - Configurar IAM Identity Center (SSO).
2. **Azure Management Groups**
   - Crear Tenant Root Group y Management Groups por ambiente.
   - Asignar suscripciones a cada ambiente.
   - Configurar Azure Policy básica (tags, ubicación).
3. **Identidad y acceso**
   - En AWS: roles IAM y Tag Policies.
   - En Azure: apps Entra ID (SPNs), Key Vaults y roles.
4. **Pruebas de acceso**
   - Validar lectura/escritura en buckets y contenedores.
5. **Documentación**
   - Registrar estructura creada y accesos probados.

---

## Fase 3 — Red y acceso  
Se diseña la red base en AWS y Azure para habilitar comunicación segura y segmentación por ambientes.  
El objetivo es que los servicios estén correctamente aislados y protegidos.

### Tareas
**Topología por ambiente:** cada ambiente tendrá su propia VPC/VNet y subnets dedicadas. El *peering* solo se habilita donde sea estrictamente necesario (por ejemplo, herramientas compartidas). QA es independiente de PROD.

1. **AWS**
   - Crear VPCs para DEV, QA y PROD.
   - Subnets públicas y privadas en al menos 2 AZs.
   - Configurar Internet Gateway y NAT Gateway.
   - Asociar Route Tables.
   - Crear VPC Endpoints para S3, STS, Logs.
   - Definir Security Groups y NACLs.
2. **Azure**
   - Crear VNets para DEV, QA y PROD.
   - Crear subnets: Ingesta, Compute, Data, Auxiliares.
   - Configurar NSGs por subnet.
   - Implementar Private Endpoints (Storage, Databricks).
3. **Documentación**
   - Evidencias de topología.
   - Decisiones de diseño.

---

## Fase 4 — Naming, tags y costos  
Se establecen estándares de nombres, etiquetas (tags) obligatorias y control de costos.  
Esto permite identificar cada recurso y mantener visibilidad de gastos.

### Tareas
**Convención de `Environment`:** todas las etiquetas y nombres incluyen explícitamente el ambiente (`dev`, `qa`, `prod`). Esto permite filtrar costos y aplicar políticas por entorno.

1. Definir estándar de naming (`dqalq-<serv>-<env>-<region>-<sufijo>`).
2. Configurar tags obligatorias: `Environment`, `Owner`, `CostCenter`, `Project`.
3. Configurar presupuestos y alertas en AWS Cost Explorer.
4. Configurar Cost Management en Azure.
5. Documentación.

---

## Fase 5 — Data Lake  
Aquí se construye el lago de datos, con las zonas raw/bronze/silver/gold.  
El objetivo es tener un almacenamiento confiable, versionado y con retención clara.

### Tareas
**Zonas por ambiente:** buckets/containers separados por `dev`, `qa` y `prod`, cada uno con sus carpetas `raw/bronze/silver/gold`. La escritura cruzada entre ambientes está prohibida.

1. AWS: crear buckets S3 separados por ambiente y zona.
2. Azure: crear Storage Accounts con containers.
3. Configurar versioning.
4. Definir políticas de retención.
5. Documentación.

---

## Fase 6 — Catálogo y formatos  
Se define cómo se almacenan los datos y cómo se describen.  
Esto asegura que los datos sean legibles, compatibles y eficientes.

### Tareas
**Catálogo por ambiente:** bases de datos/catalogs separados por ambiente (ej. `glue_dev`, `glue_qa`, `glue_prod`) para evitar choque de nombres y facilitar borrados controlados.

1. Configurar Glue Catalog (AWS).
2. Estándar en Parquet/Delta Lake.
3. Integrar Databricks con ADLS.
4. Documentación.

---

## Fase 7 — Ingesta batch/stream  
Se construyen los procesos para traer datos en lotes o en tiempo real.  
El objetivo es asegurar que cualquier fuente pueda alimentar el Data Lake.

### Tareas
**Pipelines por ambiente:** endpoints y credenciales separados. Los conectores apuntan a `raw` del ambiente correspondiente. No se comparte el *topic/stream* entre ambientes.

1. Ingesta batch: AWS DMS, AWS Transfer.
2. Ingesta stream: Kinesis (AWS), Event Hubs (Azure).
3. Kafka como opción agnóstica.
4. Validar ingestión hacia raw.
5. Documentación.

---

## Fase 8 — Procesamiento distribuido  
Se procesan datos de forma escalable y eficiente con Spark.  
Esto permite transformar grandes volúmenes de información.

### Tareas
**Clusters por ambiente:** EMR/Databricks dedicados por `dev/qa/prod` con autoescalado y *job queues* independientes. Los *jobs* en QA nunca escriben en PROD.

1. AWS: Spark sobre EMR o Glue.
2. Azure: Databricks conectado a ADLS.
3. Habilitar autoscaling y monitoreo.
4. Validar conexión a raw/bronze/silver/gold.
5. Documentación.

---

## Fase 9 — Orquestación  
Se automatizan los flujos de trabajo de extremo a extremo.  
El objetivo es que los pipelines se ejecuten de forma programada y confiable.

### Tareas
**DAGs por ambiente:** prefijos y variables de entorno separadas (ej. `AIRFLOW_ENV=dev|qa|prod`). Los despliegues de DAGs mantienen rutas y conexiones por ambiente.

1. Configurar Airflow (MWAA o self-managed).
2. Configurar Step Functions (AWS).
3. Crear DAGs para ingesta y transformación.
4. Documentación.

---

## Fase 10 — Calidad y validación de datos  
Se aplican reglas para garantizar que los datos sean correctos.  
Esto asegura confianza en la información usada por los analistas.

### Tareas
**Reglas por ambiente:** suites de validación por entorno (p.ej. `ge_dev.yml`, `ge_prod.yml`) y *stores* de resultados separados.

1. Implementar Great Expectations.
2. Definir reglas de calidad por capa.
3. Validar datasets contra reglas.
4. Generar reportes automáticos.
5. Documentación.

---

## Fase 11 — Capa de consumo analítico  
Se habilitan consultas sobre el Data Lake y Snowflake.  
El objetivo es que los usuarios puedan analizar datos sin moverlos.

### Tareas
**Schemas por ambiente:** catálogos y esquemas con sufijo/prefijo de ambiente (ej. `analytics_dev`, `analytics_prod`). Accesos de solo lectura para analistas en `dev`/`qa`.

1. Configurar Athena y Redshift Spectrum.
2. Configurar Snowflake External Stage.
3. Crear External Tables y Views.
4. Documentación.

---

## Fase 12 — BI y acceso externo  
Se crean tableros en Power BI para mostrar KPIs diarios.  
Esto da visibilidad operativa a los stakeholders.

### Tareas
**Workspaces por ambiente:** un *workspace* de BI por `dev/qa/prod` para separar credenciales y orígenes de datos. Solo PROD se publica a usuarios finales.

1. Conectar Power BI a Snowflake.
2. Crear tableros de producción diaria.
3. Configurar DirectQuery/Import.
4. (Opcional) QuickSight en AWS.
5. Documentación.

---

## Fase 13 — ML & MLOps  
Se construyen modelos predictivos y se gestionan con MLOps.  
Esto permite entrenar, desplegar y monitorear modelos en producción.

### Tareas
**Ciclos por ambiente:** *experiments* y *model registries* por entorno; promoción `dev → qa → prod` con revisiones. *Feature stores* separados o con *namespaces* por ambiente.

1. AWS: SageMaker para entrenar modelos.
2. Azure: Databricks ML con MLflow.
3. Implementar tracking de experimentos.
4. Versionado y promoción de modelos.
5. Documentación.

---

## Fase 14 — Observabilidad y seguridad operativa  
Se monitorean recursos y se aplican medidas de seguridad.  
El objetivo es detectar problemas y anomalías a tiempo.

### Tareas
**Monitoreo por ambiente:** métricas, alertas y tableros con etiquetas de `environment`. Alarmas de PROD con severidad distinta. *Guardrails* más estrictos en PROD.

1. Configurar CloudWatch (AWS).
2. Configurar Azure Monitor.
3. Configurar GuardDuty.
4. Definir alertas y auditoría.
5. Documentación.

---

## Fase 15 — CI/CD e IaC  
Se automatizan despliegues de infraestructura y pipelines de datos.  
Esto asegura repetibilidad y control de versiones.

### Tareas
**Ramas y despliegues por ambiente:** ramas `main`/`release` para PROD y `develop` para DEV; *workflows* que despliegan a cada ambiente con variables/secretos separados.

1. Crear pipelines con GitHub Actions o CodePipeline.
2. Configurar Terraform/CDK.
3. Desplegar infraestructura como código.
4. Documentación.

---

## Fase 16 — DR, backups y compliance  
Se definen planes de recuperación ante desastres y retención de datos.  
Esto protege la operación ante fallas.

### Tareas
**Políticas por ambiente:** retención y copias cruzadas pueden variar (más estrictas en PROD). Ensayos de recuperación en QA antes de aplicarlo a PROD.

1. Configurar versioning en S3.
2. Configurar lifecycle policies.
3. Definir retención según compliance.
4. Probar plan de recuperación.
5. Documentación.

---

## Fase 17 — API de servicio  
Se exponen insights y explicaciones a través de APIs.  
Esto permite que aplicaciones externas consulten resultados.

### Tareas
**APIs por ambiente:** dominios/subdominios distintos (ej. `api-dev`, `api-qa`, `api-prod`) y *rate limits* diferenciados. Las claves de API no se comparten.

1. Crear API con FastAPI.
2. Endpoints: `/insights`, `/rag/explain`, `/health`.
3. Desplegar en ECS Fargate o Lambda.
4. Integrar con Power BI.
5. Documentación.

---

## Fase 18 — Capa RAG  
Se integra una base de vectores para explicar datos en lenguaje natural.  
Esto convierte dashboards y reportes en conocimiento entendible.

### Tareas
**Índices por ambiente:** colecciones/índices vectoriales separados (ej. `rag_dev`, `rag_prod`) para evitar que contenido de prueba contamine respuestas de producción.

1. Configurar OpenSearch Serverless o Aurora pgvector.
2. Indexar dashboards de producción.
3. Probar consultas en lenguaje natural.
4. Validar citaciones a fuentes.
5. Documentación.

---

## Fase 19 — Integración con Snowflake  
Se integra Snowflake como motor analítico.  
Esto da flexibilidad adicional para consultas.

### Tareas
**Bases por ambiente:** `DB_DEV`, `DB_QA`, `DB_PROD` y *warehouses* con tamaños/políticas de *auto-suspend* distintos por entorno.

1. Configurar Snowpipe batch.
2. Configurar Snowpipe Streaming.
3. Crear vistas y modelos.
4. Documentación.

---

## Fase 20 — Pruebas y cutover  
Se valida todo el ecosistema antes de entrar en producción.  
Esto asegura que los pipelines funcionen sin errores.

### Tareas
**Promoción por ambiente:** *smoke tests* en DEV, *end-to-end* en QA, *readiness checklist* en PROD. El *cutover* solo ocurre en PROD.

1. Definir plan de validación.
2. Ejecutar pruebas integrales.
3. Definir rollback plan.
4. Validar con stakeholders.
5. Documentación.

---

## Fase 21 — Documentación y runbooks  
Se documenta todo el sistema para operación y soporte.  
El objetivo es que cualquier persona pueda administrar y resolver incidentes.

### Tareas
**Runbooks por ambiente:** se mantienen variantes de procedimientos cuando difieren entre `dev/qa/prod` (por ejemplo, tamaños de clúster o SLAs de soporte).

1. Crear guías técnicas (troubleshooting, operación).
2. Documentar causas de fallas y lecciones aprendidas.
3. Indexar documentación en vector DB (para RAG).
4. Documentación final.

---

---

## Estrategia de ambientes y posible deshabilitación
- **Separación por defecto:** todo el ecosistema se diseña para operar en `DEV`, `QA` y `PROD` de forma aislada, desde red y datos hasta APIs y BI.
- **Optimización futura:** una vez estabilizados los procesos, se puede **consolidar QA en DEV** o **pausarlo temporalmente** para reducir costos, manteniendo siempre **PROD** intacto.
- **Criterios para pausar QA:** cobertura de pruebas automatizadas alta, *feature flags* efectivos, *canary releases* en PROD y monitoreo robusto.
- **Cómo pausar de forma segura:** apagar recursos de cómputo (clústeres/warehouses), congelar pipelines de QA, mantener storage mínimo para evidencias/históricos y retener IaC para reactivar cuando sea necesario.
