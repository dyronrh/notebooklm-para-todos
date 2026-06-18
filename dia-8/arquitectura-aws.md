# Arquitectura de Infraestructura Cloud
## Plataforma E-Commerce — MegaTienda.com
### Documento de Arquitectura de Referencia (AWS)

**Versión**: 2.4 | **Fecha**: Noviembre 2024
**Equipo**: Platform Engineering | **Contacto**: plataforma@megatienda.com

---

## RESUMEN EJECUTIVO

Este documento describe la arquitectura de infraestructura AWS de MegaTienda.com, plataforma de e-commerce con 2.8 millones de usuarios activos mensuales y picos de tráfico de hasta 12.000 requests por segundo durante eventos especiales (Black Friday, Cyber Monday).

**Objetivos de arquitectura**:
- **Disponibilidad**: 99.95% uptime (máximo 4.38 horas de downtime anual)
- **Latencia**: P99 < 200ms para API de catálogo; P99 < 500ms para checkout
- **Escalabilidad**: Capacidad de escalar 10x en 15 minutos ante picos de tráfico
- **Seguridad**: Cumplimiento PCI-DSS Level 1 (procesamiento de pagos)
- **Recuperación**: RPO < 15 minutos, RTO < 30 minutos

**Costo mensual estimado** (en producción): USD 47.800/mes (USD 573.600/año)

---

## DIAGRAMA DE ARQUITECTURA

```
                          ┌─────────────────────────────────────┐
                          │           INTERNET                  │
                          └──────────────┬──────────────────────┘
                                         │
                          ┌──────────────▼──────────────────────┐
                          │         Amazon CloudFront           │
                          │    (CDN Global - 450+ edge nodes)   │
                          └──────────────┬──────────────────────┘
                                         │
                          ┌──────────────▼──────────────────────┐
                          │         AWS WAF + Shield            │
                          │  (DDoS protection + reglas OWASP)   │
                          └──────────────┬──────────────────────┘
                                         │
              ┌──────────────────────────▼──────────────────────────┐
              │                 Application Load Balancer            │
              │                    (Multi-AZ)                        │
              └────┬──────────────────────────────────────┬─────────┘
                   │                                      │
     ┌─────────────▼─────────────┐          ┌────────────▼────────────┐
     │      ECS Fargate          │          │      ECS Fargate         │
     │   Frontend (Next.js)      │          │   Backend API (Node.js)  │
     │   Auto Scaling: 3-50      │          │   Auto Scaling: 5-100    │
     └─────────────┬─────────────┘          └────────────┬────────────┘
                   │                                      │
                   │              ┌───────────────────────┤
                   │              │                       │
     ┌─────────────▼──────┐  ┌───▼──────────┐  ┌────────▼────────┐
     │   ElastiCache       │  │  Aurora       │  │   Amazon SQS    │
     │   Redis Cluster     │  │  PostgreSQL   │  │   (async jobs)  │
     │   (session cache)   │  │  Multi-AZ     │  └────────┬────────┘
     └────────────────────┘  └───────────────┘           │
                                                  ┌───────▼─────────┐
                                                  │  Lambda Workers  │
                                                  │  (email, notify) │
                                                  └─────────────────┘
```

---

## COMPONENTES DETALLADOS

### 1. Red de Distribución de Contenido (CloudFront)

**Configuración**:
- 23 distribuciones activas (1 por dominio/subdominio)
- Origins: S3 (assets estáticos), ALB (contenido dinámico)
- Cache behavior: TTL de 365 días para assets con hash, 0 para HTML dinámico
- Geo-restriction: bloqueados 15 países según política de compliance
- Custom error pages para 403, 404, 503

**Costos CloudFront** (mensual):
- Transfer out: ~45TB/mes → USD 3.800
- Requests HTTP/HTTPS: ~8.000M/mes → USD 650
- **Total CloudFront**: USD 4.450/mes

---

### 2. Seguridad Perimetral (WAF + Shield)

