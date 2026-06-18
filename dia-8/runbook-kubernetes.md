# Runbook Operativo — Kubernetes
## Plataforma MegaTienda.com | Cluster Production
### Versión: 1.8 | Última actualización: Noviembre 2024

---

## INFORMACIÓN DEL CLUSTER

| Campo | Valor |
|---|---|
| Proveedor | Amazon EKS |
| Versión de Kubernetes | 1.29 |
| Región primaria | us-east-1 |
| Región DR | us-west-2 |
| Node groups | spot-general (m6i.xlarge), on-demand-critical (m6i.2xlarge) |
| Nodos mínimos | 6 (2 por AZ) |
| Nodos máximos | 40 (Cluster Autoscaler) |
| Acceso | kubectl vía AWS SSO + kubeconfig |

---

## NAMESPACES Y WORKLOADS

```
kubectl get namespaces
```

| Namespace | Descripción | Workloads principales |
|---|---|---|
| production | Servicios de producción | frontend, api, worker, scheduler |
| monitoring | Stack de observabilidad | prometheus, grafana, alertmanager |
| ingress-nginx | Controlador de ingress | nginx-ingress-controller |
| cert-manager | Gestión de certificados TLS | cert-manager |
| kube-system | Componentes de Kubernetes | cluster-autoscaler, aws-node, coredns |

---

## PROCEDIMIENTOS OPERATIVOS COMUNES

### OP-001: Ver estado general del cluster

```bash
# Estado de todos los nodos
kubectl get nodes -o wide

# Estado de todos los pods en producción
kubectl get pods -n production

# Pods con problemas (no en Running/Completed)
kubectl get pods -n production --field-selector=status.phase!=Running,status.phase!=Succeeded

# Resumen de recursos del cluster
kubectl top nodes
kubectl top pods -n production --sort-by=memory
```

**Señales de alerta en `kubectl get nodes`**:
- Status `NotReady`: El nodo no puede recibir pods. Actuar inmediatamente.
- Status `SchedulingDisabled`: El nodo fue marcado para mantenimiento (cordon).

---

### OP-002: Reiniciar un deployment

```bash
# Reinicio rolling (sin downtime)
kubectl rollout restart deployment/api -n production

# Ver el progreso del rollout
kubectl rollout status deployment/api -n production

# Ver historial de rollouts
kubectl rollout history deployment/api -n production
```

**Cuándo usar**: Cuando un servicio se comporta de forma errática sin que haya cambios de código (bug de memoria, conexiones colgadas, cache corrompido en el pod).

---

### OP-003: Escalar manualmente un deployment

```bash
# Escalar a 10 réplicas
kubectl scale deployment/api -n production --replicas=10

# Verificar
kubectl get deployment api -n production

# Volver a dejar el HPA en control (escalar a 0 replicas manuales no aplica con HPA)
# El HPA retomará el control automáticamente en el siguiente ciclo
```

**Cuándo usar**: Durante picos anticipados (ej. horas antes de un Black Friday) para pre-escalar antes de que el HPA reaccione.

**Importante**: Después del evento, verificar que el HPA haya retomado el control y que la escala bajó correctamente.

---

### OP-004: Ver logs de un pod

```bash
# Logs del pod más reciente de un deployment
kubectl logs -n production deployment/api --tail=100

# Logs de un pod específico
kubectl logs -n production <nombre-del-pod> --tail=200

# Logs de un pod que ya murió (estado CrashLoopBackOff)
kubectl logs -n production <nombre-del-pod> --previous

# Seguir los logs en tiempo real (como tail -f)
kubectl logs -n production deployment/api --follow

# Logs de múltiples pods con stern (herramienta adicional)
stern -n production api --since=15m
```

---

### OP-005: Ejecutar un comando dentro de un pod

