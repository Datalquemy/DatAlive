# Proyecto DataAlive (DA) — Plan Maestro Multidominio

Este documento describe todas las fases del proyecto **DataAlive (DA)**.  

El propósito es construir una plataforma de datos **cognitiva, multidominio y multinube** (AWS y Azure, con posibilidad de extender a GCP) que permita a cualquier industria adaptarla mediante **Domain Packs** (ej. petróleo, agro, retail, automotriz, etc.).  

La arquitectura de DataAlive estará preparada para:  

- Procesar información de diferentes dominios en **tenants dedicados y seguros**.  
- Consultar **reportes operativos diarios** ajustados a cada industria.  
- Generar **pronósticos predictivos** a corto y mediano plazo mediante modelos de Machine Learning.  
- Explicar en **lenguaje natural** las variaciones en datos y pronósticos mediante un sistema **RAG (Retrieval Augmented Generation)**.  
- Integrar **orquestación, datos batch y en tiempo real** (Kafka/Event Hubs) como parte del *core* de la plataforma.  

Cada fase incluye una explicación en lenguaje sencillo y las tareas específicas (AWS/Azure) que se deben realizar, con un enfoque **parametrizable y agnóstico** para que DataAlive pueda operar en cualquier dominio.  


---

## Fase 1 — Gobierno y seguridad base

**Objetivo**  
Asegurar las cuentas raíz (root en AWS y Global Admin en Azure) y establecer los cimientos de seguridad de DataAlive.  
El propósito es que la infraestructura no arranque en un estado inseguro y esté preparada desde el inicio para operar en un contexto **multidominio y multi-tenant**.  
Se aplican medidas como habilitar MFA, bloquear el uso operativo de las cuentas raíz, crear usuarios y roles administrados, definir políticas de contraseñas, activar llaves de cifrado y habilitar auditorías de acciones.

---

### ✅ Checklist AWS

1. **Asegurar cuenta root**  
   1.1. Activar MFA en la cuenta root.  
   1.2. Eliminar cualquier Access Key.  
   1.3. Configurar root para usarse solo en facturación y emergencias.  

2. **Crear grupos y usuarios IAM**  
   2.1. Crear grupos: `Admin`, `PowerUser`, `ReadOnly`.  
   2.2. Crear usuarios IAM (`da-admin`, `da-power`, `da-reader`) y asignarlos a grupos.  
   2.3. Configurar MFA obligatorio para todos los usuarios.  
   2.4. Generar Access Keys iniciales solo si son necesarias y almacenarlas en Secrets Manager.  

3. **Definir roles IAM**  
   3.1. Crear rol `dalive-admin` con permisos de administrador.  
   3.2. Crear rol `dalive-readonly` con permisos de solo lectura.  
   3.3. Configurar políticas de confianza para `AssumeRole`.  
   3.4. Probar permisos con `sts assume-role`.  

4. **Definir política de contraseñas**  
   4.1. Longitud mínima: 14 caracteres.  
   4.2. Complejidad: incluir mayúscula, minúscula, número y símbolo.  
   4.3. Rotación cada 90 días.  
   4.4. Bloqueo tras 5 intentos fallidos.  

5. **Configurar cifrado con KMS**  
   5.1. Crear una CMK para `DEV`.  
   5.2. Crear una CMK para `QA`.  
   5.3. Crear una CMK para `PROD`.  
   5.4. Asociar CMKs a S3, RDS y servicios críticos.  
   5.5. Definir quién administra y quién usa cada llave.  

6. **Activar auditoría con CloudTrail**  
   6.1. Activar CloudTrail en todas las regiones.  
   6.2. Crear un trail por ambiente (`trail-dev`, `trail-qa`, `trail-prod`).  
   6.3. Configurar envío de logs a un bucket centralizado con cifrado KMS.  
   6.4. Integrar CloudTrail con CloudWatch Logs.  

---

### ✅ Checklist Azure

7. **Proteger Global Admin**  
   7.1. Activar MFA en la cuenta Global Admin.  
   7.2. Crear grupos RBAC: `Owner`, `Contributor`, `Reader`.  
   7.3. Crear usuarios base (`da-owner`, `da-contributor`, `da-reader`) y asignarlos a grupos.  

8. **Configurar Key Vault**  
   8.1. Crear un Key Vault por ambiente (`kv-dev`, `kv-qa`, `kv-prod`).  
   8.2. Activar Soft Delete.  
   8.3. Activar Purge Protection.  

9. **Habilitar logs de actividad**  
   9.1. Exportar *Activity Logs* hacia Log Analytics.  
   9.2. Validar que eventos críticos (inicio de sesión, cambios de permisos) se registren.  

---

### 📄 Documentación

10. **Generar evidencias y reportes**  
   10.1. Guardar capturas de configuración de root/MFA.  
   10.2. Registrar ARNs, roles y usuarios creados.  
   10.3. Crear un resumen ejecutivo de las medidas de seguridad implementadas.  


---

## Fase 2 — Estructura de cuentas y ambientes

**Objetivo**  
Organizar las cuentas en AWS y las suscripciones en Azure para separar los ambientes de trabajo: **Desarrollo (DEV)**, **Pruebas (QA)** y **Producción (PROD)**.  
Esto asegura aislamiento, control de acceso, trazabilidad de costos y reduce riesgos de que un cambio en un entorno afecte a otro.  
La estrategia se diseña desde el inicio para soportar el enfoque **multidominio y multi-tenant** de DataAlive, garantizando que cada cliente tenga un entorno independiente y seguro.

---

### ✅ Checklist AWS

1. **Configurar AWS Organizations**  
   1.1. Crear una Organización raíz.  
   1.2. Crear OUs: `Platform`, `DEV`, `QA`, `PROD`.  
   1.3. Crear cuentas hijas para cada ambiente.  
   1.4. Aplicar SCPs (políticas de control de servicio) por ambiente.  
   1.5. Configurar IAM Identity Center (SSO) para centralizar accesos.  

2. **Configurar roles y permisos IAM**  
   2.1. Crear roles de administración por ambiente (`dalive-dev-admin`, `dalive-qa-admin`, `dalive-prod-admin`).  
   2.2. Crear roles de solo lectura por ambiente (`dalive-dev-readonly`, `dalive-qa-readonly`, `dalive-prod-readonly`).  
   2.3. Definir permisos mínimos por OU.  
   2.4. Probar roles con `sts assume-role`.  

