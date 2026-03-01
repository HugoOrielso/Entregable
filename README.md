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

## 📂 Estructura del proyecto

### Lambda (`lambda-handler`)
```
lambda-handler/
├── .github/                  # Workflows de GitHub Actions
├── test/                     # Tests unitarios con pytest
│   └── test_lambda_function.py
├── venv/                     # Entorno virtual Python
├── cloudformation.md         # Guía completa de despliegue
├── conftest.py               # Configuración de pytest
├── infrastructure.yml        # CloudFormation (DynamoDB, Lambda, IAM, EventBridge)
├── lambda_function.py        # Código principal de la Lambda
├── requirements.txt          # Dependencias de producción
└── requirements-dev.txt      # Dependencias de desarrollo/testing
```

### Backend (`spacex-backend`)
```
spacex-backend/
├── .github/                  # Workflows de GitHub Actions
├── src/
│   ├── controllers/          # Controladores Express
│   ├── database/
│   │   └── db.ts             # Cliente DynamoDB y queries
│   ├── docs/
│   │   └── swagger.ts        # Configuración de Swagger
│   ├── lib/
│   │   └── types.ts          # Tipos TypeScript
│   └── routes/
│       └── launches.routes.ts # Definición de rutas
├── test/
│   └── launches.controller.test.ts
├── index.ts                  # Entry point
├── server.ts                 # Configuración Express
├── Dockerfile                # Imagen Docker
├── task-definition.json      # ECS Task Definition
├── arquitectura.md           # Documentación de arquitectura
└── package.json
```

### Frontend (`spacex-frontend`)
```
spacex-frontend/
├── .github/                  # Workflows de GitHub Actions
├── src/
│   ├── app/
│   │   ├── launches/         # Página de lanzamientos
│   │   ├── layout.tsx        # Layout principal
│   │   └── page.tsx          # Página principal
│   ├── components/
│   │   ├── common/           # Componentes reutilizables
│   │   ├── Launches/         # Componentes de lanzamientos
│   │   └── main/             # Componentes principales
│   ├── lib/
│   │   ├── api/              # Llamadas al backend
│   │   ├── types/            # Tipos TypeScript
│   │   └── ui-helper.ts      # Helpers de UI
│   └── store/                # Estado global
├── public/                   # Assets estáticos
├── Dockerfile                # Imagen Docker
├── next.config.ts            # Configuración Next.js
└── package.json
```

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
Frontend Next.js (ECS Fargate)

GitHub Actions (CI/CD)
├── Pipeline Lambda  → Test → Deploy a AWS Lambda
├── Pipeline Backend → Build → ECR → ECS Fargate
└── Pipeline Frontend → Build → ECR → ECS Fargate
```

### Cómo interactúan los componentes

| Componente | Tecnología | Responsabilidad |
|-----------|-----------|-----------------|
| Lambda | Python 3.11 | Extrae datos de SpaceX cada 6h y hace upsert en DynamoDB |
| DynamoDB | AWS DynamoDB | Almacena todos los lanzamientos espaciales |
| Backend | Node.js + Express + TypeScript | API REST que lee DynamoDB y expone los datos |
| Frontend | Next.js + React | Visualiza los datos con filtros y gráficos |
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

> Guía completa con todos los pasos detallados disponible en [`lambda-handler/cloudformation.md`](https://github.com/HugoOrielso/lambda-handler)

### Prerrequisitos

- AWS CLI v2 configurado (`aws configure`)
- Docker instalado
- Node.js 20+
- Python 3.11+

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

### Paso 5 — Actualizar task-definition y desplegar backend

```bash
cd spacex-backend

# Obtener ARN del secreto y actualizar task-definition
SECRET_ARN=$(aws secretsmanager describe-secret \
  --secret-id spacex-backend-secrets \
  --region us-east-2 \
  --query 'ARN' --output text)
sed -i "s|spacex-backend-secrets-[A-Za-z0-9]*|${SECRET_ARN##*:secret:}|g" task-definition.json

# Registrar y desplegar
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

### Paso 6 — Desplegar Lambda y Frontend via GitHub Actions

Haz push a `main` en cada repositorio o dispara los workflows manualmente desde GitHub Actions.

### Paso 7 — Verificar

```bash
curl http://spacex-backend-alb-574561858.us-east-2.elb.amazonaws.com/health
curl http://spacex-backend-alb-574561858.us-east-2.elb.amazonaws.com/launches
```

---

## 🧪 Correr pruebas localmente

### Lambda (Python)

```bash
cd lambda-handler
python -m pip install -r requirements-dev.txt
python -m pytest test/test_lambda_function.py -v \
  --cov=lambda_function \
  --cov-report=term-missing \
  --cov-fail-under=80
```

**Resultado:** 17 tests pasando, 97.75% de cobertura ✅

### Backend (Node.js + TypeScript)

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

### Pipeline Lambda

```
Push a main (lambda-handler)
    │
    ▼
JOB: test
  → Setup Python 3.11
  → pip install requirements + dev
  → pytest con cobertura mínima 80%
        │
        ▼
JOB: deploy
  → Configurar AWS CLI
  → Build .zip del código
  → aws lambda update-function-code
```

### Pipeline Backend

```
Push a main (spacex-backend)
    │
    ▼
  → Login a Amazon ECR
  → Build imagen Docker
  → Push imagen a ECR
  → Actualizar task definition
  → Deploy en ECS Fargate
  → Esperar estabilización
```

### Pipeline Frontend

```
Push a main (spacex-frontend)
    │
    ▼
  → Login a Amazon ECR
  → Build imagen Docker (Next.js)
  → Push imagen a ECR
  → Deploy en ECS Fargate
```

### Secrets requeridos en GitHub (cada repo)

| Secret | Valor |
|--------|-------|
| `AWS_ACCESS_KEY_ID` | Access Key del usuario IAM |
| `AWS_SECRET_ACCESS_KEY` | Secret Key del usuario IAM |
| `AWS_REGION` | `us-east-2` |

---

## 🔌 Endpoints del Backend

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/health` | Estado del servicio |
| GET | `/launches` | Lista lanzamientos (filtros: status, year, search, limit, offset) |
| GET | `/launches/:id` | Detalle de un lanzamiento |
| GET | `/stats/summary` | Resumen estadístico total/success/failed/upcoming |
| GET | `/stats/by-year` | Estadísticas agrupadas por año |
| GET | `/api-docs` | Swagger UI interactivo |

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
| `launchpad_id` | String | ID de la plataforma de lanzamiento |
| `details` | String | Descripción del lanzamiento |
| `patch_small` | String | URL del parche de la misión |
| `article` | String | Link al artículo |
| `webcast` | String | Link al webcast |

---

## 📖 Documentación adicional

| Documento | Descripción |
|-----------|-------------|
| [`lambda-handler/cloudformation.md`](https://github.com/HugoOrielso/lambda-handler) | Guía completa de despliegue de infraestructura y redeploy desde cero |
| [`spacex-backend/arquitectura.md`](https://github.com/HugoOrielso/spacex-backend) | Documentación de arquitectura del backend |
| [`spacex-backend/README.md`](https://github.com/HugoOrielso/spacex-backend) | Detalle del backend y ECS |
| [`lambda-handler/README.md`](https://github.com/HugoOrielso/lambda-handler) | Detalle de la Lambda y tests |
| [`spacex-frontend/README.md`](https://github.com/HugoOrielso/spacex-frontend) | Detalle del frontend |

---

*Go efficient, happy, and green. 🚀*