**AWS WAF**:
- Reglas administradas: AWS Managed Rules (Common, SQLi, XSS, PHP, WordPress)
- Reglas personalizadas:
  - Rate limiting: 500 req/5min por IP
  - Bloqueo de user-agents sospechosos
  - Validación de tokens CSRF
  - Bloqueo de patrones de scraping (acceso a > 1000 SKUs/hora por IP)
- Web ACLs asociados a CloudFront y ALB
- Logging completo en CloudWatch Logs + Kinesis Firehose → S3 (retención 90 días)

**AWS Shield Advanced**:
- Protección DDoS L3/L4/L7
- DDoS Response Team (DRT) disponible 24/7
- Protección contra ataques volumétricos (hasta 1 Tbps absorbidos)
- Costo: USD 3.000/mes (flat) + transfer costs

---

### 3. Balanceo de Carga (Application Load Balancer)

**Configuración**:
- Listeners: 443 (HTTPS) con certificados ACM, 80 (HTTP → redirect 301)
- Target groups: Frontend (puerto 3000), API (puerto 8080), Admin (puerto 9090)
- Health checks: /health cada 15 segundos, 2 fallos consecutivos → instancia unhealthy
- Sticky sessions: deshabilitadas (estado manejado en Redis)
- Access logs: habilitados → S3 (retención 30 días)
- Algoritmo de enrutamiento: Least Outstanding Requests

**Seguridad ALB**:
- Security groups: solo acepta tráfico de CloudFront IPs (lista mantenida con Lambda + EventBridge)
- SSL Policy: ELBSecurityPolicy-TLS13-1-2-2021 (TLS 1.3 preferido, TLS 1.2 mínimo)

---

### 4. Cómputo (ECS Fargate)

**Cluster: megatienda-production**

#### Servicio Frontend (Next.js 14)
- Task definition: 2 vCPU, 4GB RAM
- Imágenes Docker: ECR, multi-stage build (imagen final < 180MB)
- Escala mínima: 3 tareas (en 3 AZs para HA)
- Escala máxima: 50 tareas
- Auto Scaling basado en:
  - ALBRequestCountPerTarget > 2000 req/min → scale out
  - CPU > 70% por 3 minutos → scale out
  - CPU < 30% por 10 minutos → scale in
- Cool-down period: 120 segundos (scale out), 300 segundos (scale in)

#### Servicio API Backend (Node.js 20 + Express)
- Task definition: 4 vCPU, 8GB RAM
- Escala mínima: 5 tareas (resiliencia y redundancia)
- Escala máxima: 100 tareas
- Auto Scaling basado en:
  - P95 latency > 150ms por 2 minutos → scale out
  - SQS queue depth > 500 mensajes → scale out
  - CPU > 60% → scale out

**Imagen base**: node:20-alpine (por seguridad y tamaño)
**Build pipeline**: GitHub Actions → ECR → ECS Blue/Green deployment
**Deployment strategy**: CodeDeploy Blue/Green con health check (0% tráfico a nueva versión hasta pasar health checks)

---

### 5. Base de Datos (Aurora PostgreSQL)

**Cluster**: megatienda-prod-aurora-cluster
**Versión**: Aurora PostgreSQL 15.4
**Instancia writer**: db.r7g.2xlarge (8 vCPU, 64GB RAM)
**Instancias reader**: 2x db.r7g.xlarge (en diferentes AZs)

**Configuración**:
- Multi-AZ automático (failover < 30 segundos)
- Automated backups: 7 días de retención
- Point-in-time recovery habilitado
- Aurora Global Database: réplica de lectura en us-east-1 (para disaster recovery)
- Parameter group personalizado: max_connections=500, work_mem=64MB

**Bases de datos por microservicio**:
- `megatienda_catalog` → 2.4M productos, 847GB
- `megatienda_orders` → 14M órdenes, 234GB
- `megatienda_users` → 2.8M usuarios, 12GB
- `megatienda_inventory` → tiempo real stock, 45GB

**Performance Insights**: habilitado, retención 7 días
**Slow query log**: threshold 1000ms, exportado a CloudWatch

---

### 6. Caché (ElastiCache Redis)

**Cluster**: megatienda-redis-prod
**Versión**: Redis 7.2
**Modo**: Cluster mode (6 shards, 2 réplicas por shard = 18 nodos)
**Instancias**: cache.r7g.large (2 vCPU, 13GB RAM por nodo)

