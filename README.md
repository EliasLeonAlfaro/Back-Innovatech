Crea este archivo en la raíz de tu repositorio de Backend.

```markdown
# ⚙️ Innovatech Chile - Backend Microservices & CI/CD Pipeline

Este repositorio contiene la lógica de negocio de la plataforma **Innovatech Chile**, estructurada mediante dos microservicios independientes y desacoplados utilizando **Spring Boot**:
* **Microservicio de Despachos** (Puerto de escucha: `8085`)
* **Microservicio de Ventas** (Puerto de escucha: `8086`)

Ambos servicios están diseñados bajo una arquitectura *Stateless* (sin estado), lo que permite destruirlos, actualizarlos y escalarlos horizontalmente de manera ágil a través de un flujo automatizado de DevOps.

---

## 🤖 Funcionamiento del Pipeline de CI/CD (GitHub Actions)

El ciclo de vida del código está completamente automatizado mediante el flujo definido en `.github/workflows/deploy.yml`. Cada vez que se realiza un `push` a la rama `deploy`, se gatillan de forma paralela los siguientes pasos de integración y despliegue continuo:

1. **Integración Continua (CI):** Un entorno aislado de GitHub levanta un corredor virtual, inicia sesión en **Docker Hub** de manera segura y construye las imágenes Docker inmutables para cada microservicio (`back_despachos:latest` y `back_ventas:latest`), subiéndolas al registro.
2. **Despliegue Continuo (CD) - Bastion SSH Proxy:** Debido a que la EC2 del Backend se encuentra resguardada en una subred privada de AWS sin acceso a internet, el pipeline utiliza la acción `appleboy/ssh-action` para establecer una conexión segura. Utiliza la EC2 del Frontend como un **Bastion Host (Proxy)**, abriendo un túnel SSH cifrado hacia la IP privada de la máquina de destino.
3. **Idempotencia y Limpieza:** El script en la EC2 detiene y elimina de forma preventiva los contenedores viejos liberando los puertos de red, descarga las nuevas capas desde Docker Hub y levanta las aplicaciones en segundos, minimizando el *downtime*.

---

## 🔐 Configuración de Seguridad y Variables de Entorno

El pipeline no almacena contraseñas en texto plano. Utiliza **GitHub Secrets** para inyectar las credenciales directamente en la memoria RAM de los contenedores en tiempo de ejecución (`runtime`).

Si necesitas replicar el despliegue de manera manual, es obligatorio proveer las siguientes variables de entorno:

```env
DOCKERHUB_USERNAME=tu_usuario_docker
DB_ENDPOINT=10.0.X.X       # IP Privada de la EC2 donde corre MySQL
DB_NAME=db_innovatech      # Nombre de la base de datos
DB_USERNAME=api_user           # Usuario de base de datos
DB_PASSWORD=api_password_segura    # Contraseña configurada en el motor

## Para ejecutar los contenedores de forma directa en la instancia  
docker run -d \
  --name back_despachos_container \
  -p 8085:8085 \
  --restart always \
  -e SERVER_PORT=8085 \
  -e SPRING_DATASOURCE_URL="jdbc:mysql://$DB_ENDPOINT:3306/$DB_NAME?allowPublicKeyRetrieval=true&useSSL=false&serverTimezone=UTC" \
  -e SPRING_DATASOURCE_USERNAME="$DB_USERNAME" \
  -e SPRING_DATASOURCE_PASSWORD="$DB_PASSWORD" \
  "$DOCKERHUB_USERNAME/back_despachos:latest"

docker run -d \
  --name back_ventas_container \
  -p 8086:8086 \
  --restart always \
  -e SERVER_PORT=8086 \
  -e SPRING_DATASOURCE_URL="jdbc:mysql://$DB_ENDPOINT:3306/$DB_NAME?allowPublicKeyRetrieval=true&useSSL=false&serverTimezone=UTC" \
  -e SPRING_DATASOURCE_USERNAME="$DB_USERNAME" \
  -e SPRING_DATASOURCE_PASSWORD="$DB_PASSWORD" \
  "$DOCKERHUB_USERNAME/back_ventas:latest"