3. **Configurar políticas de tags y costos**  
   3.1. Definir políticas de tags obligatorias (`Environment`, `Owner`, `CostCenter`, `Project`).  
   3.2. Asociar políticas a cada cuenta hija.  
   3.3. Validar que los recursos creados incluyan las etiquetas.  

---

### ✅ Checklist Azure

4. **Configurar Azure Management Groups**  
   4.1. Crear `Tenant Root Group`.  
   4.2. Crear *Management Groups* por ambiente (`mg-dev`, `mg-qa`, `mg-prod`).  
   4.3. Asignar suscripciones a cada ambiente.  
   4.4. Configurar Azure Policy básica:  
       - Tags obligatorias (`Environment`, `Owner`, `CostCenter`, `Project`).  
       - Restricciones de ubicación (regiones permitidas).  

5. **Configurar identidades y accesos en Azure Entra ID**  
   5.1. Crear aplicaciones (SPNs) para acceso a servicios.  
   5.2. Configurar roles RBAC por ambiente (`Owner`, `Contributor`, `Reader`).  
   5.3. Crear un Key Vault por ambiente y asignar permisos mínimos.  

---

### ✅ Validaciones cruzadas

6. **Pruebas de acceso**  
   6.1. Validar lectura/escritura en buckets S3 de cada ambiente.  
   6.2. Validar lectura/escritura en contenedores de Storage Accounts.  
   6.3. Probar accesos a Key Vaults y KMS según roles definidos.  

---

### 📄 Documentación

7. **Registrar estructura creada**  
   7.1. Documentar OUs, cuentas y roles en AWS.  
   7.2. Documentar Management Groups, suscripciones y roles en Azure.  
   7.3. Guardar evidencias de pruebas de acceso realizadas.  
   7.4. Crear un resumen ejecutivo de la estrategia de ambientes.  

---

## Fase 3 — Red y acceso

**Objetivo**  
Diseñar la red base en AWS y Azure para habilitar comunicación segura y segmentación por ambientes (DEV, QA, PROD).  
Cada entorno debe estar correctamente aislado y protegido, soportando el enfoque multidominio de DataAlive, con capacidad de crecer y adaptarse a cada tenant.

---

### ✅ Checklist AWS

1. **Crear VPCs por ambiente**  
   1.1. Crear `vpc-dev`, `vpc-qa`, `vpc-prod`.  
   1.2. Definir CIDR blocks no solapados.  

2. **Crear subnets**  
   2.1. Subnets públicas y privadas en al menos 2 AZs por VPC.  
   2.2. Asociar etiquetas de ambiente (`Environment=dev`, etc.).  

3. **Configurar gateways y routing**  
   3.1. Crear Internet Gateway y asociarlo a las VPCs.  
   3.2. Crear NAT Gateway en subnets públicas.  
   3.3. Configurar Route Tables y asociarlas a subnets.  

4. **Configurar VPC Endpoints**  
   4.1. Crear endpoints privados para S3.  
   4.2. Crear endpoints para STS y Logs.  

5. **Definir reglas de seguridad**  
   5.1. Crear Security Groups por servicio.  
   5.2. Configurar NACLs por subnet.  
   5.3. Documentar reglas de entrada/salida permitidas.  

---

### ✅ Checklist Azure

6. **Crear VNets por ambiente**  
   6.1. Crear `vnet-dev`, `vnet-qa`, `vnet-prod`.  
   6.2. Definir rangos de IPs no solapados.  

7. **Crear subnets especializadas**  
   7.1. Subnet `Ingesta`.  
   7.2. Subnet `Compute`.  
   7.3. Subnet `Data`.  
   7.4. Subnets auxiliares.  

8. **Configurar seguridad de red**  
   8.1. Crear NSGs por subnet.  
   8.2. Definir reglas de acceso (ej. solo Bastion a subnets privadas).  

9. **Habilitar Private Endpoints**  
   9.1. Para Storage Accounts.  
   9.2. Para Databricks.  
   9.3. Para Key Vaults.  

---

### 📄 Documentación

10. **Evidencias y diseño**  
   10.1. Guardar diagramas de topología.  
   10.2. Documentar rangos CIDR usados.  
   10.3. Registrar Security Groups y NSGs configurados.  

---

## Fase 4 — Naming, tags y costos

**Objetivo**  
Establecer estándares de nombres, etiquetas (tags) obligatorias y control de costos.  
Esto permite identificar cada recurso, filtrar gastos por tenant/ambiente y mantener visibilidad centralizada.

---

### ✅ Checklist General

1. **Definir convención de nombres**  
   1.1. Formato estándar: `dalive-<serv>-<env>-<region>-<sufijo>`.  
   1.2. Publicar guía de naming para equipos.  

2. **Definir tags obligatorias**  
   2.1. `Environment` (dev, qa, prod).  
   2.2. `Owner`.  
   2.3. `CostCenter`.  
   2.4. `Project`.  
   2.5. `Domain` (ej. petróleo, agro, retail).  

---

### ✅ Checklist AWS

3. **Configurar control de costos en AWS**  
   3.1. Crear presupuestos en AWS Cost Explorer.  
   3.2. Configurar alertas por tenant y ambiente.  
   3.3. Asociar políticas de tags con AWS Organizations.  

---

### ✅ Checklist Azure

4. **Configurar control de costos en Azure**  
   4.1. Habilitar Cost Management + Billing.  
   4.2. Configurar presupuestos por suscripción.  
   4.3. Configurar alertas de gasto.  

---

### 📄 Documentación

5. **Registro de políticas y ejemplos**  
   5.1. Publicar ejemplos de nombres correctos/incorrectos.  
   5.2. Guardar evidencias de alertas configuradas.  

---

## Fase 5 — Data Lake

**Objetivo**  
Construir el Data Lake con zonas **raw/bronze/silver/gold** separadas por ambiente.  
El objetivo es contar con almacenamiento confiable, versionado, con retención clara y preparado para datos de distintos dominios.

---

### ✅ Checklist AWS