```bash
# Shell interactiva en el pod
kubectl exec -it -n production <nombre-del-pod> -- /bin/sh

# Comando puntual sin shell interactiva
kubectl exec -n production <nombre-del-pod> -- curl http://localhost:8080/health

# Verificar conectividad a la base de datos desde el pod
kubectl exec -it -n production <nombre-del-pod> -- nc -zv aurora-endpoint.cluster.local 5432
```

---

### OP-006: Describir un recurso (diagnóstico)

```bash
# Información detallada de un pod (útil para ver eventos y errores)
kubectl describe pod -n production <nombre-del-pod>

# Información de un nodo
kubectl describe node <nombre-del-nodo>

# Ver eventos del namespace (últimas 2 horas)
kubectl get events -n production --sort-by='.lastTimestamp' | tail -50
```

**El campo `Events` en `kubectl describe pod` es clave para diagnosticar**:
- `Failed to pull image`: Problema con el registro de contenedores (ECR)
- `Insufficient memory`: El nodo no tiene recursos suficientes
- `Liveness probe failed`: El pod no responde al health check → Kubernetes lo reinicia
- `Back-off pulling image`: Demasiados intentos fallidos de descargar la imagen

---

## PROCEDIMIENTOS DE INCIDENTE

### INC-001: Pod en CrashLoopBackOff

**Síntomas**: El pod reinicia constantemente. Estado `CrashLoopBackOff`.

**Diagnóstico**:
```bash
# 1. Ver los logs del último crash
kubectl logs -n production <pod-name> --previous

# 2. Describir el pod para ver los eventos
kubectl describe pod -n production <pod-name>

# 3. Ver si hay patrón en la hora de los crashes
kubectl get pod -n production <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState}'
```

**Causas comunes y soluciones**:

| Causa | Señal en logs | Solución |
|---|---|---|
| OOMKilled (falta memoria) | `Reason: OOMKilled` en describe | Aumentar memory limit en el deployment |
| Error de configuración | `Error: env variable X not set` | Verificar ConfigMap y Secrets |
| Error de conexión a DB | `ECONNREFUSED postgres:5432` | Verificar secret DATABASE_URL y conectividad de red |
| Error de imagen | `exec format error` | Imagen construida para arquitectura incorrecta (ARM vs x86) |
| Liveness probe muy agresivo | `Liveness probe failed` | Aumentar initialDelaySeconds en la probe |

---

### INC-002: Nodo en estado NotReady

**Síntomas**: `kubectl get nodes` muestra un nodo con STATUS `NotReady`.

**Diagnóstico**:
```bash
# 1. Describir el nodo
kubectl describe node <nombre-del-nodo>

# 2. Ver qué pods estaban en ese nodo
kubectl get pods -n production -o wide | grep <nombre-del-nodo>

# 3. Ver logs de kubelet en el nodo (requiere acceso SSH o SSM)
# Vía AWS Systems Manager:
aws ssm start-session --target <instance-id>
sudo journalctl -u kubelet -f
```

**Acciones**:
1. Si el nodo está en estado NotReady por más de 5 minutos, drenarlo:
```bash
kubectl drain <nombre-del-nodo> --ignore-daemonsets --delete-emptydir-data
```
2. Los pods migrarán automáticamente a otros nodos.
3. El Cluster Autoscaler debería provisionar un nodo nuevo automáticamente.
4. Una vez drenado, marcar el nodo para terminación desde la consola AWS o con:
```bash
aws ec2 terminate-instances --instance-ids <instance-id>
```

---

### INC-003: Deployment atascado en rollout

**Síntomas**: Un `kubectl rollout restart` o `kubectl apply` no completa. Los pods nuevos no llegan a `Running`.

**Diagnóstico**:
```bash
# Ver el estado del rollout
kubectl rollout status deployment/api -n production

# Ver los pods nuevos y viejos
kubectl get pods -n production -l app=api --sort-by=.metadata.creationTimestamp

# Ver los eventos recientes
kubectl get events -n production --sort-by='.lastTimestamp' | tail -30
```

