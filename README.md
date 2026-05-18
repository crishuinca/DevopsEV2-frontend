# InnovaTech — Frontend (React + Nginx)

**Descripción**  
SPA en React/Vite para gestión de ventas y despachos. Se empaqueta con Docker multi-stage, se publica en **Amazon ECR** y se despliega en una **EC2 frontend** mediante GitHub Actions. Nginx hace proxy de `/api/v1/*` hacia los backends en la subred privada.

---

## 🧭 Estructura del proyecto

```
DevopsEV2-frontend/
├── .github/workflows/deploy.yml
├── src/
│   ├── componentes/CrudAdmin/    # Tablas y formularios
│   └── Routes/
├── Dockerfile                      # node build + nginx-unprivileged
├── nginx.conf                      # listen 8080 (local)
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
- Para despliegue AWS: Learner Lab, secrets en GitHub (ver repo `infra`)

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

### Despliegue en AWS

1. Infra aplicada (`infra` etapa_1 + etapa_2).
2. Secrets configurados en GitHub (ECR, EC2, `BACKEND_HOST`).
3. Merge o push a la rama **`deploy`** → workflow build → ECR → SSH en EC2 frontend.

---

## 📦 ¿Qué despliega este proyecto?

| Entorno | Componente | Detalle |
|---------|------------|---------|
| Local | Contenedor `frontend` | Nginx sin root, puerto interno **8080**, host **80** |
| Local | MySQL (compose) | Named volumes `ventas-data`, `despachos-data` |
| AWS | EC2 frontend | `docker run -p 80:8080` + bind mount de `default.conf` |
| AWS | Proxy | `/api/v1/ventas` → `BACKEND_HOST:8081`, despachos → `:8082` |

**Imagen:** `nginxinc/nginx-unprivileged:alpine` + artefactos `dist` de Vite.

---

## 🧭 Diagrama de arquitectura

```
Internet → EC2 Frontend (puerto 80)
              ├── Nginx :8080 (SPA estática)
              └── proxy /api → EC2 Backend (IP privada)
                    ├── :8081 API Ventas
                    └── :8082 API Despachos
```

---

## 📌 Mejores prácticas incluidas

**Multi-stage build:** etapa `node:20-alpine` compila; etapa final solo Nginx + `dist`.

**Usuario no-root:** imagen `nginx-unprivileged`; en EC2 se publica con `-p 80:8080`.

**Volúmenes**

| Tipo | Dónde | Motivo |
|------|--------|--------|
| **Named volume** | `docker-compose.yml` → MySQL (`ventas-data`, `despachos-data`) | Persistir datos de BD al recrear contenedores; Docker gestiona el almacenamiento. |
| **Bind mount** | `deploy.yml` → `/home/ec2-user/nginx/default.conf` | Inyectar configuración de proxy (`BACKEND_HOST`) sin reconstruir la imagen (`:ro`). |

**CI/CD:** push a `deploy` → build → push ECR → SSH → `docker pull` + `docker run`.

**Secrets:** `AWS_*`, `ECR_REGISTRY`, `EC2_HOST`, `EC2_USER`, `SSH_PRIVATE_KEY`, `BACKEND_HOST` (IP privada del backend).

---

## 🔧 Cómo extender este proyecto

- Añadir HTTPS (ACM + ALB o certificado en Nginx).
- Variables de entorno en build Vite (`VITE_API_URL`) para distintos ambientes.
- Healthcheck en el contenedor y en el workflow.
- Named volume en EC2 para logs de Nginx si se requiere auditoría.
