# Procedimiento de Gestión de Incidentes
## MegaTienda.com — Platform Engineering
### Versión: 2.0 | Aprobado: Noviembre 2024

---

## DEFINICIÓN DE SEVERIDADES

| Severidad | Descripción | Ejemplos | Tiempo de respuesta |
|---|---|---|---|
| **SEV-1 (Crítico)** | Servicio totalmente caído o impacto en >50% de usuarios | Sitio inaccesible, checkout sin funcionar, pérdida de datos | **5 minutos** |
| **SEV-2 (Alto)** | Degradación significativa o impacto en función clave | Latencia >2s en toda la plataforma, pagos fallando >10%, login caído | **15 minutos** |
| **SEV-3 (Medio)** | Funcionalidad degradada con workaround disponible | Búsqueda lenta, reseñas no cargando, admin inaccesible | **1 hora** |
| **SEV-4 (Bajo)** | Problema menor sin impacto operativo inmediato | Reporte de analytics incorrecto, banner de campaña sin mostrar | **Siguiente día hábil** |

---

## FLUJO DE RESPUESTA A INCIDENTES

```
DETECCIÓN → VALIDACIÓN → CLASIFICACIÓN → RESPUESTA → COMUNICACIÓN → RESOLUCIÓN → POST-MORTEM
```

---

## FASE 1: DETECCIÓN

Los incidentes pueden detectarse por:

1. **Alertas automáticas** (PagerDuty desde CloudWatch): Se dispara el pager del on-call.
2. **Reporte de usuario**: Canal #soporte-técnico en Slack o email crítico.
3. **Monitoreo proactivo** del equipo: Revisión de dashboards de Grafana.
4. **Reporte de stakeholders internos**: Área comercial, marketing, operaciones.

**Cualquier persona puede y debe reportar un posible incidente.** Más vale investigar una falsa alarma que ignorar un problema real.

---

## FASE 2: VALIDACIÓN (0-5 minutos)

El on-call recibe la alerta y verifica si es un incidente real:

```bash
# Verificar disponibilidad del frontend
curl -o /dev/null -s -w "%{http_code}\n" https://megatienda.com

# Verificar disponibilidad de la API
curl -o /dev/null -s -w "%{http_code}\n" https://api.megatienda.com/health

# Verificar latencia del checkout
curl -o /dev/null -s -w "Latencia: %{time_total}s\n" https://api.megatienda.com/checkout/health

# Verificar estado de los pods en Kubernetes
kubectl get pods -n production

# Ver métricas en CloudWatch (últimos 5 minutos)
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=<alb-arn> \
  --start-time $(date -u -d '5 minutes ago' +%FT%TZ) \
  --end-time $(date -u +%FT%TZ) \
  --period 60 \
  --statistics Sum
```

**Si el problema se confirma** → Abrir un incidente.
**Si es falsa alarma** → Documentar en PagerDuty y resolver la alerta.

---

## FASE 3: CLASIFICACIÓN Y APERTURA

### Abrir el incidente

1. **Crear canal de Slack**: `#inc-YYYYMMDD-descripcion-breve` (ej: `#inc-20241115-checkout-caido`)
2. **Notificar en Slack**:
```
@channel 🚨 INCIDENTE SEV-X ABIERTO 🚨
Descripción: [qué está fallando]
Impacto: [a quién/qué afecta]
Incident Commander: @[nombre]
Canal de coordinación: #inc-YYYYMMDD-xxx
```
3. **Crear ticket en JIRA** (si es SEV-1 o SEV-2): Proyecto INC, tipo "Incident"
4. **Activar PagerDuty** para llamar al equipo adicional (para SEV-1):
   - Incident Commander (Tech Lead de turno)
   - On-call de la plataforma
   - On-call de desarrollo

### Rol de Incident Commander (IC)

El IC coordina la respuesta. Sus responsabilidades:
- Coordinar la comunicación (no necesariamente resolver técnicamente)
- Asignar roles y tareas
- Mantener actualizado el canal de Slack cada 15 minutos
- Tomar decisiones de escalamiento
- Declarar la resolución y abrir el post-mortem

---

## FASE 4: RESPUESTA — PLAYBOOKS POR TIPO DE INCIDENTE

### PB-001: Sitio web inaccesible (503/504)

```bash
# 1. Verificar CloudFront
aws cloudfront get-distribution --id <distribution-id> | jq '.Distribution.Status'

# 2. Verificar ALB
kubectl get nodes  # ¿Hay nodos activos?
kubectl get pods -n production  # ¿Están los pods corriendo?

# 3. Si los pods están en crash
kubectl logs -n production deployment/frontend --previous | tail -50

# 4. Verificar si el ECS Fargate (si aplica) está corriendo
aws ecs describe-services --cluster megatienda-production --services frontend

# 5. Acción rápida: scale up manual
kubectl scale deployment/frontend -n production --replicas=10
```

### PB-002: Error rate > 5% en la API

```bash
# 1. Identificar el endpoint problemático
# Buscar en CloudWatch o Kibana: filtrar por status_code:5xx

# 2. Ver los logs de error
kubectl logs -n production deployment/api --since=15m | grep ERROR | tail -100

# 3. Verificar conexión a la base de datos
kubectl exec -n production <api-pod> -- nc -zv aurora-endpoint 5432 && echo "DB OK" || echo "DB FAIL"

# 4. Verificar conexión a Redis
kubectl exec -n production <api-pod> -- redis-cli -h redis-endpoint ping

# 5. Si es problema de la DB: activar tráfico al reader endpoint para consultas de lectura
# (cambio en ConfigMap de la API)

# 6. Si es bug de código: rollback del deployment
kubectl rollout undo deployment/api -n production
```

