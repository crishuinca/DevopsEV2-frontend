# Sistema de Gestión InnovaTech - Configuración de Infraestructura y Despliegue DevOps

Este repositorio contiene la configuración de infraestructura, contenedorización y automatización CI/CD para el proyecto semestral **InnovaTech**, correspondiente a la Evaluación Parcial N°2 de la asignatura *Introducción a Herramientas DevOps (ISY1101)* en Duoc UC.

El ecosistema está compuesto por tres microservicios distribuidos:
1. **Frontend:** Aplicación SPA construida en React y Vite (`front_despacho`).
2. **Backend Ventas:** API REST en Spring Boot (`back-Ventas_SpringBoot`).
3. **Backend Despachos:** API REST en Spring Boot (`back-Despachos_SpringBoot`).


## 🛠️ Requisitos Previos

Antes de levantar el proyecto de forma local o en la nube, asegúrate de tener instalado:
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (v20.10 o superior)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2.0 o superior)
- Git

## Arquitectura de Despliegue en AWS Academy

El ecosistema se despliega de forma unificada utilizando el entorno de **AWS Academy (Learner Labs)** sobre una única instancia **EC2 (Ubuntu Server)**. Esta decisión optimiza el uso de y centraliza la orquestación a través de Docker.

### Configuración de Redes y Security Groups (AWS)
Por estrictas políticas de seguridad aplicadas al entorno de producción, el **Security Group** asignado en la consola de AWS solo expone los siguientes puertos hacia Internet (*Inbound Rules*):

| Puerto | Protocolo | Origen | Descripción |
| :--- | :--- | :--- | :--- |
| `22` | SSH | Mi IP / GitHub Actions | Acceso administrativo y despliegue automatizado. |
| `80` | HTTP | `0.0.0.0/0` (Cualquiera) | Acceso público de usuarios al contenedor Frontend. |

> ⚠️ **Nota de Seguridad:** Los puertos internos de las APIs Backend (`8081`, `8082`) y de las bases de datos MySQL (`3306`) **NO están abiertos en AWS**. La comunicación se realiza de forma interna y segura a través de la red aislada de Docker.


## 🐳 Justificación Técnica de Contenedorización

Cumpliendo con los estándares de la industria y las exigencias de la rúbrica de evaluación, el diseño de contenedores implementa las siguientes directrices esenciales:

### 1. Multi-stage Builds (Construcción en Etapas)
Tanto en el Frontend como en ambos Backends de Spring Boot, se han configurado archivos `Dockerfile` multiplataforma y multi-etapa. 
- **Fase de Compilación:** Utiliza imágenes robustas de desarrollo (`node:18-alpine` / `maven:3.9.4-eclipse-temurin`) para compilar el código fuente y generar los artefactos descargables (`dist` / `.jar`).
- **Fase de Producción:** Descarta todas las herramientas de compilación pesadas y empaqueta el binario final en imágenes ultralivianas de ejecución (`nginxinc/nginx-unprivileged:alpine` / `eclipse-temurin:17-jre-alpine`). Esto reduce drásticamente el tamaño de las imágenes finales y limita el número de dependencias con vulnerabilidades.

### 2. Principio de Mínimo Privilegio (Usuario No-Root)
Por defecto, los procesos de Docker se ejecutan como `root`, lo que representa un riesgo crítico de seguridad. 
- En las imágenes de backend, se crean explícitamente un grupo y usuario del sistema (`spring`), delegando el control del proceso mediante la directiva `USER spring:spring`.
- En el frontend, se utiliza la variante oficial desprivilegiada de Nginx (`nginx-unprivileged`), la cual expone nativamente el puerto interno `8080` (ya que los usuarios tradicionales tienen prohibido enlazar puertos del sistema inferiores al `1024`).

### 3. Persistencia de Datos (Named Volumes)
Las bases de datos relacionales asociadas a Ventas y Despachos emplean **Volúmenes Nombrados de Docker (`ventas-data` y `despachos-data`)** mapeados en el directorio `/var/lib/mysql`. Esto garantiza que, ante una detención, actualización de la imagen o falla del contenedor, la información transaccional permanezca persistida de forma segura en el almacenamiento físico del host (EC2).



## 📦 Instrucciones de Ejecución Local y Conjunta

Para levantar todo el ecosistema de microservicios de forma conjunta y en segundo plano, sitúate en la raíz del proyecto (donde se encuentra el archivo maestro `docker-compose.yml`) y ejecuta:

```bash
docker-compose up -d

#Verificación de contenedores activos:
    docker ps

    Puertos Locales Mapeados: Frontend (Web): http://localhost:80

    API Ventas: http://localhost:8081

    API Despachos: http://localhost:8082

#Para detener y limpiar la red de servicios:

    docker-compose down

#Pipeline CI/CD: Automatización con GitHub Actions Cada repositorio integra un flujo automatizado de integración y despliegue continuo configurado en .github/workflows/deploy.yml.

#Disparador (Triggers) El flujo se activa únicamente cuando se realiza un evento push o merge hacia la rama deploy.

#Etapas del Pipeline:

    Checkout Código: Descarga el código fuente actualizado del microservicio en el ejecutor de GitHub.

    Autenticación Docker Hub: Inicio de sesión seguro utilizando secretos cifrados.

    Build & Push: Construcción de la imagen Docker optimizada mediante la acción nativa de Docker y subida al registro con el tag :latest.

    Despliegue Remoto en EC2 (SSH):

    Se conecta mediante SSH a la instancia de AWS Academy.

    Ejecuta un docker-compose pull para descargar la versión más reciente de las imágenes publicadas.

    Recrea los contenedores modificados mediante docker-compose up -d --remove-orphans, garantizando cero tiempo de inactividad percibido por el usuario.

    Secretos requeridos en el repositorio de GitHub: Para que las acciones se ejecuten correctamente, se deben registrar las siguientes variables en Settings -> Secrets and variables -> Actions:

    DOCKER_USERNAME: Nombre de usuario de Docker Hub.

    DOCKER_PASSWORD: Token de acceso personal o contraseña de Docker Hub.

    EC2_HOST: Dirección IP pública de la instancia en AWS Academy.

    EC2_USERNAME: Usuario por defecto de la AMI (ubuntu).

    EC2_SSH_KEY: Contenido completo de la llave privada privada .pem proporcionada por AWS.