**Uso del caché por categoría**:

| Tipo de dato | TTL | Tamaño estimado |
|---|---|---|
| Sesiones de usuario | 24 horas | ~8GB |
| Catálogo de productos | 1 hora | ~45GB |
| Precios y descuentos | 5 minutos | ~2GB |
| Resultados de búsqueda | 15 minutos | ~12GB |
| Carrito de compras | 72 horas | ~3GB |
| Rate limiting | 5 minutos | ~1GB |

**Hit rate target**: > 85% (actual: 91.3%)

---

### 7. Cola de Mensajes (SQS)

**Colas configuradas**:

| Cola | Tipo | Propósito | Timeout visibilidad |
|---|---|---|---|
| order-processing | Standard | Procesamiento de órdenes nuevas | 30 seg |
| email-notifications | Standard | Envío de emails transaccionales | 60 seg |
| inventory-updates | Standard | Actualización de stock | 15 seg |
| search-indexing | Standard | Reindexar productos en OpenSearch | 120 seg |
| payment-webhooks | FIFO | Webhooks de pasarela de pago | 30 seg |
| dlq-order-processing | Standard | Dead letter queue para órdenes fallidas | — |

**Configuraciones de seguridad**:
- Encriptación SSE con KMS keys propias
- VPC endpoint para no exponer tráfico a internet público
- IAM roles por servicio (principio de mínimo privilegio)

---

### 8. Funciones Serverless (Lambda)

**Funciones principales**:

| Función | Trigger | Runtime | Memoria | Timeout |
|---|---|---|---|---|
| order-confirmation-email | SQS | Node.js 20 | 512MB | 30s |
| inventory-sync | SQS | Python 3.12 | 1GB | 60s |
| product-image-resize | S3 | Node.js 20 | 2GB | 120s |
| update-waf-cloudfront-ips | EventBridge (diario) | Python 3.12 | 256MB | 60s |
| aurora-snapshot-verify | EventBridge (diario) | Python 3.12 | 512MB | 300s |

**Capas Lambda** (shared dependencies):
- `megatienda-node-deps`: lodash, uuid, axios
- `megatienda-python-deps`: boto3, requests, psycopg2

---

### 9. Monitoreo y Observabilidad

**Stack de observabilidad**:
- **Métricas**: CloudWatch + Grafana (dashboard unificado)
- **Logs**: CloudWatch Logs → Kinesis → OpenSearch (Kibana para análisis)
- **Trazabilidad**: AWS X-Ray (sampling rate: 5% en producción, 100% en staging)
- **Alertas**: CloudWatch Alarms → SNS → PagerDuty (on-call)
- **Sintéticos**: CloudWatch Synthetics (canary scripts cada 5 min en flujos críticos)

**Alertas críticas (PagerDuty inmediato)**:
- Error rate API > 1% por 2 minutos
- P99 latency > 500ms por 3 minutos
- Disponibilidad checkout < 99% en 5 minutos
- Aurora failover
- ECS service scale to max

---

## COSTOS DETALLADOS (Mensual)

| Servicio | Costo USD/mes |
|---|---|
| CloudFront | $4.450 |
| WAF + Shield | $3.850 |
| ECS Fargate (Compute) | $8.200 |
| Aurora PostgreSQL | $12.400 |
| ElastiCache Redis | $7.800 |
| Application Load Balancer | $1.200 |
| Lambda | $340 |
| SQS | $180 |
| S3 (assets, logs, backups) | $2.100 |
| CloudWatch + X-Ray | $1.800 |
| VPC, NAT Gateways, data transfer | $4.200 |
| Route53, ACM, otros | $480 |
| Soporte Enterprise (prorrateado) | $800 |
| **TOTAL** | **$47.800** |

---

*Sube este documento a NotebookLM junto con el runbook de Kubernetes y pregunta: "¿Cuáles son los objetivos de disponibilidad y recuperación definidos?" o "¿Qué estrategia de caché se usa para el catálogo de productos?" o "¿Cuáles son las alertas críticas que generan un llamado inmediato al equipo?"*
