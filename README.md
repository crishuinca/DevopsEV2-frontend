# InnovaTech — Frontend (React + Nginx)

**Descripción**  
SPA en React/Vite para gestión de ventas y despachos. Se empaqueta con Docker multi-stage, se publica en **Amazon ECR** y se despliega en **EKS** mediante el pipeline central del repo `DevopsEV2-infra`. Nginx hace proxy de `/api/v1/*` hacia los backends por DNS interno del clúster (`backend-ventas`, `backend-despachos`).

---

## 🧭 Estructura del proyecto

```
DevopsEV2-frontend/
├── src/
│   ├── componentes/CrudAdmin/    # Tablas y formularios
│   └── Routes/
├── Dockerfile                      # node build + nginx-unprivileged
├── nginx.conf                      # listen 8080; proxy a backends K8s
├── docker-compose.yml              # stack local front + backends + MySQL
├── package.json
└── README.md
```

---

## 🚀 Requisitos

- Docker Desktop / Docker Engine >= 20.10
- Docker Compose v2
- Node.js 20+ (solo si desarrollas sin Docker)
- Git
- Para despliegue AWS: Learner Lab, infra aplicada (`etapa_1` + `etapa_3`) y secrets en `DevopsEV2-infra`

---

## ⚙️ Flujo de uso

### Local (ecosistema completo)

1. Clona el repositorio.
2. Desde la raíz del frontend:

```bash
docker compose up -d --build
```

3. Abre **http://localhost** (mapeo `80:8080`).
4. APIs vía proxy nginx: `/api/v1/ventas`, `/api/v1/despachos`.

Detener:

```bash
docker compose down
```

### Despliegue en AWS (EV3 — EKS)

1. Infra aplicada en `DevopsEV2-infra` (`etapa_1` + `etapa_3`).
2. Secrets AWS configurados en **DevopsEV2-infra** (no en este repo).
3. Push a la rama **`deploy`** en **DevopsEV2-infra** → workflow `cd.yml` → build imagen → push ECR → deploy en EKS.

Obtener URL pública:

```bash
kubectl get svc frontend
```

> El despliegue AWS se dispara únicamente desde **DevopsEV2-infra** (rama `deploy`).

---

## 📦 ¿Qué despliega este proyecto?

| Entorno | Componente | Detalle |
|---------|------------|---------|
| Local | Contenedor `frontend` | Nginx sin root, puerto interno **8080**, host **80** |
| Local | MySQL (compose) | Named volumes `ventas-data`, `despachos-data` |
| AWS (EKS) | Pod `frontend` | Nginx `:8080`, expuesto por Service **LoadBalancer** `:80` |
| AWS (EKS) | Proxy | `/api/v1/ventas` → `backend-ventas:8080`, despachos → `backend-despachos:8081` |

**Imagen:** `nginxinc/nginx-unprivileged:alpine` + artefactos `dist` de Vite.  
**ECR:** `innovatech-frontend`

---

## 🧭 Diagrama de arquitectura

```
Internet → Service frontend (LoadBalancer :80)
              │
              ▼
         Pod frontend (Nginx :8080, SPA estática)
              ├── proxy /api/v1/ventas   → backend-ventas:8080
              └── proxy /api/v1/despachos → backend-despachos:8081
```

```
DevopsEV2-infra (cd.yml, rama deploy)
        ├── checkout este repo
        ├── docker build + push → ECR
        └── kubectl set image deployment/frontend
```

---

## 📌 Mejores prácticas incluidas

**Multi-stage build:** etapa `node:20-alpine` compila; etapa final solo Nginx + `dist`.

**Usuario no-root:** imagen `nginx-unprivileged`; en EKS el contenedor escucha en **8080** sin privilegios root.

**Volúmenes**

| Tipo | Dónde | Motivo |
|------|--------|--------|
| **Named volume** | `docker-compose.yml` → MySQL (`ventas-data`, `despachos-data`) | Persistir datos de BD al recrear contenedores; Docker gestiona el almacenamiento. |
| Sin volumen en EKS | Manifiesto `k8s/frontend.yml` | La SPA es stateless; la configuración de proxy va embebida en la imagen (`nginx.conf`). |

**CI/CD (EV3):** el pipeline central en `DevopsEV2-infra` hace checkout de este repo, build, push ECR y actualiza el deployment en EKS.

**Secrets:** solo en `DevopsEV2-infra` — `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`.

---

## 🔧 Cómo extender este proyecto

- Añadir HTTPS (ACM + Ingress ALB).
- Variables de entorno en build Vite (`VITE_API_URL`) para distintos ambientes.
- Healthcheck (`livenessProbe` / `readinessProbe`) en el manifiesto K8s.
- Aumentar réplicas del deployment frontend para alta disponibilidad.
