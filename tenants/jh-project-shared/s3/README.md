# RUSTFS S3 Object Storage

Generated manifests from [RUSTFS Helm chart](https://github.com/rustfs/rustfs/tree/main/helm)
for standalone deployment in the `jh-project-shared` namespace.

## Regenerating from Helm

```bash
helm template s3 rustfs/rustfs \
  --namespace jh-project-shared \
  --set mode.standalone.enabled=true,mode.distributed.enabled=false \
  --set ingress.enabled=false,service.type=ClusterIP \
  --set secret.existingSecret=s3-credentials \
  --set image.rustfs.tag=1.0.0-beta.7 \
  --set storageclass.dataStorageSize=50Gi,storageclass.logStorageSize=5Gi \
  --set storageclass.name=gp3-csi \
  --set resources.limits.cpu=2,resources.limits.memory=4Gi \
  --set resources.requests.cpu=500m,resources.requests.memory=512Mi \
  --set gatewayApi.enabled=false,nameOverride=s3
```

Split the output into individual files (configmap.yaml, pvc.yaml, deployment.yaml,
service.yaml, serviceaccount.yaml). The Helm chart names the service `s3-svc` by
default; override to `s3` for backward-compatible DNS (`s3.<ns>.svc.cluster.local`).

## Key Differences from MinIO

| Aspect | MinIO | RUSTFS |
|---|---|---|
| Image | `quay.io/minio/minio` | `rustfs/rustfs` |
| Console port | 9090 | 9001 |
| API port | 9000 | 9000 |
| Liveness probe | `/minio/health/live` | `/minio/health/live` |
| Readiness probe | `/minio/health/ready` | `/minio/health/ready` |
| Secret keys | `MINIO_ROOT_USER/PASSWORD` | `RUSTFS_ACCESS_KEY/SECRET_KEY` |
| Run as | root | UID 10001 |
| Health endpoint | `/minio/health/live` | `/health` (also `/minio/health/live`) |
| Bucket creation | `mc` CLI | `mc` or `curl` S3 API |

## S3 Credentials

The `s3-credentials` secret must contain these keys (created by `apply-credentials.sh`):

```yaml
stringData:
  RUSTFS_ACCESS_KEY: "<admin-username>"
  RUSTFS_SECRET_KEY: "<admin-password>"
```

## Health Endpoints

RUSTFS provides both native and MinIO-compatible health endpoints:
- Native: `GET /health` (liveness), `GET /health/ready` (readiness)
- MinIO-compatible: `GET /minio/health/live`, `GET /minio/health/ready`

The deployment uses MinIO-compatible paths for consistency with existing tooling.