**Acción de emergencia: Revertir al deployment anterior**:
```bash
# Ver el historial
kubectl rollout history deployment/api -n production

# Revertir a la versión anterior
kubectl rollout undo deployment/api -n production

# Revertir a una revisión específica
kubectl rollout undo deployment/api -n production --to-revision=3
```

---

### INC-004: Alta latencia en la API

**Diagnóstico**:
```bash
# Ver uso de recursos de los pods de API
kubectl top pods -n production -l app=api

# Ver si el HPA está escalando
kubectl get hpa -n production
kubectl describe hpa api -n production

# Ver si hay throttling en la base de datos
# (requiere acceso a CloudWatch Performance Insights)
```

**Checklist de diagnóstico**:
- [ ] ¿Hay un aumento súbito de tráfico? (ver CloudWatch → ALB → RequestCount)
- [ ] ¿Están los pods de API saturados? (CPU > 80%?)
- [ ] ¿Está el HPA intentando escalar pero no puede? (ver `kubectl describe hpa`)
- [ ] ¿Está la base de datos saturada? (ver RDS Performance Insights)
- [ ] ¿Está el caché Redis respondiendo? (`redis-cli ping` desde un pod)
- [ ] ¿Hay algún job en batch corriendo en ese momento que esté saturando la DB?

---

## GESTIÓN DE CONFIGURACIÓN Y SECRETOS

### Ver y editar ConfigMaps

```bash
# Listar ConfigMaps en producción
kubectl get configmap -n production

# Ver el contenido de un ConfigMap
kubectl get configmap api-config -n production -o yaml

# Editar un ConfigMap (abre editor de texto)
kubectl edit configmap api-config -n production
```

### Gestión de Secretos

```bash
# Listar secretos
kubectl get secrets -n production

# Ver el nombre de las claves de un secreto (sin revelar el valor)
kubectl get secret api-secrets -n production -o jsonpath='{.data}' | python3 -c "import sys,json; [print(k) for k in json.load(sys.stdin).keys()]"

# Decodificar el valor de una clave específica
kubectl get secret api-secrets -n production -o jsonpath='{.data.DATABASE_URL}' | base64 --decode
```

**Importante**: Nunca commitear valores de secretos en git. Los secretos en producción están gestionados con AWS Secrets Manager + External Secrets Operator.

---

## ACCESO DE EMERGENCIA

### Si kubectl no funciona (cluster no responde)

1. Verificar conectividad con AWS: `aws eks describe-cluster --name megatienda-production`
2. Actualizar kubeconfig: `aws eks update-kubeconfig --name megatienda-production --region us-east-1`
3. Verificar credenciales AWS SSO: `aws sts get-caller-identity`
4. Si el Control Plane de EKS no responde: Abrir ticket con AWS Support (Enterprise Support, P1)

### Activar modo de mantenimiento (maintenance page)

```bash
# Cambiar el peso del target group en el ALB para mostrar maintenance page
# (la maintenance page es una Lambda@Edge que devuelve 503 con HTML estático)
aws elbv2 modify-listener --listener-arn <arn> --default-actions Type=fixed-response,...
```

---

## CONTACTOS DE ESCALAMIENTO

| Nivel | Condición | Contacto |
|---|---|---|
| L1 — On-call | Cualquier alerta crítica | PagerDuty → Slack #incidents |
| L2 — Tech Lead | Incidente sin resolución en 30 min | @carlos-tech-lead (Slack/phone) |
| L3 — AWS Support | Problema de plataforma AWS | Case P1 en AWS Console |
| Gestión | Incidente con impacto en ventas >1h | CTO + CEO via grupo WhatsApp Ejecutivo |

---

*Sube este runbook junto con la documentación de arquitectura AWS a NotebookLM y pregunta: "¿Cómo diagnostico un pod en CrashLoopBackOff?" o "¿Cuál es el procedimiento para revertir un deployment fallido?" o "¿Cuáles son los contactos de escalamiento según la gravedad del incidente?"*
