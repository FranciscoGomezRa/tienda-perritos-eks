# 🐶 Tienda Perritos — CI/CD + AWS EKS + Autoscaling

Aplicación CRUD de productos (alimentos para perritos) desplegada en **AWS EKS** mediante un
pipeline **CI/CD con GitHub Actions** que construye imágenes, las publica en **Amazon ECR** y las
despliega en el clúster, con **autoscaling de pods (HPA)** y **balanceo de carga (ELB)**.

> Evaluación Parcial N°3 — *Introducción a Herramientas DevOps* (Duoc UC).

---

## 🏗️ Arquitectura

```
                 GitHub (push a main)
                        │
                        ▼
            GitHub Actions (CI/CD)  ──build & push──►  Amazon ECR (3 repos)
                        │                                tienda-frontend
                        │                                tienda-backend
                        │ kubectl apply / set image       tienda-db
                        ▼
        ┌─────────────────────  AWS EKS (namespace: tienda)  ─────────────────────┐
        │                                                                          │
        │   Internet ──► ELB ──► Service(tienda-frontend, LoadBalancer)            │
        │                              │                                           │
        │                              ▼                                           │
        │                     Pods frontend (Nginx)  ──/api/──►  Service backend   │
        │                       [HPA 2..6]                       (ClusterIP)       │
        │                                                            │             │
        │                                                            ▼             │
        │                                                   Pods backend (Node)    │
        │                                                     [HPA 2..10]          │
        │                                                            │             │
        │                                                            ▼             │
        │                                                   Service tienda-db ──► Pod MySQL
        │                                                   (headless)            (Secret) │
        └──────────────────────────────────────────────────────────────────────────┘
   Red: VPC 10.0.0.0/16 en 2 AZ (us-east-1a/1b) · 2 subredes públicas (ELB + NAT) +
   4 privadas (nodos worker, sin IP pública) · NAT Gateway para la salida de las privadas.
```

- **Frontend**: Nginx sirve `index.html` + `app.js` y hace **proxy `/api/` → backend** (`default.conf`).
- **Backend**: Node.js/Express, API REST `/api/productos` (CRUD) + `/api/health`. Se conecta a MySQL.
- **DB**: MySQL 8 con `init.sql` (crea tabla `productos` y la siembra). Contraseña vía **Secret**.
- **Comunicación interna**: por **DNS de Kubernetes** (`tienda-backend`, `tienda-db`).

## 📁 Estructura del repositorio

```
.
├── frontend/        # Nginx + HTML/JS (Dockerfile, default.conf, index.html, app.js)
├── backend/         # Node/Express (Dockerfile, server.js, package.json)
├── db/              # MySQL 8 + init.sql (Dockerfile)
├── k8s/             # Manifiestos Kubernetes (deployments, services, HPA, namespace, secret.example)
├── .github/workflows/deploy-eks.yml   # Pipeline CI/CD
├── .env.example     # Plantilla de variables (copiar a .env, que está en .gitignore)
├── Planificacion Prueba 3.md          # Checklist maestro de la prueba
├── Informe_Prueba3.txt                # Plantilla del informe (con marcadores de FOTO)
├── fotos/           # Capturas para el informe
└── docs_profesor/   # Material original entregado por el profesor (referencia)
```

## ✅ Requisitos previos

- Cuenta **AWS Academy Learner Lab** activa (rol `LabRole`).
- **GitHub** (repositorio + Environment `production` con Secrets).
- Para build local opcional: **Docker Desktop**. Para operar el clúster: **kubectl** (o CloudShell).

## 🔐 Secrets (GitHub → Settings → Environments → `production`)

| Secret | Ejemplo | Notas |
|---|---|---|
| `AWS_ACCESS_KEY_ID` | `ASIA...` | Del *AWS Details* del lab. **Rota cada sesión.** |
| `AWS_SECRET_ACCESS_KEY` | `...` | **Rota cada sesión.** |
| `AWS_SESSION_TOKEN` | `...` | **Rota cada sesión.** |
| `AWS_REGION` | `us-east-1` | Fijo. |
| `EKS_CLUSTER_NAME` | `tienda-eks` | Nombre de tu clúster. |
| `EKS_NAMESPACE` | `tienda` | Namespace. |
| `MYSQL_ROOT_PASSWORD` | `tu_password` | Contraseña root de MySQL (desde `.env`). |

> ⚠️ Las credenciales del Learner Lab son **temporales**: al reiniciar el lab debes
> actualizar `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` y `AWS_SESSION_TOKEN`.

## 🚀 Despliegue automático (CI/CD)

1. Crea el clúster EKS, los nodos y los 3 repos ECR (ver `Planificacion Prueba 3.md`, Fases 1–2).
2. Configura los Secrets del Environment `production`.
3. Haz `push` a `main` (o lanza el workflow manualmente). El pipeline:
   build → push a ECR → `kubectl apply`/`set image` de DB, backend y frontend → Metrics Server → HPA.
4. Obtén la URL pública: `kubectl get svc tienda-frontend -n tienda` → abre el `EXTERNAL-IP`.

## 🛠️ Despliegue manual (alternativa, desde CloudShell)

```bash
aws eks update-kubeconfig --region us-east-1 --name tienda-eks
kubectl apply -f k8s/namespace.yaml

# Secret desde tu .env (NO se versiona):
kubectl create secret generic mysql-secret \
  --from-literal=MYSQL_ROOT_PASSWORD="TU_PASSWORD" \
  -n tienda --dry-run=client -o yaml | kubectl apply -f -

# Reemplaza 000000000000 por tu AWS Account ID en los 3 *-deployment.yaml, luego:
kubectl apply -f k8s/mysql-deployment.yaml -f k8s/mysql-service.yaml -n tienda
kubectl apply -f k8s/backend-deployment.yaml -f k8s/backend-service.yaml -n tienda
kubectl apply -f k8s/frontend-deployment.yaml -f k8s/frontend-service.yaml -n tienda
kubectl apply -f k8s/backend-hpa.yaml -f k8s/frontend-hpa.yaml -n tienda
```

## 📈 Autoscaling (HPA)

- `k8s/backend-hpa.yaml`: 2→10 réplicas al 70% CPU.
- `k8s/frontend-hpa.yaml`: 2→6 réplicas al 60% CPU.
- Requiere **Metrics Server** (lo instala el pipeline). Verifica: `kubectl get hpa -n tienda`.

## 🔎 Comandos útiles

```bash
kubectl get ns
kubectl get nodes -o wide
kubectl get pods -n tienda -o wide
kubectl get svc -n tienda
kubectl logs <pod> -n tienda
kubectl delete pod <pod> -n tienda   # demuestra autorrecuperación
```
