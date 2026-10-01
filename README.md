# AWS BDA --> MVP (Producto Minimo Viable)

Repositorio SDD (Spec-Driven Development) para desarrollar, validar y desplegar el MVP hibrido **BDA + SPA** .

## Proposito

Implementar el MVP para carga controlada de Excel, conversion XLS -> PDF, extraccion con Amazon Bedrock Data Automation (BDA), persistencia de JSON en S3, normalizacion hacia Aurora PostgreSQL Serverless v2 y operacion funcional mediante una SPA.

El flujo tecnico objetivo conserva el patron base:

```
S3 raw-xls/ -> EventBridge -> Step Functions -> Lambda -> BDA -> S3 json-result-bp/ -> Aurora
```

Prefijos S3 utilizados: `raw-xls/`, `converted-pdfs/`, `json-result-bp/`, `control-json/`, `dead-letter/`.

## Arquitectura

Definida como IaC con AWS SAM en `infrastructure/template.yaml`.

### Ingesta y orquestacion

- **S3 (`ca-bda-data-lake-prod`)** - Data lake con prefijos `raw-xls/`, `converted-pdfs/`, `json-result-bp/`, `control-json/`, `dead-letter/`. Pre-provisionado fuera del stack y referenciado por parametro; cifrado con KMS (`DataLakeKmsKeyArn`).
- **EventBridge (`ca-bda-objectcreated-rule`)** - Detecta `Object Created` con prefijo `raw-xls/` y arranca la state machine via `ca-bda-eventbridge-sfn-role`.
- **Step Functions (`ca-bda-extraction-state-machine`, STANDARD)** - Orquesta el pipeline con un Map State (concurrencia por ambiente) y enruta errores a `log-error` via `Catch`.

### Lambdas (runtime Python 3.12)

- `ca-bda-prepare-document` - Prepara el documento y calcula los items del Map State. **Se despliega fuera del stack** como imagen Docker en ECR (no esta en el template para evitar conflictos). Para soporte de 100+ hojas se recomienda `Timeout=900`, `MemorySize=2048`.
- `ca-bda-invoke-bda-lambda` - Invoca `InvokeDataAutomationAsync` y consulta `GetDataAutomationStatus` (usa el inference profile `us.data-automation-v1`).
- `ca-bda-validate-persist-lambda` - Valida confidence por campo contra el umbral (0.80), aplica acciones (`reject_chunk`, `send_to_human_review`, `flag_and_null`) y persiste en Aurora en una transaccion atomica. Corre en VPC.
- `ca-bda-log-error-lambda` - Registra errores del pipeline y escribe en `dead-letter/`. Corre en VPC.
- `ca-bda-spa-api-lambda` - Backend HTTP de la SPA (presigned URLs sobre `raw-xls/`, consultas de estado). Corre en VPC.
- **Lambda Layer `SharedUtilsLayer`** - Utilidades comunes (`db_utils`, `logger`, `metrics`) desde `lambdas/shared/`.

### Datos, API y frontend

- **API Gateway HTTP API (`ca-bda-spa-http-api`, stage `v1`)** - Integracion Lambda proxy (`/{proxy+}`) hacia `spa-api`, con CORS restringido al dominio de CloudFront.
- **Aurora PostgreSQL Serverless v2** - Esquema `ca_bda` con tablas `files`, `sheets`, `extractions`, `extraction_arrays`, `inconsistencies`, `audit_log`, `error_log`. Acceso desde Lambdas en VPC (subnets privadas + security group 5432).
- **CloudFront + S3 (`ca-bda-spa-web-bucket`)** - Hosting privado de la SPA con Origin Access Control (`ca-bda-spa-oac`) y bucket policy para el principal de CloudFront. SPA fallback a `index.html` en 403/404.
- **Secrets Manager** - Credenciales de Aurora via `DB_SECRET_ARN` (descifrado condicionado por `kms:ViaService`).

### Observabilidad

- **CloudWatch** - Logging estructurado JSON y metricas personalizadas en el namespace `CaBda` (dimension `Environment`).
- **Dashboard `ca-bda-dashboard`** - Files procesados, estado de ejecuciones de Step Functions, latencia y errores de BDA, errores por Lambda e inconsistencias por grupo de campos.
- **Alarms** - `ca-bda-alarm-bda-errors` (BdaErrors > 5/h) y `ca-bda-alarm-sfn-failures` (ejecuciones fallidas).
- **SNS `ca-bda-alarm-notifications`** - Topic destino de las alarmas (Alarm/OK actions).

> Nota: el `template.yaml` incluye un topic SNS unicamente para notificaciones de alarmas de CloudWatch. Esto no forma parte del pipeline de datos, que se mantiene S3 -> EventBridge -> Step Functions -> Lambda/BDA -> Aurora (sin SQS/SNS/DynamoDB/Kinesis en el flujo).

## Estructura del repositorio

