# Gitea

Servidor Git autoalojado ligero y rápido, escrito en Go. Alternativa a GitLab y GitHub on-premises, optimizado para consumir pocos recursos.

## Características

- 🚀 **Ligero y rápido**: Escrito en Go, consume menos recursos que GitLab
- 📦 **Repositorios ilimitados**: Sin límites de repos, usuarios u organizaciones
- 👥 **Gestión de usuarios y equipos**: Control de acceso granular
- 🔀 **Pull Requests y Code Review**: Flujo de trabajo completo de desarrollo
- 🐛 **Issue Tracking**: Sistema integrado de gestión de incidencias
- 📊 **Wiki y Proyectos**: Documentación y gestión de tareas
- 🔐 **Autenticación múltiple**: LDAP, OAuth2, OpenID Connect
- 🌐 **Interfaz web intuitiva**: Similar a GitHub/GitLab

## Requisitos Previos

- Docker Engine instalado
- Portainer configurado (recomendado)
- **Para Traefik o NPM**: Red Docker `proxy` creada
- **Dominio configurado**: Para acceso HTTPS
- **DB_PASSWORD generada**: Contraseña segura para PostgreSQL

⚠️ **IMPORTANTE**: Gitea requiere PostgreSQL. Este compose incluye un contenedor PostgreSQL 18 Alpine.

## Generar DB_PASSWORD

**Antes de cualquier despliegue**, genera una contraseña segura para PostgreSQL:

```bash
openssl rand -base64 32
```

Guarda el resultado, lo necesitarás como valor de `DB_PASSWORD`.

> ⚠️ **Importante**: Usa comillas simples en el archivo `.env` s


> Ejemplo: `DB_PASSWORD='tu_password_generado'`

---

## Despliegue con Portainer

### Opción A: Git Repository (Recomendada)

Permite mantener la configuración actualizada automáticamente desde Git.

1. En Portainer, ve a **Stacks** → **Add stack**
2. Nombra el stack: `gitea`
3. Selecciona **Git Repository**
4. Configura:
   - **Repository URL**: `https://git.ictiberia.com/groales/gitea`
   - **Repository reference**: `refs/heads/main`
   - **Compose path**: `docker-compose.yml`
   - **Additional paths**: Solo para Traefik: `docker-compose.override.traefik.yml.example`

5. En **Environment variables**, añade:

   **Para Traefik**:
   ```
   DOMAIN_HOST=gitea.tudominio.com
   DB_PASSWORD='tu_password_generado'
   ```

   **Para NPM**:
   ```
   DB_PASSWORD='tu_password_generado'
   ```

   ⚠️ **Nota para NPM**: No uses Additional paths, el docker-compose.yml base es suficiente.

6. Haz clic en **Deploy the stack**

#### Configuración de WebSocket

- **Traefik**: Ya configurado en el override
- **NPM**: Puede requerir activar **WebSocket Support** para push/pull en tiempo real

### Opción B: Web Editor

Copia y pega el contenido consolidado según tu configuración de proxy.

1. En Portainer, ve a **Stacks** → **Add stack**
2. Nombra el stack: `gitea`
3. Selecciona **Web editor**
4. Copia el contenido según tu entorno:

<details>
<summary>📋 Despliegue con Traefik</summary>

```yaml
services:
  gitea:
    container_name: gitea
    image: gitea/gitea:latest
    restart: unless-stopped
    environment:
      USER_UID: 1000
      USER_GID: 1000
      GITEA__database__DB_TYPE: postgres
      GITEA__database__HOST: gitea-db:5432
      GITEA__database__NAME: ${DB_NAME:-gitea}
      GITEA__database__USER: ${DB_USER:-gitea}
      GITEA__database__PASSWD: ${DB_PASSWORD}
      TZ: Europe/Madrid
    volumes:
      - gitea_data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    networks:
      - proxy
      - gitea-internal
    depends_on:
      - gitea-db
    labels:
      # HTTP → HTTPS redirect
      - "traefik.enable=true"
      - "traefik.http.routers.gitea-http.rule=Host(`${DOMAIN_HOST}`)"
      - "traefik.http.routers.gitea-http.entrypoints=web"
      - "traefik.http.routers.gitea-http.middlewares=redirect-to-https@docker"
      
      # HTTPS router
      - "traefik.http.routers.gitea.rule=Host(`${DOMAIN_HOST}`)"
      - "traefik.http.routers.gitea.entrypoints=websecure"
      - "traefik.http.routers.gitea.tls.certresolver=letsencrypt"
      - "traefik.http.routers.gitea.service=gitea-svc"
      - "traefik.http.services.gitea-svc.loadbalancer.server.port=3000"
      
      # Redirect middleware
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.permanent=true"

  gitea-db:
    container_name: gitea-db
    image: postgres:18-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME:-gitea}
      POSTGRES_USER: ${DB_USER:-gitea}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - gitea_db:/var/lib/postgresql
    networks:
      - gitea-internal

volumes:
  gitea_data:
    name: gitea_data
  gitea_db:
    name: gitea_db

networks:
  proxy:
    external: true
  gitea-internal:
    name: gitea-internal
```

