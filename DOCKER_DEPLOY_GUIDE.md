# Guía de Arquitectura Docker y Despliegue CI/CD

Este documento explica la configuración de contenedores y el flujo de automatización para el despliegue del proyecto en AWS EC2.

## 1. Arquitectura de Dockerfiles

Todos los servicios utilizan **Multi-stage builds** (construcción en múltiples etapas) para optimizar el tamaño de las imágenes finales y mejorar la seguridad.

### A. Backend (Spring Boot)
*Archivos: `back-Despachos_SpringBoot/.../dockerfile` y `back-Ventas_SpringBoot/.../dockerfile`*

1.  **Etapa de Construcción (Build)**:
    *   Usa una imagen de `maven:3.9.6-eclipse-temurin-17`.
    *   Copia el código fuente y el archivo `pom.xml`.
    *   Ejecuta `mvn clean package` para generar el archivo `.jar`.
2.  **Etapa de Ejecución (Run)**:
    *   Usa una imagen ligera de `eclipse-temurin:17-jre`.
    *   Copia únicamente el `.jar` generado en la etapa anterior.
    *   **Puertos**: Expone el puerto `8080` (Ventas) u `8081` (Despachos).
    *   **Seguridad**: Crea un usuario `spring` sin privilegios para ejecutar la aplicación.

### B. Frontend (React + Vite)
*Archivo: `front_despacho/dockerfile`*

1.  **Etapa de Construcción**:
    *   Usa `node:20` para instalar dependencias y compilar el proyecto (`npm run build`).
2.  **Etapa de Servicio**:
    *   Usa `nginx:alpine` para servir los archivos estáticos de la carpeta `dist`.
    *   **Puerto**: Expone el puerto `80`.

---

## 2. Orquestación con Docker Compose

### Producción (`docker-compose.prod.yml`)
Este archivo está diseñado para ejecutarse en la **EC2 Privada**. A diferencia del archivo de desarrollo, este no construye las imágenes localmente, sino que las descarga de **Docker Hub**.

*   **Servicios**: Incluye las bases de datos MySQL y los microservicios de Backend.
*   **Variables de Entorno**: Utiliza `${DB_ROOT_PASSWORD}` y `${DOCKERHUB_USERNAME}` para evitar valores fijos en el código.
*   **Redes**: Define una red `prod-network` para la comunicación interna entre contenedores.

---

## 3. Flujo de Despliegue (GitHub Actions)

El archivo `.github/workflows/deploy.yml` automatiza todo el proceso al detectar cambios en las ramas `produccion`, `Dev` o `deploy`.

### Job 1: Build & Push
*   Compila las aplicaciones (Maven para back, Node para front).
*   Construye las imágenes de Docker.
*   Inicia sesión en Docker Hub y sube (`push`) las imágenes con el tag `:latest`.

### Job 2: Deploy Frontend (EC2 Pública)
*   Se conecta vía **SSH** a la instancia pública de AWS.
*   Descarga la nueva imagen del frontend desde Docker Hub.
*   Detiene y elimina el contenedor anterior.
*   Inicia el nuevo contenedor mapeando el puerto `80:80`.

### Job 3: Deploy Backend & DB (EC2 Privada)
*   Configura credenciales de AWS.
*   Sube el archivo `docker-compose.prod.yml` a un bucket S3 temporal.
*   Usa **AWS Systems Manager (SSM)** para enviar comandos a la instancia privada:
    1. Descargar el archivo compose desde S3.
    2. Crear un archivo `.env` con las credenciales necesarias.
    3. Ejecutar `docker-compose pull` y `docker-compose up -d`.

---

## 4. Requisitos de Seguridad (Secrets)

Para que el despliegue funcione, el repositorio de GitHub debe tener configurados los siguientes **Secrets**:

*   `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN`: Para subir/bajar imágenes.
*   `EC2_PUBLIC_IP` / `EC2_PRIVATE_KEY`: Para el acceso SSH a la instancia pública.
*   `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`: Para interactuar con S3 y SSM.
*   `EC2_PRIVATE_INSTANCE_ID`: El ID de la instancia privada en AWS.
*   `DB_ROOT_PASSWORD`: Contraseña maestra para las bases de datos MySQL.