1. **Crear buckets por ambiente y zona**  
   1.1. `dalive-dev-raw`, `dalive-dev-bronze`, `dalive-dev-silver`, `dalive-dev-gold`.  
   1.2. Repetir para QA y PROD.  

2. **Configurar versioning**  
   2.1. Activar versioning en cada bucket.  
   2.2. Probar restauración de versiones.  

3. **Definir políticas de retención**  
   3.1. Configurar lifecycle policies en S3.  
   3.2. Definir retención distinta por ambiente.  

---

### ✅ Checklist Azure

4. **Crear Storage Accounts por ambiente**  
   4.1. `stgdalive-dev`, `stgdalive-qa`, `stgdalive-prod`.  

5. **Crear containers por zona**  
   5.1. `raw`, `bronze`, `silver`, `gold`.  
   5.2. Repetir para cada ambiente.  

6. **Configurar versioning y retención**  
   6.1. Activar versioning en Storage Accounts.  
   6.2. Configurar políticas de retención.  

---

### 📄 Documentación

7. **Evidencias y guías**  
   7.1. Registrar estructura de buckets/containers creada.  
   7.2. Documentar políticas de retención aplicadas.  
   7.3. Crear diagrama de zonas del Data Lake.  

---

## Fase 6 — Catálogo y formatos

**Objetivo**  
Definir cómo se almacenan y describen los datos dentro del Data Lake.  
Esto asegura que los datos sean **legibles, compatibles y eficientes**, y que cada dominio pueda ser integrado sin romper la arquitectura.  
El catálogo se gestiona por ambiente para evitar choques de nombres y facilitar mantenimientos.

---

### ✅ Checklist AWS

1. **Configurar Glue Catalog por ambiente**  
   1.1. Crear `glue_dev`, `glue_qa`, `glue_prod`.  
   1.2. Crear bases de datos por dominio (`oil_prod`, `retail_sales`, etc.).  
   1.3. Definir permisos de acceso por base de datos.  

2. **Definir estándares de formato**  
   2.1. Estándar primario: **Parquet**.  
   2.2. Estándar transaccional: **Delta Lake**.  
   2.3. Probar lectura y escritura en cada formato.  

3. **Integrar con Data Lake**  
   3.1. Validar Glue con S3 raw/bronze/silver/gold.  
   3.2. Configurar crawlers para descubrimiento automático.  

---

### ✅ Checklist Azure

4. **Integrar catálogo en Databricks/ADLS**  
   4.1. Crear catálogos por ambiente (`catalog_dev`, `catalog_qa`, `catalog_prod`).  
   4.2. Integrar Databricks con ADLS y habilitar Unity Catalog.  
   4.3. Configurar bases por dominio.  

---

### 📄 Documentación

5. **Guías de uso**  
   5.1. Registrar catálogos creados.  
   5.2. Documentar estándares de formato aceptados.  
   5.3. Guardar evidencias de pruebas de integración.  

---

## Fase 7 — Ingesta batch/stream

**Objetivo**  
Construir procesos de ingesta en **lotes (batch)** y en **tiempo real (streaming)** para alimentar el Data Lake.  
Esto asegura que cualquier fuente pueda integrarse y que cada dominio defina cómo ingresar sus datos, sin impactar a otros.

---

### ✅ Checklist Batch

1. **Ingesta batch en AWS**  
   1.1. Configurar AWS DMS para fuentes relacionales.  
   1.2. Configurar AWS Transfer para cargas SFTP.  
   1.3. Validar escritura en `raw`.  

2. **Ingesta batch en Azure**  
   2.1. Configurar Data Factory para pipelines batch.  
   2.2. Crear datasets por ambiente.  
   2.3. Validar escritura en `raw`.  

---

### ✅ Checklist Streaming

3. **Ingesta streaming en AWS**  
   3.1. Configurar Kinesis Data Streams.  
   3.2. Asociar Firehose a buckets `raw`.  
   3.3. Probar ingesta con mensajes de prueba.  

4. **Ingesta streaming en Azure**  
   4.1. Configurar Event Hubs para cada ambiente.  
   4.2. Asociar con Databricks Structured Streaming.  
   4.3. Validar escritura en `raw`.  

5. **Kafka como opción agnóstica**  
   5.1. Instalar Kafka en clústeres compartidos.  
   5.2. Crear topics por dominio (`oil_prod`, `retail_sales`, etc.).  
   5.3. Validar lectura y escritura de mensajes.  

---

### 📄 Documentación

6. **Registro de pipelines**  
   6.1. Documentar conectores configurados.  
   6.2. Guardar ejemplos de datasets ingeridos.  
   6.3. Crear diagrama de flujo batch y streaming.  

---

## Fase 8 — Procesamiento distribuido

**Objetivo**  
Procesar datos de forma **escalable y eficiente** usando Spark.  
Esto permite transformar grandes volúmenes de información en pipelines robustos, aplicables a distintos dominios, garantizando aislamiento por ambiente.

---

### ✅ Checklist AWS

1. **Clusters EMR/Glue**  
   1.1. Crear clúster Spark en EMR por ambiente.  
   1.2. Configurar colas de jobs separadas (dev/qa/prod).  
   1.3. Habilitar autoscaling.  
   1.4. Configurar acceso a raw/bronze/silver/gold.  

2. **Monitoreo**  
   2.1. Habilitar métricas de EMR en CloudWatch.  
   2.2. Configurar alertas de fallos de jobs.  

---

### ✅ Checklist Azure

3. **Clusters Databricks**  
   3.1. Crear clusters por ambiente (`dbx-dev`, `dbx-qa`, `dbx-prod`).  
   3.2. Configurar pools de autoscaling.  
   3.3. Validar integración con ADLS.  

4. **Orquestar jobs Spark**  
   4.1. Crear notebooks por dominio (ej. petróleo, retail).  
   4.2. Configurar tareas programadas en Job Scheduler.  
   4.3. Validar outputs en bronze/silver.  

---

### 📄 Documentación

5. **Pruebas y registros**  
   5.1. Guardar evidencias de ejecución de jobs.  
   5.2. Documentar parámetros de clusters.  
   5.3. Crear un diagrama del flujo de transformación.  


---

## Fase 9 — Orquestación

