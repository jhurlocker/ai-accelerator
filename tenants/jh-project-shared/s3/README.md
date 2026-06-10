# RUSTFS S3 Object Storage

Generated manifests from [RUSTFS Helm chart](https://github.com/rustfs/rustfs/tree/main/helm)
for standalone deployment in the `jh-project-shared` namespace.

Based on the RUSTFS Operator deployment pattern:
- Single PVC: `rustfs-data` (100Gi)
- Split services: `rustfs-s3` (API port 9000) and `rustfs-console` (Console port 9001)
- Image: `docker.io/rustfs/rustfs:latest`
- Security: `runAsNonRoot` with SCC-managed UID (1000960000)
- Health endpoint: `/health` (native RUSTFS endpoint)
- Credentials: `rustfs-credentials` secret with `RUSTFS_ACCESS_KEY` and `RUSTFS_SECRET_KEY`

## Regenerating from Helm

```bash
helm template rustfs rustfs/rustfs \
  --namespace jh-project-shared \
  --set mode.standalone.enabled=true,mode.distributed.enabled=false \
  --set ingress.enabled=false,service.type=ClusterIP \
  --set secret.existingSecret=rustfs-credentials \
  --set image.rustfs.tag=latest \
  --set storageclass.dataStorageSize=100Gi \
  --set storageclass.name=gp3-csi
```

Split the output into individual files (pvc.yaml, deployment.yaml, service.yaml,
serviceaccount.yaml). Note: the Helm chart creates a single service by default;
split into `rustfs-s3` and `rustfs-console` for proper port separation.

## Key Differences from MinIO

| Aspect | MinIO | RUSTFS |
|---|---|---|
| Image | `quay.io/minio/minio` | `docker.io/rustfs/rustfs` |
| Console port | 9090 | 9001 |
| API port | 9000 | 9000 |
| Liveness probe | `/minio/health/live` | `/health` |
| Readiness probe | `/minio/health/ready` | `/health` |
| Secret keys | `MINIO_ROOT_USER/PASSWORD` | `RUSTFS_ACCESS_KEY/SECRET_KEY` |
| Run as | root | UID 1000960000 (SA-based) |
| Service names | Single combined service | Separate `rustfs-s3` and `rustfs-console` |
| Health endpoint | `/minio/health/live` | `/health` (native only, no MinIO compat) |
| Bucket creation | `mc` CLI | `mc` or `curl` S3 API |

## S3 Credentials

The `rustfs-credentials` secret must contain these keys (created by `apply-credentials.sh`):

```yaml
stringData:
  RUSTFS_ACCESS_KEY: "<admin-username>"
  RUSTFS_SECRET_KEY: "<admin-password>"
```

## Health Endpoints

RUSTFS provides a native health endpoint:
- Native: `GET /health`

The deployment uses the native `/health` endpoint for both liveness and readiness probes.
**Note:** RUSTFS does NOT provide MinIO-compatible health endpoints (`/minio/health/live`,
`/minio/health/ready`) — use `/health` only.

## DNS Endpoints

- **S3 API**: `rustfs-s3.<namespace>.svc.cluster.local:9000`
- **Console**: `rustfs-console.<namespace>.svc.cluster.local:9001`

For external access, use the OpenShift Routes:
- `rustfs-s3-<namespace>.<cluster-domain>`
- `rustfs-console-<namespace>.<cluster-domain>`
