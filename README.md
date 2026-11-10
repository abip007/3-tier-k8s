# Result Publishing App on Kubernetes

A three-tier app deployed to Kubernetes:

```
Browser ──► frontend (React + nginx) ──/api──► backend (Django + gunicorn) ──► PostgreSQL
                                                          └──────────────────► Redis (cache)
```

- **Students** look up a result by Student ID. Lookups are cached in Redis.
- **Teachers** sign in and upload a CSV of results ([backend/sample_results.csv](backend/sample_results.csv)).

## Kubernetes resources

All resources are in the `result-app` namespace and are applied together with [k8s/kustomization.yaml](k8s/kustomization.yaml).

| File | Resources |
|------|-----------|
| [k8s/namespace.yaml](k8s/namespace.yaml) | Namespace `result-app` |
| [k8s/configmap.yaml](k8s/configmap.yaml) | ConfigMap `backend-config` (non-secret settings) |
| [k8s/postgres.yaml](k8s/postgres.yaml) | Headless Service `postgres`, StatefulSet `postgres`, PVC `data-postgres-0` (1Gi) |
| [k8s/redis.yaml](k8s/redis.yaml) | Headless Service `redis`, StatefulSet `redis` with AOF persistence, PVC `data-redis-0` (1Gi) |
| [k8s/backend.yaml](k8s/backend.yaml) | Job `django-migrate`, Deployment `backend` (2 replicas), ClusterIP Service `backend:8000` |
| [k8s/frontend.yaml](k8s/frontend.yaml) | Deployment `frontend`, LoadBalancer Service `frontend:8080` |
| [k8s/kustomization.yaml](k8s/kustomization.yaml) | Generates Secrets `django-secret` and `postgres-credentials`, sets the image names |

**Configuration**
- **ConfigMap:** the DB and Redis hosts are Service DNS names, `postgres.result-app.svc.cluster.local` and `redis.result-app.svc.cluster.local`.
- **Secrets:** the Django secret key, teacher account and PostgreSQL credentials are generated from `k8s/secrets/*.env`. Those files are git-ignored; only the `*.env.example` templates are in the repo.
- **Internal Services:** PostgreSQL, Redis and the backend are not exposed outside the cluster. Only the frontend Service is.
- **Migrations:** the `django-migrate` Job runs the database migrations and creates the teacher account. Backend pods only serve traffic, and their readiness probe calls `/api/health/`, which checks both PostgreSQL and Redis.
- **Frontend proxy:** nginx forwards `/api`, `/admin` and `/static` to `backend.result-app.svc.cluster.local:8000`.

**Images (Docker Hub)**
- `docker.io/abip007/result-backend:1.0`
- `docker.io/abip007/result-frontend:1.0`

## Deployment, step by step

### Step 1: Check the cluster and storage

```bash
kubectl get nodes
kubectl get storageclass          # a (default) StorageClass must exist for the PVCs
```

### Step 2: Build and publish the images (optional)

The manifests already use the published images above. To use your own, build and push them, then edit `images:` in `k8s/kustomization.yaml`:

```bash
docker build -t docker.io/<user>/result-backend:1.0  ./backend
docker build -t docker.io/<user>/result-frontend:1.0 ./frontend
docker push docker.io/<user>/result-backend:1.0
docker push docker.io/<user>/result-frontend:1.0
```

### Step 3: Create the secret files

```bash
cp k8s/secrets/django.env.example   k8s/secrets/django.env
cp k8s/secrets/postgres.env.example k8s/secrets/postgres.env
```

Edit both files and replace every `change-me` value with a strong random value:
- `django.env`: `DJANGO_SECRET_KEY` and `DJANGO_SUPERUSER_PASSWORD` (the teacher password)
- `postgres.env`: `POSTGRES_PASSWORD`

### Step 4: Preview and apply with Kustomize

```bash
kubectl kustomize k8s             # optional: preview the rendered manifests
kubectl apply -k k8s
```

### Step 5: Wait for everything to become ready

```bash
kubectl -n result-app wait --for=condition=complete job/django-migrate --timeout=300s
kubectl -n result-app rollout status statefulset/postgres
kubectl -n result-app rollout status statefulset/redis
kubectl -n result-app rollout status deployment/backend
kubectl -n result-app rollout status deployment/frontend
```

### Step 6: Open the app

- App: **http://localhost:8080**. On Docker Desktop the LoadBalancer is published on localhost.
- Health check: http://localhost:8080/api/health/
- Teacher login: username `teacher`, password = `DJANGO_SUPERUSER_PASSWORD` from `k8s/secrets/django.env`.
- If there is no LoadBalancer, use port-forwarding: `kubectl -n result-app port-forward svc/frontend 8080:8080`.

## Verification

### Workloads, endpoints and PVCs

```bash
kubectl -n result-app get statefulsets,deployments,jobs,pods
kubectl -n result-app get svc,endpoints
kubectl -n result-app get pvc       # data-postgres-0 and data-redis-0 must be Bound
```

### Application

1. Run `curl http://localhost:8080/api/health/`. It should return `{"status":"ok","database":"ok","redis":"ok"}`.
2. Open **Teacher Portal**, sign in as `teacher`, and upload `backend/sample_results.csv`.
3. Under **Check Result**, search `STU001` twice. The first search shows *served from database* and the second *served from cache*. In the backend logs you'll see `cache MISS`, then `cache HIT`:

   ```bash
   kubectl -n result-app logs -l app=backend | grep cache
   ```

### Persistence

PostgreSQL: delete the pod. The StatefulSet recreates it on the same PVC.

```bash
kubectl -n result-app delete pod postgres-0
kubectl -n result-app rollout status statefulset/postgres
kubectl -n result-app exec redis-0 -- redis-cli DEL :1:result:STU001     # force a database read
kubectl -n result-app exec postgres-0 -- sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "select * from results_result;"'
```

Search `STU001` again. It is *served from database* with the same marks.

Redis: the cached keys survive a restart because the append-only file is stored on the PVC.

```bash
kubectl -n result-app delete pod redis-0
kubectl -n result-app rollout status statefulset/redis
kubectl -n result-app exec redis-0 -- redis-cli KEYS '*result:*'
```

## Updating

The Job's pod template can't be changed after it is created. To run migrations again after a new image:

```bash
kubectl -n result-app delete job django-migrate
kubectl apply -k k8s
```

## Teardown

```bash
kubectl delete -k k8s
kubectl -n result-app delete pvc --all     # PVCs created by StatefulSets are not deleted automatically
kubectl delete namespace result-app
```
