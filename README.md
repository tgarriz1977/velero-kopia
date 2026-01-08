# RUNBOOK

## Backup de Kubernetes con Velero y Kopia

**Amazon EKS + S3 + Volúmenes (GP2, GP3, EFS)**

*Versión 1.1 - Enero 2026*

---

## Tabla de Contenidos

1. [Introducción](#1-introducción)
2. [Configuración de AWS S3](#2-configuración-de-aws-s3)
3. [Configuración de IAM para Velero](#3-configuración-de-iam-para-velero)
4. [Instalación de Velero](#4-instalación-de-velero)
5. [Configuración para Tipos de Volúmenes](#5-configuración-para-tipos-de-volúmenes)
6. [Procedimientos de Backup](#6-procedimientos-de-backup)
7. [Procedimientos de Restore](#7-procedimientos-de-restore)
8. [Monitoreo y Troubleshooting](#8-monitoreo-y-troubleshooting)
9. [Mejores Prácticas](#9-mejores-prácticas)
10. [Checklist de Validación](#10-checklist-de-validación)

---

## 1. Introducción

### 1.1 Objetivo

Este runbook proporciona los procedimientos detallados para configurar una solución de backup completa para clusters Amazon EKS utilizando Velero con Kopia (fs-backup), permitiendo realizar copias de seguridad de namespaces que contienen diferentes tipos de volúmenes persistentes.

> **Nota:** A partir de Velero v1.17, Restic fue deprecado y reemplazado por Kopia como uploader por defecto.

### 1.2 Alcance

Este documento cubre:

- Configuración de servicios AWS S3 para almacenamiento de backups
- Instalación y configuración de Velero en Amazon EKS
- Configuración de Kopia para backup de volúmenes (fs-backup)
- Soporte para volúmenes: GP2, GP3, EFS y S3 StorageClass
- Procedimientos de backup y restore de namespaces

### 1.3 Prerrequisitos

| Componente | Requisito |
|------------|-----------|
| AWS CLI | v2.x configurado con credenciales |
| kubectl | v1.28+ compatible con EKS |
| Helm | v3.x instalado |
| EKS Cluster | Versión 1.28 o superior |
| IAM | Permisos para roles, políticas y S3 |
| Velero CLI | v1.17+ (se instalará) |

---

## 2. Configuración de AWS S3

### 2.1 Crear el Bucket S3 para Backups

Definir las variables de entorno necesarias:

```bash
#!/bin/bash
export AWS_REGION="us-east-2"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="velero-backups-staging-${AWS_ACCOUNT_ID}-${AWS_REGION}"
export CLUSTER_NAME="colegio-staging"
```

Crear el bucket S3 con configuraciones de seguridad:

```bash
# Crear el bucket S3
aws s3api create-bucket \
    --bucket ${BUCKET_NAME} \
    --region ${AWS_REGION} \
    --create-bucket-configuration LocationConstraint=${AWS_REGION}

# Habilitar versionado
aws s3api put-bucket-versioning \
    --bucket ${BUCKET_NAME} \
    --versioning-configuration Status=Enabled

# Bloquear acceso público
aws s3api put-public-access-block \
    --bucket ${BUCKET_NAME} \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Habilitar cifrado SSE-S3 (sin costo adicional, a diferencia de KMS)
aws s3api put-bucket-encryption \
    --bucket ${BUCKET_NAME} \
    --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

### 2.2 Configurar Lifecycle Policy (Opcional - No aplicado en staging)

> **Nota:** En staging se omitió este paso. Se usa TTL de Velero para controlar la retención de backups.

```bash
# Crear lifecycle policy
cat > /tmp/lifecycle.json << 'EOF'
{
    "Rules": [
        {
            "ID": "MoveToGlacier",
            "Status": "Enabled",
            "Filter": {"Prefix": "backups/"},
            "Transitions": [{"Days": 30, "StorageClass": "GLACIER"}],
            "Expiration": {"Days": 365}
        },
        {
            "ID": "CleanKopia",
            "Status": "Enabled",
            "Filter": {"Prefix": "kopia/"},
            "Expiration": {"Days": 90}
        }
    ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
    --bucket ${BUCKET_NAME} \
    --lifecycle-configuration file:///tmp/lifecycle.json
```

---

## 3. Configuración de IAM para Velero

### 3.1 Crear la Política IAM

```bash
cat > /tmp/velero-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "S3Access",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": ["arn:aws:s3:::BUCKET_NAME/*"]
        },
        {
            "Sid": "S3List",
            "Effect": "Allow",
            "Action": ["s3:ListBucket"],
            "Resource": ["arn:aws:s3:::BUCKET_NAME"]
        },
        {
            "Sid": "EC2",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeVolumes",
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot"
            ],
            "Resource": "*"
        }
    ]
}
EOF

sed -i "s/BUCKET_NAME/${BUCKET_NAME}/g" /tmp/velero-policy.json

aws iam create-policy --policy-name VeleroAccessPolicy --policy-document file:///tmp/velero-policy.json
```

### 3.2 Configurar IRSA

IRSA (IAM Roles for Service Accounts) permite que los pods de Velero obtengan credenciales AWS temporales de forma segura, sin necesidad de almacenar access keys.

```bash
# Obtener OIDC Provider
export OIDC_PROVIDER=$(aws eks describe-cluster --name ${CLUSTER_NAME} \
    --query "cluster.identity.oidc.issuer" --output text | sed 's|https://||')

# Trust Policy
cat > /tmp/trust-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {"Federated": "arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"},
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {"StringEquals": {
            "${OIDC_PROVIDER}:aud": "sts.amazonaws.com",
            "${OIDC_PROVIDER}:sub": "system:serviceaccount:velero:velero-server"
        }}
    }]
}
EOF

# Crear rol
aws iam create-role --role-name VeleroRole-${CLUSTER_NAME} \
    --assume-role-policy-document file:///tmp/trust-policy.json

aws iam attach-role-policy --role-name VeleroRole-${CLUSTER_NAME} \
    --policy-arn arn:aws:iam::${AWS_ACCOUNT_ID}:policy/VeleroAccessPolicy

export VELERO_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/VeleroRole-${CLUSTER_NAME}"
```

---

## 4. Instalación de Velero

### 4.1 Instalar Velero CLI

```bash
VELERO_VERSION="v1.17.1"
curl -LO https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz
tar -xzf velero-${VELERO_VERSION}-linux-amd64.tar.gz
sudo mv velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/
velero version --client-only
```

### 4.2 Crear Namespace

```bash
kubectl create namespace velero
```

### 4.3 Instalación con Helm

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

cat > /tmp/velero-values.yaml << EOF
image:
  repository: velero/velero
  tag: v1.17.1

initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.11.0
    volumeMounts:
      - mountPath: /target
        name: plugins

configuration:
  backupStorageLocation:
    - name: default
      provider: aws
      bucket: ${BUCKET_NAME}
      prefix: backups
      config:
        region: ${AWS_REGION}
  volumeSnapshotLocation:
    - name: default
      provider: aws
      config:
        region: ${AWS_REGION}
  features: EnableCSI
  defaultVolumesToFsBackup: true

serviceAccount:
  server:
    create: true
    name: velero-server
    annotations:
      eks.amazonaws.com/role-arn: ${VELERO_ROLE_ARN}

credentials:
  useSecret: false

deployNodeAgent: true

nodeAgent:
  podVolumePath: /var/lib/kubelet/pods
  privileged: true
EOF

helm install velero vmware-tanzu/velero -n velero -f /tmp/velero-values.yaml --wait

kubectl get pods -n velero
kubectl get daemonset -n velero
```

---

## 5. Configuración para Tipos de Volúmenes

### 5.0.1 Para usar snapshots nativos de EBS, necesitás instalarlos

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/master/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
```

### 5.1 Volúmenes EBS (GP2/GP3)
Los volúmenes GP2 y GP3 soportan snapshots nativos EBS y backup file-system con Kopia:

```bash
# StorageClass GP3
cat << 'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-velero
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF

# VolumeSnapshotClass
cat << 'EOF' | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-csi-snapclass
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: ebs.csi.aws.com
deletionPolicy: Delete
EOF
```

### 5.2 Volúmenes EFS

EFS requiere backup file-system (Kopia) ya que no soporta snapshots CSI como EBS:

```bash
# StorageClass EFS
cat << 'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-velero
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-XXXXXXXXX
  directoryPerms: "700"
EOF
```

Anotación en Pods para backup EFS:

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        backup.velero.io/backup-volumes: efs-data
```

### 5.3 S3 StorageClass

Para Mountpoint for S3, se recomienda usar S3 Replication para backups:

```bash
# Instalar CSI Driver S3
helm repo add aws-mountpoint-s3-csi-driver https://awslabs.github.io/mountpoint-s3-csi-driver
helm install aws-mountpoint-s3-csi-driver aws-mountpoint-s3-csi-driver/aws-mountpoint-s3-csi-driver -n kube-system

# StorageClass S3
cat << 'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: s3-storage
provisioner: s3.csi.aws.com
parameters:
  bucketName: mi-bucket-datos
EOF

# Excluir volumen S3 de Velero (usar S3 Replication)
kubectl annotate pod <pod> backup.velero.io/backup-volumes-excludes=s3-volume
```

---

## 6. Procedimientos de Backup

### 6.1 Backup Manual

```bash
# Backup completo de namespace
velero backup create backup-$(date +%Y%m%d-%H%M%S) \
    --include-namespaces mi-namespace \
    --default-volumes-to-fs-backup \
    --snapshot-volumes --wait

# Verificar backup
velero backup describe <backup-name> --details
velero backup logs <backup-name>
```

### 6.2 Backup Programado

```bash
# Schedule diario con TTL de 7 días
velero schedule create backup-diario \
    --schedule="0 2 * * *" \
    --include-namespaces produccion \
    --default-volumes-to-fs-backup \
    --ttl 168h

# Listar schedules
velero schedule get

# Pausar/reanudar
velero schedule pause backup-diario
velero schedule unpause backup-diario
```

### 6.3 Backup con Hooks

```yaml
# Anotaciones para hooks en Deployment
metadata:
  annotations:
    pre.hook.backup.velero.io/container: mysql
    pre.hook.backup.velero.io/command: '["/bin/sh","-c","mysqldump..."]'
    post.hook.backup.velero.io/container: mysql
    post.hook.backup.velero.io/command: '["/bin/sh","-c","..."]'
    backup.velero.io/backup-volumes: mysql-data
```
### 6.4 Se programan los siguientes schedulers de backups
```bash
velero schedule create backup-argocd-diario \
    --schedule="0 1 * * *" \
    --include-namespaces argocd \
    --default-volumes-to-fs-backup \
    --ttl 168h
    
velero schedule create backup-nextcloud-diario \
    --schedule="15 1 * * *" \
    --include-namespaces nextcloud \
    --default-volumes-to-fs-backup \
    --ttl 168h
    
velero schedule create backup-sigma2-staging-diario \
    --schedule="30 1 * * *" \
    --include-namespaces sigma2-staging \
    --default-volumes-to-fs-backup \
    --ttl 168h

velero schedule create backup-ingress-nginx-diario \
    --schedule="45 5 * * *" \
    --include-namespaces ingress-nginx \
    --default-volumes-to-fs-backup \
    --ttl 168h
```
---

## 7. Procedimientos de Restore

### 7.1 Restore Completo

```bash
# Listar backups
velero backup get

# Restore completo
velero restore create --from-backup <backup-name> --wait

# Restore a namespace diferente
velero restore create --from-backup <backup-name> \
    --namespace-mappings produccion:produccion-restored --wait

# Verificar restore
velero restore describe <restore-name> --details
```

### 7.2 Restore Selectivo

```bash
# Solo ciertos recursos
velero restore create --from-backup <backup> \
    --include-resources deployments,services,configmaps --wait

# Con selector
velero restore create --from-backup <backup> \
    --selector app=frontend --wait
```

### 7.3 Disaster Recovery

```bash
# En cluster DR, instalar Velero con mismo bucket
helm install velero vmware-tanzu/velero -n velero \
    -f /tmp/velero-values.yaml --wait

# Verificar backups visibles
velero backup get

# Restore en cluster DR
velero restore create dr-restore --from-backup <backup> --wait
```

---

## 8. Monitoreo y Troubleshooting

### 8.1 Comandos de Diagnóstico

```bash
velero version
velero backup-location get
velero snapshot-location get
velero backup get
kubectl logs deployment/velero -n velero
kubectl logs daemonset/node-agent -n velero
kubectl get podvolumebackups -n velero
```

### 8.2 Problemas Comunes

| Problema | Solución |
|----------|----------|
| PartiallyFailed | `velero backup logs <n>` |
| Node Agent no inicia | Verificar privilegios DaemonSet |
| Error S3 | Verificar IAM/IRSA |
| EBS snapshot falla | Verificar VolumeSnapshotClass |
| Volumen no incluido | Agregar anotación `backup.velero.io` |

### 8.3 Prometheus Metrics

```yaml
# ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: velero
  namespace: velero
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: velero
  endpoints:
    - port: http-monitoring
      interval: 30s
```

---

## 9. Mejores Prácticas

- **Estrategia 3-2-1**: 3 copias, 2 medios, 1 offsite (S3 Cross-Region)
- Probar restores mensualmente en staging
- **TTL**: diarios 7d, semanales 30d, mensuales 1 año
- **Excluir**: Events, ReplicaSets, endpoints
- Usar hooks para consistencia de bases de datos
- Monitorear espacio S3 con CloudWatch
- Cifrar backups (SSE-S3 o KMS)
- Versionar bucket S3
- Separar buckets prod/dev

---

## 10. Checklist de Validación

| Verificación | Comando |
|--------------|---------|
| Velero pods | `kubectl get pods -n velero` |
| Node Agent | `kubectl get ds -n velero` |
| Backup Location | `velero backup-location get` |
| Snapshot Location | `velero snapshot-location get` |
| Test backup | `velero backup create test --include-namespaces default` |
| Test restore | `velero restore create --from-backup test` |
| Schedules | `velero schedule get` |

---

## Changelog

| Versión | Fecha | Cambios |
|---------|-------|---------|
| 1.0 | Enero 2026 | Versión inicial con Velero 1.13 |
| 1.1 | Enero 2026 | Actualizado a Velero 1.17.1, reemplazo Restic por Kopia, cifrado SSE-S3 |