**Objetivo**  
Automatizar los flujos end-to-end para cada **tenant** y **dominio** (batch y streaming), de forma confiable y parametrizada por **env (dev/qa/prod)**.  
Los orquestadores deben leer variables (tenant/domain/env/SLA) para armar el plan de ejecución, manejar **retries/backfill**, aplicar **ventanas de SLA**, y registrar telemetría/alertas. La promoción **dev→qa→prod** se realizará con aprobaciones ligeras y secretos gestionados de forma segura.

---

### ✅ Checklist AWS

1. **Airflow/MWAA (DAGs parametrizados)**
   1.1. Crear entorno MWAA por ambiente (`mwAA-dev`, `mwAA-qa`, `mwAA-prod`).  
   1.2. Definir variables globales: `TENANT`, `DOMAIN`, `ENV`, `SLA`, `DATA_ROOT`.  
   1.3. Estándar de DAGs: naming `dalive_<domain>_<pipeline>_<env>`.  
   1.4. Políticas de `retries`, `retry_delay`, `max_active_runs`, `catchup`.  
   1.5. Operadores para: ingest (batch/stream), transform (Spark), quality (GE), publish (semántica/BI).

2. **EventBridge/Step Functions**
   2.1. Reglas EventBridge por disparador (cron, file-arrival, webhook).  
   2.2. Step Functions para orquestaciones de múltiples servicios (DMS/Kinesis/S3/Glue).  
   2.3. Map state por dominio y rama condicional por `SLA` (D+1 vs near-real-time).

3. **Gestión de secretos y parámetros**
   3.1. AWS Secrets Manager: credenciales por tenant/ambiente.  
   3.2. Parameter Store: rutas y toggles de features por dominio.  
   3.3. Rotación programada de secretos (cada 90 días).

4. **Backfill y ventanas de datos**
   4.1. Tareas `catchup` por rango (`start_date`, `end_date`).  
   4.2. Validar no solapamiento de particiones al re-procesar.  
   4.3. Etiquetado de runs (`RunType=Backfill|Scheduled|OnDemand`).

5. **Alertas y SLA**
   5.1. CloudWatch Alarms por DAG/pipeline (éxito/fallo/duración).  
   5.2. Notificaciones por SNS (correo/Slack) con `Tenant/Domain/Env`.  
   5.3. Panel de SLA (tiempo de fin vs ventana de disponibilidad).

---

### ✅ Checklist Azure

6. **Azure Data Factory / Synapse Pipelines**
   6.1. Crear ADF por ambiente (`adf-dev`, `adf-qa`, `adf-prod`).  
   6.2. Linked Services y Datasets por tenant/domain/env.  
   6.3. Pipelines parametrizados (tenant/domain/env/SLA).  
   6.4. Triggers: schedule, event-based (Event Grid para file-arrival) y manual.

7. **Databricks Jobs**
   7.1. Jobs por dominio con parámetros (tenant/domain/env).  
   7.2. Pools/Clusters dedicados por ambiente.  
   7.3. Reintentos, timeouts y dependencias entre tareas.

8. **Key Vault e identidades**
   8.1. Secretos por tenant/ambiente en Key Vault.  
   8.2. Managed Identities para ADF/Databricks y RBAC mínimo.

9. **Backfill y SLA**
   9.1. Pipelines de backfill con rangos de fecha.  
   9.2. Alertas de fallo/latencia vía Azure Monitor/Action Groups.  
   9.3. Tablero de cumplimiento de ventanas SLA.

---

### 📄 Documentación

10. **Evidencias y estándares**
   10.1. Publicar estándar de nombres de DAG/Jobs/Triggers.  
   10.2. Diagramas de orquestación (batch/stream).  
   10.3. Matriz de variables: `TENANT`, `DOMAIN`, `ENV`, `SLA`, `TIME_GRAIN`.  
   10.4. Runbook de backfill y de emergencia (reanudar/pausar).

---

## Fase 10 — Calidad y validación de datos

**Objetivo**  
Asegurar la **confiabilidad** de los datos en todas las capas (raw/bronze/silver/gold) mediante **reglas declarativas por dominio** y **contratos de datos**.  
Se implementarán suites de **Great Expectations (GE)** y validaciones de **contrato (schema/constraints)** con severidades (crítica/mayor/menor). La publicación a **Gold** se **bloquea** si fallan reglas críticas.

---

### ✅ Checklist General

1. **Diseño de la estrategia DQ**
   1.1. Catálogo de reglas por capa (nulls, rangos, unicidad, referenciales).  
   1.2. Severidad por regla: Crítica (bloquea), Mayor (bloquea Gold), Menor (solo alerta).  
   1.3. Contratos de datos por dominio (JSON Schema/Delta constraints).

2. **Great Expectations (GE)**
   2.1. Repos de suites por dominio/ambiente (`ge_oil_dev`, etc.).  
   2.2. Stores separados de resultados por env (dev/qa/prod).  
   2.3. Data Docs generados y publicados (acceso de solo lectura).  
   2.4. Integración con orquestación (Airflow/ADF) como pasos obligatorios.

3. **Gates de publicación**
   3.1. Si fallan reglas críticas → detener pipeline antes de Gold.  
   3.2. Registrar artefactos fallidos en `quarantine` con etiquetas.  
   3.3. Reproceso automático tras corrección (retries/backoff).

4. **Métricas y observabilidad**
   4.1. KPIs de DQ: % filas válidas, fallas por regla, tiempo de validación.  
   4.2. Alertas de DQ via SNS/Action Groups con `Tenant/Domain/Env`.  
   4.3. Panel DQ por dominio (tendencias y severidades).

---

### ✅ Checklist AWS

5. **Integración AWS**
   5.1. GE ejecutado en Jobs (Glue/Spark) o MWAA tasks.  
   5.2. Resultados a S3 (`dq-results/<env>/<domain>/`).  
   5.3. CloudWatch Logs y alarmas de fallas críticas.  
   5.4. Delta constraints/Expectations en tablas Bronze/Silver/Gold.

---

### ✅ Checklist Azure

6. **Integración Azure**
   6.1. GE desde Databricks Jobs (Clusters por env).  
   6.2. Resultados a ADLS (`/dq-results/<env>/<domain>/`).  
   6.3. Alertas por Azure Monitor (Log Analytics consultas guardadas).  
   6.4. Constraints en Delta/Unity Catalog (NOT NULL, CHECK, UNIQUE).