**Variables de entorno necesarias**:
```
DOMAIN_HOST=gitea.tudominio.com
DB_PASSWORD='tu_password_generado'
```

</details>

<details>
<summary>📋 Despliegue con Nginx Proxy Manager</summary>

```yaml
services:
  gitea:
    container_name: gitea
    image: gitea/gitea:latest
    restart: unless-stopped
    environment:
      USER_UID: 1000
      USER_GID: 1000
      GITEA__database__DB_TYPE: postgres
      GITEA__database__HOST: gitea-db:5432
      GITEA__database__NAME: ${DB_NAME:-gitea}
      GITEA__database__USER: ${DB_USER:-gitea}
      GITEA__database__PASSWD: ${DB_PASSWORD}
      TZ: Europe/Madrid
    volumes:
      - gitea_data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    networks:
      - proxy
      - gitea-internal
    depends_on:
      - gitea-db

  gitea-db:
    container_name: gitea-db
    image: postgres:18-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME:-gitea}
      POSTGRES_USER: ${DB_USER:-gitea}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - gitea_db:/var/lib/postgresql
    networks:
      - gitea-internal

volumes:
  gitea_data:
    name: gitea_data
  gitea_db:
    name: gitea_db

networks:
  proxy:
    external: true
  gitea-internal:
    name: gitea-internal
```

**Variables de entorno necesarias**:
```
DB_PASSWORD='tu_password_generado'
```

**⚠️ IMPORTANTE**: Debes configurar en NPM:
1. Crea un Proxy Host apuntando a `gitea:3000`
2. Configura SSL con Let's Encrypt
3. Opcionalmente activa **WebSocket Support** para mejor rendimiento

</details>

5. En **Environment variables**, añade las variables correspondientes
6. Haz clic en **Deploy the stack**

## Despliegue con Docker CLI

### 1. Clonar el repositorio

```bash
git clone https://git.ictiberia.com/groales/gitea.git
cd gitea
```

### 2. Elegir modo de despliegue

#### Opción A: Traefik

```bash
cp docker-compose.override.traefik.yml.example docker-compose.override.yml
cp .env.example .env
# Editar .env y configurar DOMAIN_HOST y DB_PASSWORD
```

#### Opción B: Nginx Proxy Manager

No necesitas archivo override, usa el `docker-compose.yml` base directamente.

Copiar y configurar `.env`:
```bash
cp .env.example .env
# Editar .env y configurar DB_PASSWORD
```

### 3. Iniciar el servicio

```bash
docker compose up -d
```

### 4. Verificar el despliegue

```bash
docker compose logs -f gitea
docker compose logs -f gitea-db
```

## Configuración Inicial

### 1. Acceder a Gitea

Visita `https://gitea.tudominio.com` y completa el asistente de instalación inicial.

### 2. Configuración Recomendada en el Asistente

**Configuración de Base de Datos** (pre-rellenado):
- Tipo: PostgreSQL
- Host: `gitea-db:5432`
- Usuario: `gitea`
- Contraseña: (la que configuraste en `DB_PASSWORD`)
- Nombre: `gitea`

**Configuración General**:
- **Título del sitio**: Nombre de tu instalación
- **URL Base**: `https://gitea.tudominio.com`
- **Ruta de repositorios Git**: `/data/git/repositories` (default)
- **Ruta raíz LFS**: `/data/git/lfs` (default)

**Configuración de Administrador**:
- **Nombre de usuario**: Tu usuario admin
- **Contraseña**: Contraseña fuerte y segura
- **Email**: Tu email

⚠️ **Importante**: 
- Verifica que la URL base sea correcta (con HTTPS)
- El primer usuario creado será automáticamente administrador

### 3. Configuración Post-Instalación

Accede como administrador y configura:

**En Site Administration → Configuration**:
- Deshabilitar auto-registro si no lo necesitas
- Configurar SMTP para notificaciones por email
- Configurar límites de tamaño de repositorios si lo deseas

## Configuración de Email (SMTP)

Para notificaciones y recuperación de contraseñas, edita el archivo de configuración:

```bash
# Acceder al contenedor
docker compose exec gitea bash

# Editar app.ini
vi /data/gitea/conf/app.ini
```

Añade la sección SMTP:

```ini
[mailer]
ENABLED = true
FROM = gitea@tudominio.com
PROTOCOL = smtp
SMTP_ADDR = smtp.tudominio.com
SMTP_PORT = 587
USER = gitea@tudominio.com
PASSWD = tu_password_smtp
```

Reinicia Gitea:
```bash
docker compose restart gitea
```

## Uso Básico

### Crear un Repositorio

1. Inicia sesión en Gitea
2. Haz clic en el **+** en la esquina superior derecha
3. Selecciona **New Repository**
4. Configura nombre, descripción, visibilidad
5. Haz clic en **Create Repository**

### Clonar un Repositorio

