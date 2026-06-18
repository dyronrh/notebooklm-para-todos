# Documentación de Infraestructura como Código
## Terraform — MegaTienda.com AWS Infrastructure
### Versión: 3.1 | Equipo: Platform Engineering

---

## ESTRUCTURA DEL REPOSITORIO

```
infrastructure/
├── environments/
│   ├── production/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── development/
│       └── ...
├── modules/
│   ├── networking/          # VPC, subnets, security groups
│   ├── eks/                 # Cluster EKS + node groups
│   ├── aurora/              # Bases de datos Aurora PostgreSQL
│   ├── elasticache/         # Clusters Redis
│   ├── alb/                 # Application Load Balancers
│   ├── cloudfront/          # Distribuciones CDN
│   ├── waf/                 # Web Application Firewall
│   └── monitoring/          # CloudWatch dashboards y alarmas
├── .terraform-version       # 1.7.3
├── Makefile                 # Comandos habituales
└── README.md
```

---

## MÓDULO: NETWORKING

### Descripción
Crea la VPC, subnets (públicas y privadas), Internet Gateway, NAT Gateways y tablas de rutas.

### Variables

| Variable | Tipo | Default | Descripción |
|---|---|---|---|
| `vpc_cidr` | string | "10.0.0.0/16" | CIDR block de la VPC |
| `az_count` | number | 3 | Número de Availability Zones |
| `enable_nat_gateway` | bool | true | Crear NAT Gateway |
| `single_nat_gateway` | bool | false | Un solo NAT vs. uno por AZ |
| `environment` | string | — | Tag: production, staging, dev |

### Outputs

| Output | Descripción |
|---|---|
| `vpc_id` | ID de la VPC creada |
| `private_subnet_ids` | Lista de IDs de subnets privadas |
| `public_subnet_ids` | Lista de IDs de subnets públicas |
| `nat_gateway_ips` | IPs de los NAT Gateways |

### Uso

```hcl
module "networking" {
  source = "../../modules/networking"

  vpc_cidr           = "10.0.0.0/16"
  az_count           = 3
  enable_nat_gateway = true
  single_nat_gateway = false  # Un NAT por AZ para HA
  environment        = var.environment

  tags = {
    Project     = "megatienda"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

### Decisiones de diseño

- **Por qué 3 AZs**: Requerimiento de disponibilidad 99.95%. Con 2 AZs, la pérdida de una AZ impacta el 50% de la capacidad. Con 3, solo el 33%.
- **Por qué NAT Gateway por AZ**: Un solo NAT Gateway es un SPOF. Si la AZ del NAT falla, los recursos en otras AZs pierden acceso a internet. El costo adicional (~$32/mes por NAT extra) justifica la redundancia.
- **CIDR /16**: Permite hasta 65.536 IPs, suficiente para crecimiento a 5 años sin necesidad de re-IP.

---

## MÓDULO: EKS

### Descripción
Crea el cluster EKS con sus node groups (spot y on-demand), configuración de IRSA (IAM Roles for Service Accounts), y addons del cluster.

### Variables principales

| Variable | Tipo | Default | Descripción |
|---|---|---|---|
| `cluster_name` | string | — | Nombre del cluster EKS |
| `cluster_version` | string | "1.29" | Versión de Kubernetes |
| `spot_instance_types` | list | ["m6i.xlarge", "m6a.xlarge"] | Tipos de instancias spot |
| `on_demand_instance_types` | list | ["m6i.2xlarge"] | Instancias on-demand para workloads críticos |
| `spot_min_size` | number | 3 | Mínimo de nodos spot |
| `spot_max_size` | number | 30 | Máximo de nodos spot |
| `on_demand_min_size` | number | 3 | Mínimo de nodos on-demand |
| `on_demand_max_size` | number | 10 | Máximo de nodos on-demand |

### Uso

```hcl
module "eks" {
  source = "../../modules/eks"

  cluster_name    = "megatienda-${var.environment}"
  cluster_version = "1.29"
  vpc_id          = module.networking.vpc_id
  subnet_ids      = module.networking.private_subnet_ids

  spot_instance_types    = ["m6i.xlarge", "m6a.xlarge", "m5.xlarge"]
  on_demand_instance_types = ["m6i.2xlarge"]

  spot_min_size      = 3
  spot_max_size      = 30
  on_demand_min_size = 3
  on_demand_max_size = 10

  # Addons del cluster
  enable_cluster_autoscaler    = true
  enable_aws_load_balancer_controller = true
  enable_external_secrets      = true
  enable_metrics_server        = true