---

### 📄 Documentación

7. **Guías y evidencias**
   7.1. Catálogo de reglas DQ por dominio/capa (tabla maestra).  
   7.2. Contratos de datos (schemas) versionados por dominio.  
   7.3. Procedimiento de “quarantine & release”.  
   7.4. Informe mensual de DQ por tenant/domain.

---

## Fase 11 — Capa de consumo analítico

**Objetivo**  
Habilitar consultas analíticas **sin mover los datos** del lake, con **esquemas externos y vistas** por dominio, gobernados por permisos de solo lectura.  
Alinear los modelos con un **contrato canónico** (entidad/tiempo/valor/unidad) para que la capa semántica (BI/SQL) sea **parametrizable** y re-usable en múltiples dominios y tenants.

---

### ✅ Checklist AWS

1. **Athena / Lake Formation**
   1.1. Catálogos por env (`glue_dev/qa/prod`) registrados en Lake Formation.  
   1.2. Databases por dominio (`oil_gold`, `retail_gold`, etc.).  
   1.3. External Tables sobre `gold` (particionadas por fecha/tenant/domain).  
   1.4. Permisos LF granulares (READ a roles de analistas).

2. **Redshift Spectrum (opcional)**
   2.1. Crear `external schema` apuntando a Glue Catalog.  
   2.2. Definir vistas materiales para queries frecuentes.  
   2.3. WLM/Concurrencia por tipo de usuario (analista vs servicio).

---

### ✅ Checklist Azure

3. **Synapse Serverless / Databricks SQL**
   3.1. Habilitar pools serverless para consultas sobre ADLS.  
   3.2. Crear **views** por dominio (proyección canónica entidad/tiempo/valor).  
   3.3. Roles de solo lectura y mascaramiento de datos sensibles si aplica.

4. **Purview/Lineage (opcional)**
   4.1. Registrar assets de `gold` y relaciones con pipelines.  
   4.2. Habilitar búsqueda/descubrimiento para analistas.

---

### ✅ Checklist Snowflake (complementario)

5. **External Stage y External Tables**
   5.1. Crear `STAGE` apuntando a S3/ADLS (por env y dominio).  
   5.2. Definir `EXTERNAL TABLES` sobre `gold`.  
   5.3. Vistas lógicas alineadas al contrato canónico.  
   5.4. Warehouses con auto-suspend/auto-resume por costo.

---

### 📄 Documentación

6. **Manuales y evidencias**
   6.1. Diccionario de datos por dominio (campos, tipos, particiones).  
   6.2. Guía de acceso para analistas (Athena/Synapse/Snowflake).  
   6.3. Estándar de vistas canónicas (plantillas SQL).  
   6.4. Evidencias de permisos de solo lectura y pruebas de consulta.


---

## Fase 12 — BI y acceso externo

**Objetivo**  
Crear tableros de BI operativos y ejecutivos que lean directamente de la capa **Gold**, con un **modelo semántico parametrizable** por dominio (Field Parameters/Calculation Groups) y workspaces aislados por **tenant** y **ambiente** (dev/qa/prod).  
El acceso externo se limitará a PROD y bajo RBAC/RLS internos del tenant.

---

### ✅ Checklist Power BI (principal)

1. **Workspaces por ambiente y tenant**  
   1.1. Crear workspaces: `pbi-<tenant>-dev`, `pbi-<tenant>-qa`, `pbi-<tenant>-prod`.  
   1.2. Asignar roles: `WorkspaceAdmin`, `Developer`, `Viewer`.  
   1.3. Configurar gateways/Direct Lake/DirectQuery según fuente (Snowflake, Synapse, Databricks SQL).

2. **Modelo semántico parametrizable**  
   2.1. Definir tabla de `DimDate` y jerarquías (Año–Mes–Día/Turno).  
   2.2. Implementar **Field Parameters** para activar dims/medidas por dominio.  
   2.3. Medidas canónicas: `Total`, `Promedio`, `%Participación`, `YoY`, `Rolling N`.  
   2.4. Cálculos específicos viven en el **Domain Pack** (no en el core).

3. **Conexiones a datos (por dominio)**  
   3.1. Dataset `gold/<domain>/facts` con vistas canónicas (entidad/tiempo/valor).  
   3.2. Definir orígenes por ambiente (dev/qa/prod).  
   3.3. Comprobar rendimiento: medidas con agregaciones pre-calculadas si aplica.

4. **Seguridad (RLS interno del tenant)**  
   4.1. Roles de planta/región/sitio según `DimOrg` del tenant.  
   4.2. Mapear usuarios del tenant a roles RLS.  
   4.3. Validar recortes de datos por perfil (tests con usuarios dummy).

5. **Publicación y ciclo de vida**  
   5.1. Publicar primero en DEV, luego promover a QA y PROD.  
   5.2. Configurar `Deployment Pipelines` (si aplica).  
   5.3. Documentar conexiones, parámetros y reglas RLS.

---

### ✅ Checklist AWS/Azure (conectividad y fuentes)

6. **Fuentes AWS**  
   6.1. Exponer vistas `Gold` vía Athena/Redshift Spectrum con permisos READ.  
   6.2. Si se usa Power BI con Athena: revisar conector y latencias; evaluar extracto incremental.  
   6.3. Alternativa: publicar a Snowflake y conectar Power BI a Snowflake.

7. **Fuentes Azure**  
   7.1. Exponer vistas `Gold` vía Synapse Serverless o Databricks SQL.  
   7.2. Configurar `Azure AD SSO` para Power BI.  
   7.3. Verificar Direct Lake/DirectQuery según caso de uso.

---

### 📄 Documentación

8. **Guías y evidencias**  
   8.1. Diccionario de medidas y dims (por dominio).  
   8.2. Matriz RLS (rol → filtro).  
   8.3. Procedimiento de despliegue DEV→QA→PROD.  
   8.4. Evidencias de rendimiento (DAX Studio/Query Diagnostics).

---

## Fase 13 — ML & MLOps

**Objetivo**  
Desarrollar y operar modelos de **pronóstico y detección** por dominio con prácticas de **MLOps**: tracking de experimentos, registro de modelos, despliegue por ambiente, monitoreo de deriva y re-entrenamiento.  
Todo versionado y aislado por **tenant** y **dominio**.

