# ✅ Planificación Prueba 3 — CI/CD + EKS + Autoscaling (Tienda Perritos)

Checklist maestro. Marca `[x]` a medida que avances. Las 📸 indican capturas para el informe
(guárdalas en `fotos/` con el nombre sugerido). Mapeo a indicadores de la rúbrica: **IE1–IE7**.

**Decisiones del proyecto:**
- Orquestación: **AWS EKS** (no ECS). Provisión por **consola AWS Academy**; `kubectl` por **CloudShell**.
- Entrega: **monorepo** (este repo). Registro de imágenes: **ECR** (sin DockerHub).
- Autoscaling: **HPA de pods** (Cluster Autoscaler queda como mejora futura, no se implementa).
- Región: **us-east-1**. Clúster: **tienda-eks**. Namespace: **tienda**.
- Secrets: en GitHub **Environment `production`**; credenciales de BD desde `.env` (gitignored).

---

## Fase 0 — Preparación del repositorio (✔ hecho por Claude)
- [x] Promover la app `3.6.3` a la raíz del repo (los 3 repos del profesor eran idénticos).
- [x] Archivar material del profesor en `docs_profesor/` y eliminar carpetas duplicadas.
- [x] Fix bug `backend/server.js` (`DB_HOST = "tienda-db"`).
- [x] Quitar credenciales horneadas en `db/Dockerfile` (password va por runtime/Secret).
- [x] Manifiestos: URIs de ECR como placeholder `000000000000`; quitar anotaciones LB del frontend.
- [x] Secret de MySQL fuera del repo → `k8s/mysql-secret.example.yaml` + generación en el pipeline.
- [x] Crear `.gitignore`, `.env.example`, `README.md`, este checklist e `Informe_Prueba3.txt`.
- [ ] **Tú:** copiar `.env.example` a `.env` y poner tu `MYSQL_ROOT_PASSWORD`.

## Fase 1 — Infraestructura AWS por consola  [IE1]
> **Diseño elegido: patrón de producción (B)** — nodos en subredes **privadas** + **NAT Gateway**.
> ELB en públicas, nodos sin IP pública (seguridad). Se argumentan ambos caminos en la defensa.
> ⚠️ El NAT consume créditos: hacer **Stop Lab** al terminar y borrar la VPC al cerrar el proyecto.

- [ ] Iniciar Learner Lab; copiar **AWS Details** (access key, secret, **session token**). Account ID: `268450856324`.
- [ ] Crear **VPC** (`tienda-vpc`, asistente "VPC and more", `10.0.0.0/16`, **2 AZ**):
      **2 subredes públicas + 4 privadas**, **NAT gateways: In 1 AZ** (1 NAT para ahorrar créditos),
      VPC endpoints: None, DNS: ambas marcadas.
- [ ] **Etiquetar subredes** (para que EKS ubique bien los balanceadores):
      - Públicas: `kubernetes.io/role/elb = 1` + `kubernetes.io/cluster/tienda-eks = shared`
      - Privadas: `kubernetes.io/role/internal-elb = 1` + `kubernetes.io/cluster/tienda-eks = shared`
- [ ] Revisar Security Groups (el ELB crea el suyo; EKS gestiona reglas nodo↔ELB).
- [ ] Crear **clúster EKS** `tienda-eks` con rol **`LabRole`**; en *Networking* seleccionar las
      **6 subredes** (públicas para ELB/ENIs, privadas para nodos/ENIs). Endpoint access: Public.
- [ ] Crear **Node Group** (rol `LabRole`, `t3.medium`, desired=2, min=2, max=4) **en las 4 (o 2) subredes PRIVADAS**.
- [ ] 📸 `IE1_01_vpc_subredes_2az.png` — las 6 subredes (2 públicas + 4 privadas) en 2 AZ.
- [ ] 📸 `IE1_00_nat_gateway.png` — el NAT Gateway creado (evidencia del diseño).
- [ ] 📸 `IE1_02_eks_cluster_active.png` — clúster en estado **Active**.
- [ ] 📸 `IE1_03_nodegroup.png` — node group **Active** / nodos `Ready`.
- [ ] 📸 `IE1_04_nodos_ec2_ip_publica.png` — instancias EC2 con **IP pública**.

## Fase 2 — Amazon ECR  [IE2]
- [ ] Crear 3 repos ECR: `tienda-frontend`, `tienda-backend`, `tienda-db` (us-east-1).
- [ ] Anotar tu **AWS Account ID** (12 dígitos).
- [ ] (Opcional) Reemplazar `000000000000` por tu Account ID en los `k8s/*-deployment.yaml`.
- [ ] 📸 `IE2_01_ecr_repos.png` — los 3 repositorios creados.