  environment = var.environment
}
```

### Estrategia de nodos: Spot vs. On-Demand

**Nodos Spot** (70% del cómputo en condiciones normales):
- Costo: 60-70% menor que on-demand
- Riesgo: AWS puede reclamarlos con 2 minutos de aviso
- Mitigación: 3+ tipos de instancia para maximizar disponibilidad de spot; Cluster Autoscaler migra pods automáticamente
- **No** corren aquí: bases de datos stateful, PVCs con datos críticos

**Nodos On-Demand** (30% del cómputo):
- Corren: Prometheus, Grafana, Alertmanager, cert-manager, external-secrets
- También reciben overflow cuando los spot se agotan
- Garantizan que el control plane de monitoreo nunca dependa de nodos spot

---

## MÓDULO: AURORA POSTGRESQL

### Descripción
Crea el cluster Aurora PostgreSQL Multi-AZ con instancias de escritura y lectura.

### Uso

```hcl
module "aurora" {
  source = "../../modules/aurora"

  cluster_identifier = "megatienda-${var.environment}"
  engine_version     = "15.4"
  database_name      = "megatienda"

  # Instancias
  writer_instance_class  = "db.r7g.2xlarge"
  reader_instance_count  = 2
  reader_instance_class  = "db.r7g.xlarge"

  # Red
  vpc_id     = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids

  # Acceso
  allowed_security_group_ids = [
    module.eks.node_security_group_id,
    module.bastion.security_group_id
  ]

  # Backup
  backup_retention_period = 7
  preferred_backup_window = "03:00-04:00"  # UTC, 23:00-00:00 Argentina

  # Encriptación
  storage_encrypted = true
  kms_key_id        = module.kms.aurora_key_arn

  environment = var.environment
}
```

### Outputs

```hcl
output "aurora_writer_endpoint" {
  value = module.aurora.writer_endpoint
}

output "aurora_reader_endpoint" {
  value = module.aurora.reader_endpoint
}

output "aurora_port" {
  value = 5432
}
```

---

## FLUJO DE TRABAJO CON TERRAFORM

### Inicializar el proyecto

```bash
# Desde el directorio del environment
cd environments/production

# Inicializar backend de S3 y descargar módulos
terraform init

# Verificar que está conectado al backend correcto
terraform workspace show  # → production
```

### Planificar cambios

```bash
# Plan estándar
terraform plan -out=tfplan

# Plan para un módulo específico
terraform plan -target=module.aurora -out=tfplan

# Ver el plan en formato JSON (para scripts)
terraform show -json tfplan | jq '.resource_changes[] | select(.change.actions != ["no-op"])'
```

### Aplicar cambios

```bash
# Aplicar el plan guardado (recomendado para producción)
terraform apply tfplan

# Aplicar con aprobación automática (SOLO en CI/CD con revisión previa)
terraform apply -auto-approve tfplan
```

### Destruir recursos (PELIGROSO)

```bash
# NUNCA ejecutar en producción sin aprobación explícita del CTO
terraform destroy -target=module.aurora  # Destruye la base de datos
```

**Protecciones contra destrucción accidental**:
- Los recursos críticos tienen `lifecycle { prevent_destroy = true }` en el código
- El backend de S3 tiene bucket versioning + MFA delete habilitado
- La cuenta de AWS de producción requiere MFA para operaciones destructivas

---

## BACKEND Y STATE MANAGEMENT

### Configuración del backend (S3 + DynamoDB)

```hcl
terraform {
  backend "s3" {
    bucket         = "megatienda-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:ACCOUNT_ID:key/KEY_ID"
    dynamodb_table = "megatienda-terraform-locks"
  }
}
```

### Estado bloqueado

Si Terraform muestra "State is locked", significa que otro proceso está aplicando cambios:

```bash
# Ver quién tiene el lock
aws dynamodb get-item \
  --table-name megatienda-terraform-locks \
  --key '{"LockID": {"S": "megatienda-terraform-state/production/terraform.tfstate"}}'

# Liberar el lock (solo si estás seguro que el proceso anterior falló)
terraform force-unlock <LOCK_ID>
```

---

## CHECKLIST PREVIO A APPLY EN PRODUCCIÓN

- [ ] El plan fue revisado por al menos una persona del equipo (PR aprobado)
- [ ] El plan no destruye recursos críticos (buscar `destroy` en el output del plan)
- [ ] Hay un backup reciente de la base de datos (< 4 horas)
- [ ] El deploy no es en horario pico (evitar 12:00-14:00 y 19:00-22:00 Argentina)
- [ ] PagerDuty está configurado para el on-call del turno
- [ ] El canal #deployments en Slack fue notificado

---

*Sube este documento junto con el runbook de Kubernetes a NotebookLM y pregunta: "¿Por qué se usa un NAT Gateway por AZ y no uno solo?" o "¿Cuál es el checklist antes de aplicar cambios en producción?" o "Explica la diferencia entre nodos spot y on-demand en la arquitectura."*