---

### ✅ Checklist Diseño y Datos

1. **Definir problema y dataset por dominio**  
   1.1. Casos típicos: pronóstico de producción/demanda, anomalías, scrap/mermas.  
   1.2. Seleccionar features desde `Gold` (y `Silver` si aplica).  
   1.3. Particiones temporales y validación **time-series** (train/val/test).

2. **Feature Store / Versionado**  
   2.1. Crear vistas/tabla de features por dominio (`fs_<domain>_<version>`).  
   2.2. Registrar esquema y linaje de features.  
   2.3. Controlar drift de features (estadísticos base).

---

### ✅ Checklist AWS

3. **SageMaker + MLflow (o SM Experiments)**  
   3.1. Experimentos por env: `ml_<domain>_dev`, `ml_<domain>_qa`, `ml_<domain>_prod`.  
   3.2. Entrenamiento con pipelines (Processing/Training/Inference).  
   3.3. Registro de modelos por versión (`v1`, `v2`) y **staging** (`Staging`/`Production`).  
   3.4. Endpoints por ambiente con auto-scaling y canary (traffic shifting).  
   3.5. Monitoreo: latencia, error rate, distribución de features.

---

### ✅ Checklist Azure

4. **Databricks ML + MLflow**  
   4.1. Experimentos por env y dominio (`/ml/<env>/<domain>/`).  
   4.2. Entrenamiento en Jobs; artefactos en DBFS/ADLS versionados.  
   4.3. Model Registry: registrar, transicionar (`Staging`→`Production`).  
   4.4. Despliegue:  
       - Batch scoring (Jobs programados).  
       - Real-time (Model Serving o AKS/ACI).  
   4.5. Monitoreo de deriva (evidently/propias métricas) y alertas.

---

### ✅ Operación y Gobierno

5. **Ciclo de vida y control de cambios**  
   5.1. Política de retraining (semanal/mensual o por drift).  
   5.2. Pruebas de regresión (métricas: MAPE, RMSE, F1, AUC según caso).  
   5.3. Rollback express (mantener N versiones) y plan de contingencia.  
   5.4. Costeo por entrenamiento/inferencia (etiquetas `Tenant/Domain/Env`).

---

### 📄 Documentación

6. **Manual ML**  
   6.1. Hoja de modelo (objetivo, datos, features, métricas, límites).  
   6.2. Procedimiento de promoción y retiro de modelos.  
   6.3. Evidencias de monitoreo y alertas de deriva.

---

## Fase 14 — Observabilidad y seguridad operativa

**Objetivo**  
Implementar **monitoreo integral**, alertas y **guardrails de seguridad** para proteger la plataforma y permitir reacción temprana ante fallas.  
Cubrir métricas de pipelines, cómputo, costos, seguridad (IAM/RBAC) y calidad de datos, con paneles por **tenant/domain/env**.

---

### ✅ Checklist Monitoreo (común)

1. **Telemetría de pipelines**  
   1.1. Métricas: duración, éxito/fallo, throughput, colas.  
   1.2. Logs estructurados con `Tenant/Domain/Env/Pipeline/SLA`.  
   1.3. Trazabilidad (job id → dataset → dashboard).

2. **Monitoreo de costos**  
   2.1. Dashboards de gasto por etiqueta (`Tenant`, `Domain`, `Env`).  
   2.2. Alertas por umbrales diarios/mensuales.  
   2.3. Reportes automáticos a stakeholders del tenant.

3. **Seguridad y cumplimiento**  
   3.1. Alertas de IAM/RBAC (elevación de privilegios, fallos MFA).  
   3.2. Detección de amenazas en servicios administrados.  
   3.3. Revisión periódica de claves/secretos y rotación.

---

### ✅ Checklist AWS

4. **CloudWatch / GuardDuty / Security Hub**  
   4.1. CloudWatch: logs, métricas y alarmas por pipeline/cluster.  
   4.2. GuardDuty: habilitar en todas las cuentas/regs (hallazgos críticos).  
   4.3. Security Hub: agrupar cumplimiento (CIS, Foundational).  
   4.4. EventBridge para enrutar alertas a SNS/Slack/Email.

5. **CloudTrail y Config**  
   5.1. CloudTrail multi-region con S3 centralizado (KMS).  
   5.2. AWS Config: reglas de cumplimiento (cifrado en reposo, public access off).  
   5.3. Remediación automática para desvíos comunes (SSM Automation).

---

### ✅ Checklist Azure

6. **Azure Monitor / Log Analytics / Sentinel**  
   6.1. Azure Monitor: métricas de servicios y alertas.  
   6.2. Log Analytics: consultas guardadas para incidentes comunes.  
   6.3. Microsoft Sentinel (opcional): reglas SIEM para eventos críticos.  
   6.4. Action Groups: notificaciones (correo/Teams/Webhook).

7. **Defender for Cloud / Policies**  
   7.1. Habilitar Defender para servicios clave (Storage, SQL, AKS).  
   7.2. Azure Policy: auditoría de cifrado, endpoints públicos, tagging.  
   7.3. Remediaciones automáticas donde aplique.

---

### 📄 Documentación

8. **Runbooks y evidencias**  
   8.1. Runbook de incidentes (detección → triage → mitigación → RCA).  
   8.2. Catálogo de alarmas (nombre, umbral, dueño, acción).  
   8.3. Informes mensuales: disponibilidad, MTTR, hallazgos de seguridad.  

---

## Fase 15 — CI/CD e Infraestructura como Código (IaC)

**Objetivo**  
Automatizar la construcción y despliegue de la infraestructura y pipelines de datos mediante **CI/CD**, asegurando **repetibilidad, trazabilidad y versionado**.  
Cada ambiente (DEV/QA/PROD) debe ser reproducible desde código, con ramas/versiones que controlen su promoción.

---

### ✅ Checklist General

1. **Repositorios y ramas**
   1.1. Repos separados para Infraestructura, Data Pipelines y ML.  
   1.2. Convención de ramas:  
       - `main` → PROD  
       - `release/*` → QA  
       - `develop` → DEV  
   1.3. Estrategia de PR obligatoria con revisiones.  