## Fase 3 — GitHub: repo + Environment + Secrets  [IE4, IE5]
- [ ] Crear repositorio en GitHub y subir el código (Claude prepara commits + push).
- [ ] Settings → **Environments** → crear **`production`**.
- [ ] Cargar Secrets en `production`: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`,
      `AWS_SESSION_TOKEN`, `AWS_REGION`, `EKS_CLUSTER_NAME`, `EKS_NAMESPACE`, `MYSQL_ROOT_PASSWORD`.
- [ ] 📸 `IE5_01_github_environment.png` — Environment `production` creado.
- [ ] 📸 `IE5_02_github_secrets.png` — lista de Secrets (valores ocultos).

## Fase 4 — Ejecutar el pipeline CI/CD  [IE4]
- [ ] Hacer `push` a `main` (o lanzar el workflow manualmente).
- [ ] Verificar que cada step pasa (build → push → deploy).
- [ ] 📸 `IE4_01_workflow_verde.png` — ejecución del workflow en verde.
- [ ] 📸 `IE2_02_ecr_imagenes.png` — imágenes con tag dentro de los repos ECR.

## Fase 5 — Verificar despliegue en EKS  [IE2, IE3]
- [ ] (Si HPA muestra `<unknown>`) confirmar **Metrics Server** corriendo.
- [ ] 📸 `IE2_03_get_pods_running.png` — `kubectl get pods -n tienda` todos **Running**.
- [ ] 📸 `IE3_01_get_hpa.png` — `kubectl get hpa -n tienda` con targets (no `<unknown>`).

## Fase 6 — Operación desde CloudShell  [pizarra punto 4, IE7]
- [ ] `aws eks update-kubeconfig --region us-east-1 --name tienda-eks`
- [ ] `kubectl get ns` · `kubectl get nodes` · `kubectl get pods -n tienda` · `kubectl get svc -n tienda`
- [ ] Borrar un pod y ver cómo se recrea: `kubectl delete pod <pod> -n tienda` + `kubectl get pods -n tienda -w`
- [ ] 📸 `IE7_01_cloudshell_get_ns_nodes.png` — salida de `get ns` / `get nodes`.
- [ ] 📸 `IE7_02_pod_recreado.png` — pod borrado y Kubernetes recreándolo.

## Fase 7 — Load Balancer y URL pública  [IE2]
- [ ] `kubectl get svc tienda-frontend -n tienda` → copiar `EXTERNAL-IP` (DNS del ELB).
- [ ] Abrir el DNS en el navegador → se ve la Tienda Perritos con productos.
- [ ] 📸 `IE2_04_external_ip.png` — el `EXTERNAL-IP` del service.
- [ ] 📸 `IE2_05_app_navegador.png` — la app cargando productos en el navegador.

## Fase 8 — Demostrar Autoscaling (HPA)  [IE3]
- [ ] Generar carga al backend y observar `kubectl get hpa -n tienda -w` escalar réplicas.
- [ ] 📸 `IE3_02_hpa_escalando.png` — HPA subiendo réplicas (2→N).
- [ ] 📸 `IE3_03_pods_nuevos.png` — `kubectl get pods` con réplicas nuevas.
- [ ] 📝 Anotar justificación del umbral (70% CPU backend / 60% frontend).

## Fase 9 — Logs y métricas  [IE6]
- [ ] `kubectl logs <pod-backend> -n tienda` (y/o panel **CloudWatch**).
- [ ] 📸 `IE6_01_logs.png` — `kubectl logs` (o CloudWatch).
- [ ] 📸 (opcional) `IE6_02_cloudwatch.png` — métricas en CloudWatch.

## Fase 10 — Validación funcional y recuperación  [IE7]
- [ ] Probar CRUD completo en la web (crear/editar/eliminar producto).
- [ ] Hacer un nuevo commit → ver redeploy automático y `rollout` exitoso.
- [ ] 📸 `IE7_03_crud_ok.png` — CRUD funcionando (Front↔Back).
- [ ] 📸 `IE7_04_redeploy.png` — rollout/redeploy tras commit.

## Fase 11 — Documentación e informe
- [ ] Verificar `README.md` actualizado.
- [ ] Rellenar `Informe_Prueba3.txt` con análisis y pegar las fotos de `fotos/`.
- [ ] Commits explicativos en todo el historial (la rúbrica los valora).
- [ ] Entregar el repositorio por **AVA**.

---

## Verificación final ("listo" cuando todo esto se cumple)
- [ ] IE1 — Clúster EKS Active + node group en 2 AZ.
- [ ] IE2 — 3 imágenes en ECR; pods Front/Back/DB Running; URL pública sirve la app.
- [ ] IE3 — HPA con métricas y escalando bajo carga.
- [ ] IE4 — Pipeline en verde, deploy automático tras commit.
- [ ] IE5 — Secrets en GitHub Environment + Secret de k8s, sin credenciales en el código.
- [ ] IE6 — Evidencia de logs/métricas.
- [ ] IE7 — CRUD Front↔Back operativo + autorrecuperación de pod demostrada.
