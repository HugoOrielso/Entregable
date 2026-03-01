# 🚀 SpaceX Launches — Sistema Fullstack en AWS

Sistema completo que obtiene información de lanzamientos espaciales desde la API pública de SpaceX, los procesa mediante una función AWS Lambda, los almacena en DynamoDB y los visualiza a través de una aplicación web moderna desplegada en Amazon ECS Fargate.

---

## 🌐 URLs Públicas

| Servicio | URL |
|----------|-----|
| **Frontend** | http://spacex-alb-110258141.us-east-2.elb.amazonaws.com |
| **Backend API** | http://spacex-backend-alb-574561858.us-east-2.elb.amazonaws.com |
| **Swagger UI** | http://spacex-backend-alb-574561858.us-east-2.elb.amazonaws.com/api-docs |
| **Health Check** | http://spacex-backend-alb-574561858.us-east-2.elb.amazonaws.com/health |
| **Lambda URL** | https://x2j244r7gcqo4bljyuqnwifayi0ruvxm.lambda-url.us-east-2.on.aws/ |

---

## 📁 Repositorios

| Módulo | Repositorio |
|--------|------------|
| **Lambda** | https://github.com/HugoOrielso/lambda-handler |
| **Backend** | https://github.com/HugoOrielso/spacex-backend |
| **Frontend** | https://github.com/HugoOrielso/spacex-frontend |

---

## 🏗️ Arquitectura

```
EventBridge (cada 6h)
        │
        ▼
Lambda (Python 3.11) ──────► SpaceX API (v4/launches)
        │
        ▼
Amazon DynamoDB (spaces_launches_spacex-infra)
        │
        ▼
Backend API (Node.js + TypeScript / ECS Fargate)
        │
        ▼
Frontend React (ECS Fargate)

GitHub Actions (CI/CD)
├── Pipeline Lambda  → Test → Deploy a AWS Lambda
├── Pipeline Backend → Build → ECR → ECS Fargate
└── Pipeline Frontend → Build → ECR → ECS Fargate
```

### Componentes e interacción

| Componente | Tecnología | Responsabilidad |
|-----------|-----------|-----------------|
| Lambda | Python 3.11 | Extrae datos de SpaceX cada 6h y hace upsert en DynamoDB |
| DynamoDB | AWS DynamoDB | Almacena todos los lanzamientos espaciales |
| Backend | Node.js + Express + TypeScript | API REST que lee DynamoDB y expone los datos |
| Frontend | React | Visualiza los datos con filtros y gráficos |
| CloudFormation | `infrastructure.yml` | Define DynamoDB, Lambda, EventBridge, IAM Roles |
| GitHub Actions | 3 workflows | CI/CD automatizado para cada módulo |

---

## ☁️ Infraestructura en AWS

| Recurso | Nombre |
|---------|--------|
| CloudFormation Stack | `spacex-infra` |
| DynamoDB Table | `spaces_launches_spacex-infra` |
| Lambda Function | `spacex-launches-handler` |
| EventBridge Rule | `spacex-launches-every-6h` |
| IAM Role Lambda | `spacex-lambda-execution-role-spacex-infra` |
| IAM Role ECS | `ecsTaskExecutionRole-spacex-infra` |
| Secrets Manager | `spacex-backend-secrets` |
| ECS Cluster | `spacex-cluster` |
| ECS Service Backend | `spacex-backend-service` |
| ECR Repository | `spacex-backend` |
| Load Balancer Backend | `spacex-backend-alb` |
| Load Balancer Frontend | `spacex-alb` |

---

## 🚀 Despliegue desde cero

### Prerrequisitos

- AWS CLI v2 configurado (`aws configure`)
- Docker instalado
- Node.js 20+
- Python 3.11+
- Git

### Paso 1 — Clonar los repositorios

```bash
git clone https://github.com/HugoOrielso/lambda-handler
git clone https://github.com/HugoOrielso/spacex-backend
git clone https://github.com/HugoOrielso/spacex-frontend
```

### Paso 2 — Crear el secreto manualmente (solo la primera vez)

```bash
aws secretsmanager create-secret \
  --name spacex-backend-secrets \
  --secret-string '{"TABLE_NAME":"spaces_launches_spacex-infra","PORT":"4000"}' \
  --region us-east-2
```

### Paso 3 — Desplegar infraestructura con CloudFormation

```bash
cd lambda-handler
aws cloudformation deploy \
  --template-file infrastructure.yml \
  --stack-name spacex-infra \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides Environment=prod \
  --region us-east-2
```

### Paso 4 — Agregar permisos DynamoDB al rol ECS

```bash
aws iam put-role-policy \
  --role-name ecsTaskExecutionRole-spacex-infra \
  --policy-name DynamoDBFullAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["dynamodb:Scan","dynamodb:Query","dynamodb:GetItem","dynamodb:DescribeTable"],
      "Resource": [
        "arn:aws:dynamodb:us-east-2:148761674962:table/spaces_launches_spacex-infra",
        "arn:aws:dynamodb:us-east-2:148761674962:table/spaces_launches_spacex-infra/index/*"
      ]
    }]
  }'
```

### Paso 5 — Obtener ARN del secreto y actualizar task-definition

```bash
SECRET_ARN=$(aws secretsmanager describe-secret \
  --secret-id spacex-backend-secrets \
  --region us-east-2 \
  --query 'ARN' --output text)

cd spacex-backend
sed -i "s|spacex-backend-secrets-[A-Za-z0-9]*|${SECRET_ARN##*:secret:}|g" task-definition.json
```

### Paso 6 — Registrar task-definition y desplegar backend

```bash
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json \
  --region us-east-2

aws ecs update-service \
  --cluster spacex-cluster \
  --service spacex-backend-service \
  --task-definition spacex-backend-task \
  --force-new-deployment \
  --region us-east-2
```