2. **Workflows CI/CD**
   2.1. Pipelines de build y test para código infra (Terraform/CDK/Bicep).  
   2.2. Pipelines de deploy: ambientes separados con `env vars` y `secrets`.  
   2.3. Validaciones previas: lint, policy-as-code (OPA/Conftest).  
   2.4. Promoción manual/revisiones en QA → PROD.  

---

### ✅ Checklist AWS

3. **CodePipeline / GitHub Actions**
   3.1. Crear pipelines por dominio/infraestructura.  
   3.2. Integrar con Terraform o CDK.  
   3.3. Despliegue en cuentas separadas (`dev/qa/prod`).  
   3.4. Validaciones post-deploy (smoke tests).  

---

### ✅ Checklist Azure

4. **Azure DevOps / GitHub Actions**
   4.1. Crear pipelines YAML por dominio.  
   4.2. Integrar con Bicep/Terraform para IaC.  
   4.3. Variables/secrets en Azure Key Vault.  
   4.4. Gates de aprobación entre QA → PROD.  

---

### 📄 Documentación

5. **Evidencias**
   5.1. Diagrama de flujos CI/CD.  
   5.2. Guía de ramas y PRs.  
   5.3. Catálogo de pipelines por dominio.  

---

## Fase 16 — DR, backups y compliance

**Objetivo**  
Definir planes de **recuperación ante desastres (DR)**, políticas de **retención de datos** y pruebas periódicas de recuperación.  
Esto protege la continuidad del negocio de cada tenant/domain y asegura cumplimiento regulatorio.

---

### ✅ Checklist AWS

1. **S3 y Versioning**
   1.1. Activar versioning en buckets (raw/bronze/silver/gold).  
   1.2. Replicación cruzada (cross-region replication) en PROD.  
   1.3. Lifecycle policies: transición a Glacier según retención.  

2. **Snapshots**
   2.1. Snapshots regulares en RDS/Redshift.  
   2.2. Copias automáticas cross-region.  
   2.3. Validación de restauración.  

3. **Backups Infra**
   3.1. Backup de configs IAM, CloudFormation/CDK.  
   3.2. Guardar en S3 seguro (cifrado KMS).  

---

### ✅ Checklist Azure

4. **Storage Accounts**
   4.1. Activar versioning y soft delete en containers.  
   4.2. Configurar políticas de retención (GDPR, SOX, sectoriales).  
   4.3. Replicación GRS/LRS según criticidad.  

5. **Snapshots y backups**
   5.1. Snapshots de SQL DB, Synapse, Databricks.  
   5.2. Copias automáticas en región secundaria.  
   5.3. Pruebas de restauración programadas.  

---

### ✅ Checklist Compliance

6. **Políticas globales**
   6.1. Definir retención por dominio (ej. petróleo: 7 años, retail: 3 años).  
   6.2. Auditorías de cumplimiento anuales.  
   6.3. Evidencias en carpeta centralizada de compliance.  

---

### 📄 Documentación

7. **Plan DR**
   7.1. Runbook de recuperación (RTO, RPO).  
   7.2. Escenarios de prueba (fallo región, borrado accidental).  
   7.3. Resultados de pruebas y mejoras aplicadas.  

---

## Fase 17 — API de servicio

**Objetivo**  
Exponer **insights, pronósticos y explicaciones RAG** mediante APIs seguras y escalables.  
Esto permite a aplicaciones externas y BI consumir resultados directamente, con rate limits y autenticación separada por tenant/domain/env.

---

### ✅ Checklist General

1. **Diseño de la API**
   1.1. Framework base: **FastAPI**.  
   1.2. Endpoints:  
       - `/insights` (KPIs, métricas).  
       - `/forecasts` (predicciones ML).  
       - `/rag/explain` (explicaciones en lenguaje natural).  
       - `/health` (estado).  
   1.3. Versionado: `v1`, `v2` para cambios mayores.  

2. **Seguridad**
   2.1. API Keys únicas por tenant/domain.  
   2.2. Límite de rate requests por plan (ej. Free/Premium).  
   2.3. Logs de acceso con tenant/domain/env.  

---

### ✅ Checklist AWS

3. **Despliegue en AWS**
   3.1. API Gateway (REST/HTTP API) por env.  
   3.2. Lambda/ECS Fargate para ejecutar FastAPI.  
   3.3. CloudWatch Logs por tenant/domain.  
   3.4. WAF para proteger endpoints expuestos.  

---

### ✅ Checklist Azure

4. **Despliegue en Azure**
   4.1. Azure API Management (APIM) por env.  
   4.2. AKS o Azure Functions con FastAPI.  
   4.3. Logs y métricas en Azure Monitor.  
   4.4. Políticas de throttling y quotas en APIM.  

---

### 📄 Documentación

5. **Evidencias**
   5.1. Swagger/OpenAPI publicado por ambiente.  
   5.2. Ejemplos de consumo (Python, Power BI, Postman).  
   5.3. Matriz de clientes (tenant/domain → API Key).  


---

## Fase 18 — Capa RAG (Retrieval Augmented Generation)

**Objetivo**  
Implementar una capa de **explicaciones en lenguaje natural** soportada por bases vectoriales.  
Esta capa permite que usuarios y aplicaciones entiendan **el porqué** detrás de métricas, pronósticos y anomalías, usando documentos, dashboards y reglas como contexto.  
Cada ambiente tendrá índices separados, asegurando aislamiento por tenant/domain/env.

---

### ✅ Checklist General

1. **Diseño del pipeline RAG**
   1.1. Definir fuentes de conocimiento: dashboards, reportes, contratos de datos, runbooks.  
   1.2. Procesar y limpiar el contenido (tokenización, embeddings).  
   1.3. Crear índice vectorial separado por dominio y ambiente.  

2. **Base vectorial**
   2.1. Selección: OpenSearch Serverless, Aurora pgvector, Pinecone o equivalente.  
   2.2. Crear índices: `rag_<domain>_<env>`.  
   2.3. Configurar retención y políticas de actualización.  

3. **Embeddings**
   3.1. Elegir modelo embeddings (ej. OpenAI, Hugging Face).  
   3.2. Batch embeddings para documentos históricos.  
   3.3. Embeddings incrementales para nuevos documentos (on ingest).  