### PB-003: Latencia alta (P99 > 2 segundos)

```bash
# 1. Verificar si es un problema de capacidad
kubectl top pods -n production -l app=api  # ¿CPU > 80%?
kubectl get hpa -n production  # ¿El HPA está intentando escalar?

# 2. Si el HPA no puede escalar (falta de nodos)
kubectl get nodes  # ¿Cuántos nodos activos?
# El Cluster Autoscaler debería provisionar más. Si no lo hace:
kubectl describe pod <pod-pending> -n production  # Ver por qué está en Pending

# 3. Pre-escalar manualmente
kubectl scale deployment/api -n production --replicas=30

# 4. Verificar si la DB es el cuello de botella
# → Abrir RDS Performance Insights en la consola AWS
# Buscar: queries con wait time alto, conexiones activas

# 5. Si la DB está saturada: forzar que el tráfico de lectura vaya al reader endpoint
```

### PB-004: Base de datos no disponible

```bash
# 1. Verificar estado del cluster Aurora
aws rds describe-db-clusters --db-cluster-identifier megatienda-prod-aurora-cluster \
  | jq '.DBClusters[0].Status'

# 2. Si hay un failover en curso (esperado: < 30 segundos)
aws rds describe-events --source-identifier megatienda-prod-aurora-cluster \
  --source-type db-cluster --duration 60

# 3. Si el failover no se completó automáticamente: forzarlo
aws rds failover-db-cluster --db-cluster-identifier megatienda-prod-aurora-cluster

# 4. Mientras se resuelve: activar modo de mantenimiento en el sitio
# (para no mostrar errores al usuario)

# 5. Verificar que la aplicación reconectó al nuevo writer
kubectl rollout restart deployment/api -n production
```

---

## FASE 5: COMUNICACIÓN

### Comunicación interna (Slack #inc-xxx)
Actualización cada 15 minutos como mínimo:

```
⏰ ACTUALIZACIÓN [HH:MM]
Estado: Investigando / Mitigando / Resuelto
Qué sabemos: [resumen de hallazgos]
Qué estamos haciendo: [acciones en curso]
Próxima actualización: [hora]
```

### Comunicación a stakeholders (CEO, CMO, COO)
Solo para SEV-1 o SEV-2 con impacto económico. Usar el grupo de WhatsApp "Ejecutivo - Alertas":

```
🚨 INCIDENTE EN PLATAFORMA
Hora de inicio: [HH:MM]
Qué está fallando: [descripción no técnica]
Impacto estimado: [usuarios afectados, ventas impactadas si se puede estimar]
Acción: Equipo técnico en resolución. Próxima actualización en 30 min.
```

### Comunicación al cliente (página de status)
Para SEV-1: Actualizar status.megatienda.com con el estado del incidente.

---

## FASE 6: RESOLUCIÓN

Una vez resuelto el incidente:

1. **Declarar la resolución** en el canal de Slack:
```
✅ INCIDENTE RESUELTO [HH:MM]
Duración total: X horas Y minutos
Causa raíz (preliminar): [descripción]
Acciones tomadas: [resumen]
Post-mortem: a realizarse el [fecha]
```

2. **Actualizar status page** a "All systems operational"

3. **Notificar a stakeholders** con resumen

4. **Cerrar el ticket** en JIRA con el resumen de resolución

5. **Agendar el post-mortem** (ver siguiente sección)

---

## FASE 7: POST-MORTEM

**Objetivo**: Aprender del incidente para evitar que se repita. No es una sesión para asignar culpas.

### Cuándo hacer post-mortem
- **Obligatorio**: Todo SEV-1 y SEV-2
- **Opcional**: SEV-3 con aprendizajes significativos

### Cuándo hacerlo
- Dentro de los 3 días hábiles siguientes al incidente
- Separado del momento de máxima presión (no inmediatamente después)

### Estructura del post-mortem

**1. Resumen ejecutivo** (1 párrafo)

**2. Timeline del incidente** (tabla hora por hora)

**3. Causa raíz** (5 Whys o análisis de causa raíz)

**4. Impacto**
- Duración del incidente
- Usuarios afectados
- Impacto económico estimado (si aplica)

**5. ¿Qué funcionó bien?**
- Qué del proceso de respuesta fue efectivo

**6. ¿Qué podría mejorar?**
- Detección, comunicación, resolución

**7. Acciones de mejora** (tabla con responsables y fechas)

---

## MÉTRICAS DE INCIDENTES (KPIs)

| Métrica | Target | Medición |
|---|---|---|
| MTTD (Mean Time to Detect) | < 3 minutos | Tiempo desde primer síntoma hasta alerta |
| MTTR (Mean Time to Respond) | < 10 minutos para SEV-1 | Tiempo desde alerta hasta primer acción |
| MTTR (Mean Time to Resolve) | < 60 min SEV-1, < 4h SEV-2 | Tiempo desde apertura hasta resolución |
| Disponibilidad mensual | > 99.95% | (minutos disponibles / minutos totales) × 100 |
| Porcentaje de incidentes con post-mortem | 100% SEV-1/2 | Tracking en JIRA |

---

*Sube este procedimiento junto con el runbook de Kubernetes y la arquitectura AWS a NotebookLM. Pregunta: "¿Cuál es el procedimiento cuando la base de datos no está disponible?" o "¿Qué debe hacer el Incident Commander durante un incidente?" o "¿Cuándo es obligatorio hacer un post-mortem?"*