```bash
# Con HTTPS
git clone https://gitea.tudominio.com/usuario/repo.git

# Con SSH (requiere configurar clave SSH en Gitea)
git clone git@gitea.tudominio.com:usuario/repo.git
```

### Configurar SSH

1. Genera una clave SSH (si no tienes):
   ```bash
   ssh-keygen -t ed25519 -C "tu_email@ejemplo.com"
   ```

2. Copia la clave pública:
   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```

3. En Gitea: **Settings** → **SSH / GPG Keys** → **Add Key**

## Backup y Restauración

### Backup Manual

```bash
# Detener Gitea (mantener DB corriendo)
docker compose stop gitea

# Backup de datos de Gitea
docker run --rm -v gitea_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/gitea-data-$(date +%Y%m%d).tar.gz -C /data .

# Backup de PostgreSQL
docker compose exec gitea-db pg_dump -U gitea gitea > gitea-db-$(date +%Y%m%d).sql

# Reiniciar Gitea
docker compose start gitea
```

### Restauración

```bash
# Detener servicios
docker compose down

# Restaurar datos de Gitea
docker run --rm -v gitea_data:/data -v $(pwd):/backup alpine \
  sh -c "cd /data && tar xzf /backup/gitea-data-20240101.tar.gz"

# Iniciar solo la DB
docker compose up -d gitea-db

# Esperar a que PostgreSQL esté listo
sleep 10

# Restaurar DB
docker compose exec -T gitea-db psql -U gitea gitea < gitea-db-20240101.sql

# Iniciar todo
docker compose up -d
```

### Backup Automático

Considera usar:
- [docker-volume-backup](https://github.com/offen/docker-volume-backup)
- Cron job con el script de backup manual
- Duplicati para backups regulares

⚠️ **CRÍTICO**: Un servidor Git requiere backups regulares y probados.

## Actualización

### Desde Portainer (Git Repository)

1. Ve a tu stack `gitea`
2. Haz clic en **Pull and redeploy**

### Desde CLI

```bash
docker compose pull
docker compose up -d
```

⚠️ **Nota**: Gitea puede requerir ejecutar migraciones automáticamente al actualizar.

## Solución de Problemas

### Gitea no puede conectar a PostgreSQL

**Síntomas**: Error de conexión a base de datos en el asistente

**Soluciones**:
```bash
# Verificar que ambos contenedores están corriendo
docker compose ps

# Verificar logs de PostgreSQL
docker compose logs gitea-db

# Verificar que están en la misma red
docker network inspect gitea-internal

# Verificar variable DB_PASSWORD
docker compose exec gitea env | grep GITEA__database__PASSWD
```

### Error en instalación inicial

**Síntomas**: El asistente falla al guardar configuración

**Soluciones**:
1. Verifica permisos del volumen `gitea_data`
2. Asegúrate de que USER_UID/USER_GID son correctos (1000 por defecto)
3. Revisa logs: `docker compose logs gitea`

### No puedo hacer push/pull via HTTPS

**Síntomas**: Error de autenticación o SSL

**Soluciones**:
1. Verifica que la URL base en Gitea es correcta (`https://gitea.tudominio.com`)
2. Comprueba que el certificado SSL es válido
3. Asegúrate de usar credenciales correctas
4. Para NPM: verifica que WebSocket Support está activado

### Problemas de rendimiento

**Síntomas**: Git lento, interfaz web lenta

**Soluciones**:
1. Verifica recursos del servidor (CPU, RAM, disco)
2. PostgreSQL puede necesitar tuning para muchos repos
3. Considera migrar a SSD si usas HDD
4. Revisa logs: `docker compose logs`

### Ver logs detallados

```bash
# Logs de Gitea
docker compose logs -f gitea

# Logs de PostgreSQL
docker compose logs -f gitea-db

# Logs dentro del contenedor
docker compose exec gitea cat /data/gitea/log/gitea.log
```

### Reiniciar completamente

```bash
# Cuidado: esto eliminará TODOS tus datos
docker compose down -v
docker compose up -d
```

## Recursos Adicionales

- [Documentación Oficial de Gitea](https://docs.gitea.com/)
- [GitHub de Gitea](https://github.com/go-gitea/gitea)
- [Wiki de este repositorio](https://git.ictiberia.com/groales/gitea/wiki)
- [Repositorio en Gitea](https://git.ictiberia.com/groales/gitea)

## Seguridad

- ⚠️ **Nunca** compartas tu DB_PASSWORD
- ⚠️ Usa contraseñas fuertes para usuarios administradores
- ⚠️ Deshabilita auto-registro si es una instalación privada
- ⚠️ Realiza backups regulares y pruébalos
- ⚠️ Mantén actualizado Gitea y PostgreSQL
- ⚠️ Usa HTTPS en producción (obligatorio)
- ⚠️ Configura SSH keys en lugar de contraseñas para git operations

## Licencia

Este repositorio de configuración está bajo licencia MIT. Gitea es software libre bajo licencia MIT.