4. **API de consulta**
   4.1. Endpoint `/rag/explain` que combine query + contexto + respuesta.  
   4.2. Incluir citaciones de origen en la respuesta.  
   4.3. Validar seguridad: acceso solo con API Key por tenant/domain.  

---

### ✅ Checklist AWS

5. **AWS OpenSearch Serverless**
   5.1. Crear colección serverless por dominio/env.  
   5.2. Integrar con Lambda para consultas RAG.  
   5.3. Cifrado con KMS y auditoría con CloudTrail.  

---

### ✅ Checklist Azure

6. **Azure Cognitive Search (opcional)**
   6.1. Crear índice por dominio/env.  
   6.2. Integrar embeddings desde Azure OpenAI Service.  
   6.3. Configurar consultas híbridas (texto + vector).  

---

### 📄 Documentación

7. **Guías**
   7.1. Diagrama de arquitectura RAG.  
   7.2. Matriz de índices (`domain/env → colección`).  
   7.3. Evidencias de consultas con citaciones.  

---

## Fase 19 — Integración con Snowflake

**Objetivo**  
Integrar **Snowflake** como motor analítico centralizado, aprovechando **Snowpipe batch/streaming** y vistas sobre el Data Lake.  
Esto da flexibilidad para dominios que requieran análisis ad-hoc o consultas complejas, sin duplicar datos innecesariamente.

---

### ✅ Checklist General

1. **Bases y warehouses**
   1.1. Crear DBs por ambiente: `DB_DEV`, `DB_QA`, `DB_PROD`.  
   1.2. Crear esquemas por dominio (`oil_prod`, `retail_sales`).  
   1.3. Warehouses dedicados por ambiente con auto-suspend/resume.  

2. **Snowpipe Batch**
   2.1. Configurar stage externo a S3/ADLS (`stage_<env>`).  
   2.2. Pipes para ingestión automática desde `bronze` o `silver`.  
   2.3. Validar carga continua de archivos.  

3. **Snowpipe Streaming**
   3.1. Configurar streaming ingestion con Kafka/Event Hubs.  
   3.2. Crear streams por dominio.  
   3.3. Probar ingesta en tiempo real.  

4. **Modelado analítico**
   4.1. Crear vistas y modelos en `gold`.  
   4.2. Vistas canónicas (entidad/tiempo/valor).  
   4.3. Validar performance con clustering y micro-partitioning.  

---

### ✅ Checklist Seguridad y Costos

5. **Acceso y control**
   5.1. Roles por tenant/domain/env.  
   5.2. Mascaramiento dinámico de datos sensibles.  
   5.3. Etiquetas de costo en warehouses.  

---

### 📄 Documentación

6. **Evidencias**
   6.1. Guía de conexión a Snowflake desde BI/ML.  
   6.2. Matriz de roles y permisos.  
   6.3. Reportes de prueba de Snowpipe.  

---

## Fase 20 — Pruebas y cutover

**Objetivo**  
Validar todo el ecosistema antes de pasar a producción.  
Asegurar que pipelines, seguridad, BI, ML, APIs y RAG funcionan correctamente en **PROD**, con planes de rollback definidos.  
El cutover se ejecuta solo una vez cumplidas todas las validaciones.

---

### ✅ Checklist General

1. **Plan de pruebas**
   1.1. Definir smoke tests en DEV.  
   1.2. Pruebas end-to-end en QA.  
   1.3. Validaciones de SLA, seguridad y calidad en PROD.  

2. **Validaciones funcionales**
   2.1. Ingesta batch y streaming → validación en raw.  
   2.2. Transformaciones Spark → validación en bronze/silver/gold.  
   2.3. Reglas DQ en GE → sin fallos críticos.  
   2.4. Dashboards BI → datos correctos.  
   2.5. Modelos ML → métricas en rango esperado.  
   2.6. API/RAG → respuestas válidas con citaciones.  

3. **Checklist de readiness**
   3.1. Seguridad → MFA, roles, secretos.  
   3.2. DR → backups activos y probados.  
   3.3. Observabilidad → alarmas configuradas.  
   3.4. Costos → presupuestos y alertas por tenant.  

4. **Plan de rollback**
   4.1. Documentar escenarios de rollback (infra/pipelines/BI).  
   4.2. Procedimientos automatizados para revertir despliegues.  
   4.3. Evidencia de prueba de rollback en QA.  

---

### 📄 Documentación

5. **Cierre de pruebas**
   5.1. Acta de validación firmada por stakeholders.  
   5.2. Evidencias de pruebas ejecutadas.  
   5.3. Checklist de cutover completado.  

## Estrategia de ambientes, tenants y posible deshabilitación

- **Separación por defecto:**  
  Todo el ecosistema se diseña para operar en `DEV`, `QA` y `PROD` de forma **aislada**, tanto en red como en datos, APIs, BI y ML.  
  Cada **tenant** tendrá su propio espacio de datos, seguridad y costos, garantizando aislamiento total.

- **Multi-dominio parametrizable:**  
  La infraestructura base es **agnóstica**, preparada para cargar “Domain Packs” (ej. petróleo, retail, agro).  
  Los ambientes soportan múltiples dominios sin que se mezclen datos o reglas entre tenants.

- **Optimización futura:**  
  Una vez estabilizados los procesos, se puede **consolidar QA en DEV** o **pausarlo temporalmente** para reducir costos, manteniendo siempre **PROD** intacto y seguro.  
  Esta decisión se evaluará **por tenant y dominio**, según la madurez de sus pipelines.

- **Criterios para pausar QA:**  
  - Cobertura de pruebas automatizadas alta (unitarias, integración y regresión).  
  - *Feature flags* efectivos y trazabilidad de despliegues.  
  - *Canary releases* controlados en PROD.  
  - Monitoreo y alertas robustas por tenant/domain/env.  

- **Cómo pausar de forma segura:**  
  - Apagar recursos de cómputo de QA (clústeres, warehouses, endpoints).  
  - Congelar orquestadores y pipelines de QA.  
  - Mantener storage mínimo para evidencias/históricos.  
  - Retener configuración IaC para poder reactivar QA cuando se requiera.  
  - Comunicar a cada tenant/domain el impacto y procedimiento de reactivación.  