```
.
|- AGENTS.md                # Reglas obligatorias del agente
|- infrastructure/          # IaC SAM (template.yaml, samconfig.toml, policies, tags, parameters)
|- lambdas/                 # Codigo de las funciones Lambda y utilidades compartidas
|  |- prepare-document/     # Imagen Docker (ECR), desplegada fuera del stack
|  |- invoke-bda/
|  |- validate-persist/
|  |- log-error/
|  |- spa-backend/
|  \- shared/               # Layer compartido: db_utils, logger, metrics
|- step-functions/          # Definicion ASL de la maquina de estados y sample-inputs
|- eventbridge/             # Configuracion de reglas de EventBridge
|- s3-buckets/              # Configuracion de buckets y prefijos
|- api-gateway/             # Configuracion del HTTP API y ejemplos
|- aurora/                  # Migraciones, esquemas, vistas y MER
|- bda/                     # Blueprints de BDA (patron-blueprint)
|- frontend-spa/            # SPA React + TypeScript + Vite
|- docs/                    # Diseno, arquitectura, referencias y reglas de negocio
|- scripts/                 # Scripts operativos
|- tests/                   # Pruebas del proyecto
|- requirements.txt         # Dependencias Python
|- pytest.ini / conftest.py # Configuracion de pruebas
\- run_tests.py
```

## Prerrequisitos

- Windows PowerShell 7+
- AWS CLI v2
- AWS SAM CLI
- Python 3.12
- Node.js 20 LTS
- Git
- Perfil AWS SSO: `admin_ca`
- Cuenta AWS con permisos para S3, Lambda, Step Functions, EventBridge, API Gateway, Aurora, CloudFront, IAM, KMS, Secrets Manager, SNS y CloudWatch.

## Configuracion inicial (PowerShell)

```powershell
winget install Amazon.AWSCLI
winget install Amazon.SAM-CLI
winget install Python.Python.3.12
winget install OpenJS.NodeJS.LTS
winget install Git.Git

aws configure sso --profile admin_ca
aws sts get-caller-identity --profile admin_ca

$env:AWS_PROFILE="admin_ca"
$env:AWS_REGION="us-east-1"
$env:ENVIRONMENT="dev"
```

## Dependencias Python

Instalar en un entorno virtual:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

Paquetes principales: `boto3`, `aws-lambda-powertools`, `pydantic`, `psycopg[binary]`, `openpyxl`, `pandas`, `pytest`, `moto`.

## Base de datos (Aurora)

Las migraciones viven en `aurora/migrations/`. Ejecutarlas con:

```powershell
python aurora/run_migrations.py
```

Las credenciales se obtienen exclusivamente desde Secrets Manager mediante `DB_SECRET_ARN`. No se hardcodean host, puerto ni credenciales.

## Frontend SPA

Stack: React 19 + TypeScript + Vite. Pruebas con Vitest y fast-check (property-based testing).

```powershell
cd frontend-spa
npm install
npm run build     # build de produccion
npm run test      # pruebas unitarias
```

El servidor de desarrollo (`npm run dev`) debe ejecutarse manualmente en una terminal.

La SPA no se conecta directamente a Aurora; opera via API Gateway HTTP API + Lambda backend. Valida la extension `.xls/.xlsx` antes de solicitar la presigned URL. Maximo estimado por batch de carga masiva: 200 PDFs.

## Despliegue (SAM)

El stack requiere parametros de infraestructura pre-existente (buckets, KMS, VPC, ARNs de BDA y del secreto de Aurora). Revisa `infrastructure/parameters/` y `infrastructure/samconfig.toml`.

```powershell
sam build --template infrastructure/template.yaml
sam deploy --config-file infrastructure/samconfig.toml --profile bigcheese_admin_ca
```

Parametros clave del template:

- `Environment` (`dev` | `stg` | `prod`) - controla retencion de logs, concurrencia del Map State y nivel de log.
- `RawBucketName`, `SpaWebBucketName` - buckets S3 pre-provisionados.
- `DbSecretArn` - secreto de Aurora en Secrets Manager.
- `BdaProjectArn`, `BdaBlueprintArn`, `BdaInferenceProfileName` - configuracion de BDA.
- `DataLakeKmsKeyArn` - KMS key del data lake (gestionada fuera del stack).
- `VpcSubnetIds`, `VpcSecurityGroupId` - red privada para Lambdas con acceso a Aurora.
- `PrepareDocumentImageUri` - imagen ECR de `prepare-document`.

## Pruebas

```powershell
python run_tests.py
# o directamente con pytest
pytest
```

## Convenciones

- Todo recurso AWS inicia con el prefijo `ca-bda-`.
- Todo recurso desplegable incluye tags: `Owner`, `Project`, `EOL`, `Environment`, `CostCenter`.
- Cada componente mantiene un `README.md` con proposito, recursos AWS, variables, IAM, dependencias, despliegue y smoke test.
- El pipeline de datos no usa SQS, DynamoDB ni Kinesis, ni Lambda starter (el disparo es S3 -> EventBridge -> Step Functions directo). SNS se usa unicamente para notificaciones de alarmas CloudWatch.

## Documentacion

- `docs/` - Diseno de arquitectura, referencias y reglas de negocio.
- `AGENTS.md` - Reglas obligatorias del agente.
- READMEs por componente en `infrastructure/`, `lambdas/`, `step-functions/`, `eventbridge/`, `api-gateway/`, `aurora/` y `frontend-spa/`.