### Paso 7 — Desplegar Lambda con GitHub Actions

Haz push a `main` en el repositorio `lambda-handler` o dispara el workflow manualmente desde GitHub Actions. El pipeline corre los tests y despliega automáticamente.

### Paso 8 — Verificar el sistema

```bash
curl http://spacex-backend-alb-574561858.us-east-2.elb.amazonaws.com/health
curl http://spacex-backend-alb-574561858.us-east-2.elb.amazonaws.com/launches
```

> Para el proceso completo de redespliegue desde cero ver [`lambda-handler/cloudformation.md`](https://github.com/HugoOrielso/lambda-handler).

---

## 🧪 Correr pruebas localmente

### Lambda (Python)

```bash
cd lambda-handler

# Instalar dependencias
python -m pip install -r requirements-dev.txt

# Correr tests con cobertura
python -m pytest test/test_lambda_function.py -v \
  --cov=lambda_function \
  --cov-report=term-missing \
  --cov-fail-under=80
```

**Resultado esperado:** 17 tests pasando, 97.75% de cobertura ✅

### Backend (Node.js)

```bash
cd spacex-backend
pnpm install
pnpm test
```

### Verificar backend localmente con Docker

```bash
cd spacex-backend
docker build -t spacex-backend .
docker run -p 4000:4000 \
  -e TABLE_NAME=spaces_launches_spacex-infra \
  -e AWS_DEFAULT_REGION=us-east-2 \
  -e AWS_ACCESS_KEY_ID=<tu-key> \
  -e AWS_SECRET_ACCESS_KEY=<tu-secret> \
  spacex-backend

curl http://localhost:4000/health
curl http://localhost:4000/launches
```

---

## 🔁 Pipeline CI/CD

### Pipeline Lambda (`lambda-handler/.github/workflows/deploy-lambda.yml`)

Se activa con push a `main` que modifique `lambda_function.py`, `requirements.txt` o `test/**`.

```
Push a main
    │
    ▼
JOB: test
  → Setup Python 3.11
  → pip install requirements + dev
  → pytest con cobertura mínima 80%
        │ (solo si pasa)
        ▼
JOB: deploy
  → Configurar AWS CLI
  → Build .zip del código
  → aws lambda update-function-code
  → Verificar despliegue
```

### Pipeline Backend (`spacex-backend/.github/workflows/deploy-backend.yml`)

Se activa con push a `main`.

```
Push a main
    │
    ▼
  → Login a Amazon ECR
  → Build imagen Docker
  → Push imagen a ECR
  → Actualizar task definition
  → Deploy en ECS Fargate
  → Esperar estabilización
```

### Pipeline Frontend (`spacex-frontend/.github/workflows/deploy-frontend.yml`)

Mismo flujo que el backend — Build → ECR → ECS Fargate.

### Secrets requeridos en GitHub

Configurar en **Settings → Secrets and variables → Actions** en cada repositorio:

| Secret | Valor |
|--------|-------|
| `AWS_ACCESS_KEY_ID` | Access Key del usuario IAM |
| `AWS_SECRET_ACCESS_KEY` | Secret Key del usuario IAM |
| `AWS_REGION` | `us-east-2` |

### Extender el pipeline

Para agregar nuevos pasos al pipeline basta con editar el archivo `.github/workflows/deploy-*.yml` correspondiente. Por ejemplo, para agregar notificaciones de Slack al finalizar el deploy:

```yaml
- name: Notify Slack
  if: always()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
```

---

## 🔌 Endpoints del Backend

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/health` | Estado del servicio |
| GET | `/launches` | Lista todos los lanzamientos (filtros: status, year, search, limit, offset) |
| GET | `/launches/:id` | Detalle de un lanzamiento |
| GET | `/stats/summary` | Resumen estadístico (total, success, failed, upcoming) |
| GET | `/stats/by-year` | Estadísticas agrupadas por año |
| GET | `/api-docs` | Swagger UI interactivo |

### Ejemplo de respuesta `/launches`

```json
[
  {
    "launch_id": "5eb87cd9ffd86e000604b32a",
    "mission_name": "FalconSat",
    "date_utc": "2006-03-24T22:30:00.000Z",
    "status": "failed",
    "rocket_id": "5e9d0d95eda69955f709d1eb",
    "details": "Engine failure at 33 seconds..."
  }
]
```

---

## 🗄️ Esquema DynamoDB

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `launch_id` | String (PK) | ID único del lanzamiento |
| `mission_name` | String | Nombre de la misión |
| `flight_number` | Number | Número de vuelo |
| `date_utc` | String (GSI) | Fecha UTC del lanzamiento |
| `status` | String | upcoming / success / failed / unknown |
| `rocket_id` | String | ID del cohete |
| `launchpad_id` | String | ID de la plataforma |
| `details` | String | Descripción del lanzamiento |
| `patch_small` | String | URL del parche de la misión |
| `article` | String | Link al artículo |
| `webcast` | String | Link al webcast |

---

## 📖 Documentación adicional

| Documento | Descripción |
|-----------|-------------|
| [`lambda-handler/README.md`](https://github.com/HugoOrielso/lambda-handler) | Detalle de la Lambda, tests y pipeline |
| [`lambda-handler/cloudformation.md`](https://github.com/HugoOrielso/lambda-handler) | Guía completa de despliegue de infraestructura |
| [`spacex-backend/README.md`](https://github.com/HugoOrielso/spacex-backend) | Detalle del backend, task-definition y ECS |
| [`spacex-frontend/README.md`](https://github.com/HugoOrielso/spacex-frontend) | Detalle del frontend y despliegue |

---

*Go efficient, happy, and green. 🚀*