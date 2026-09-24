# CloudShop — Registro del Despliegue Realizado

> Bitácora completa del despliegue automatizado de CloudShop en AWS mediante un único
> archivo CloudFormation (`cloudshop-infra.yaml`) con todos los valores *hardcodeados*.
> Incluye qué se construyó, los cambios hechos al repositorio, y todos los problemas
> encontrados con su solución.

---

## 1. Objetivo

Desplegar **toda** la infraestructura de CloudShop con la mínima intervención manual:
un solo `stack` de CloudFormation que crea la red, las 4 máquinas virtuales, el balanceador,
el data lake, el API Gateway y el frontend, dejando los contenedores corriendo y los datos
cargados automáticamente.

El único trabajo manual del operador es: publicar las imágenes Docker una vez, subir el YAML,
y ejecutar el crawler de Glue al final.

---

## 2. Arquitectura desplegada

```text
Usuario final
     │ HTTPS
     ▼
AWS Amplify (React SPA)
     │ HTTPS  (VITE_API_BASE_URL)
     ▼
API Gateway HTTP  ──►  termina HTTPS, reenvía por HTTP
     │ HTTP
     ▼
ALB :80  (path-based routing)
     ├── /usuarios*        → MS1 Usuarios   :8000
     ├── /api/catalogo/*   → MS2 Catálogo   :8080
     ├── /ventas*          → MS3 Ventas     :8002
     ├── /usuarios/*/ventas→ MS3 Ventas     :8002
     ├── /productos/*/resenas → MS3 Ventas  :8002
     ├── /ordenes*         → MS4 Órdenes    :8003
     └── /analitica*       → MS5 Analítica  :8001
     │
     ▼  (2 réplicas: app-1 10.0.1.11 y app-2 10.0.1.12)
MV App (Docker Compose, 5 microservicios + swagger-ui :8081)
     │ IP privada
     ▼
MV Datos 10.0.1.10 (t3.small)
     ├── MySQL 8.4     :3306  (cloudshop_catalogo)
     ├── PostgreSQL 16 :5432  (cloudshop_usuarios)
     └── MongoDB 7     :27017 (cloudshop_ventas)
     ▲
     │ pull solo lectura
MV Ingesta 10.0.1.20 (3 contenedores Python → boto3)
     │ put_object
     ▼
S3 data lake ──► Glue (crawler) ──► Athena ──► MS5 Analítica
```

### Recursos creados por el YAML

| Categoría | Recursos |
|---|---|
| Red | VPC `10.0.0.0/16`, 2 subredes públicas (`10.0.1.0/24`, `10.0.2.0/24`), IGW, route table |
| Seguridad | 4 Security Groups: `sg-alb`, `sg-app`, `sg-bd`, `sg-ingesta` |
| Cómputo | 4 EC2: `cloudshop-mv-datos` (t3.small), `app-1`, `app-2`, `ingesta` (t3.micro) |
| Balanceo | ALB + 5 Target Groups + Listener :80 + 7 reglas de path |
| API | API Gateway HTTP + Integration + Route `ANY /{proxy+}` + Stage `$default` |
| Datos | Bucket S3 `cloudshop-data-lake-2026-utec-mr-cs2032-v2`, Glue DB `cloudshop_analytics`, Glue Crawler |
| Frontend | Amplify App + Branch `main` |

### Valores hardcodeados

- **IPs privadas fijas:** datos `10.0.1.10`, app-1 `10.0.1.11`, app-2 `10.0.1.12`, ingesta `10.0.1.20`
- **Docker Hub:** usuario `rcuba`
- **Credenciales de BD:** `cloud_user` / `CloudShop2026!`, root MySQL `CloudShopRoot2026!`
- **Usuario solo-lectura ingesta:** `ingesta_my` / `ingesta_pg` con `IngestaReadOnly2026!`
- **JWT secret compartido:** `cloudshop-jwt-secret-2026-cs2032-utec-production`
- **Admin bootstrap:** `ADMIN_EMAILS=admin@cloudshop.pe`
- **IAM:** `LabRole` / `LabInstanceProfile` (AWS Academy)

---

## 3. Microservicios

| # | Servicio | Lenguaje | Base de datos | Puerto |
|---|---|---|---|---|
| MS1 | Usuarios y Direcciones | Python / FastAPI | PostgreSQL | 8000 |
| MS2 | Catálogo e Inventario | Go / Gin | MySQL | 8080 |
| MS3 | Ventas y Reseñas | Node.js / Express | MongoDB | 8002 |
| MS4 | Órdenes (orquestador) | Python / FastAPI | — (sin BD) | 8003 |
| MS5 | Analítica | Python / FastAPI | Athena | 8001 |

Todos verifican el **mismo JWT HS256** que emite MS1 (claim `rol`), cada uno con la librería
de su lenguaje, sin consultar la base de usuarios.

---

## 4. Cambios hechos al repositorio

| # | Archivo | Cambio | Motivo |
|---|---|---|---|
| 1 | `cloudshop-infra.yaml` | **Nuevo** — plantilla CloudFormation completa | Automatizar todo el despliegue |
| 2 | `backend/docker-compose.yml` | Agregado servicio `swagger-ui` (:8081) | Catálogo centralizado de APIs |
| 3 | `backend/products/Dockerfile` | `COPY *.go .` → `COPY *.go ./` | El builder clásico exige `/` con varios archivos |
| 4 | `backend/.env` | `DOCKERHUB_USER=rcuba` | Estaba en `cloudshopuser` (no versionado) |
| 5 | `cloudshop-infra.yaml` | Agregado API Gateway HTTP | Amplify HTTPS no puede llamar al ALB HTTP (mixed content) |
| 6 | Todo el repo | URL `CloudComputing-Project` → `cloudshop-project` | Cambio de repositorio |
| 7 | `cloudshop-infra.yaml` | Guiones largos `—` → `-` en `GroupDescription` | AWS rechaza caracteres no-ASCII |
| 8 | `cloudshop-infra.yaml` | MV datos `t3.micro` → `t3.small` | 1 GB de RAM mataba MySQL (OOM) |
| 9 | Frontend (`authService.js`, `ventasService.js`, `Admin.jsx`) | Paginación en Usuarios y Órdenes | Solo mostraba los primeros 100 |

---

## 5. Pasos ejecutados

### 5.1 Publicar imágenes Docker (desde la laptop)

```bash
# Backend — 5 imágenes
cd backend
docker compose build
docker login -u rcuba
docker compose push
# → rcuba/cloudshop-{catalogo,usuarios,ventas-resenas,ordenes,analitica}:latest

# Ingesta — 3 imágenes
cd ../Ingesta
echo "DOCKERHUB_USER=rcuba" > .env
cp ingesta-usuarios/.env.example ingesta-usuarios/.env
cp ingesta-catalogo/.env.example ingesta-catalogo/.env
cp ingesta-ventas/.env.example  ingesta-ventas/.env
docker compose build
docker compose push
# → rcuba/cloudshop-ingesta-{usuarios,catalogo,ventas}:latest
```

> `swagger-ui` es imagen pública (`swaggerapi/swagger-ui`), no se construye ni se publica.

### 5.2 Desplegar el stack

1. AWS Console → **CloudFormation → Create stack → Upload template** → `cloudshop-infra.yaml`
2. Nombre: `cloudshop` → Next → Next → Submit
3. Esperar ~30 min hasta `CREATE_COMPLETE`

### 5.3 Ejecutar el crawler de Glue

AWS Console → **Glue → Crawlers → cloudshop-crawler → Run crawler**
(solo después de que la ingesta haya subido los archivos a S3)

### 5.4 URLs resultantes (pestaña Outputs del stack)

- **Frontend:** `https://main.d2jm0gou6s49cg.amplifyapp.com`
- **API Gateway:** `https://xtfv16om90.execute-api.us-east-1.amazonaws.com`
- **ALB:** interno (referencia)

---

## 6. Problemas encontrados y soluciones

### 6.1 Amplify falla con `Bad credentials` (401)
**Causa:** el token de GitHub estaba *hardcodeado* en el YAML, se subió al repo y **GitHub lo
revocó automáticamente** por seguridad.
**Solución:** generar un token nuevo, ponerlo solo en el YAML local, agregar el YAML al
`.gitignore` (luego se reemplazó por un placeholder `TU_GITHUB_TOKEN_AQUI` para poder versionarlo).

### 6.2 `S3 bucket already exists`
**Causa:** el bucket quedó del intento anterior (`DeletionPolicy: Retain`).
**Solución:** eliminarlo manualmente en S3 antes de recrear el stack.

### 6.3 `GroupDescription ... Character sets beyond ASCII are not supported`
**Causa:** las descripciones de los Security Groups tenían guión largo `—` (Unicode).
**Solución:** reemplazar por guión normal `-`.

### 6.4 MySQL muere apenas arranca (`Out of memory: Killed process mysqld`)
**Causa:** el `t3.micro` (1 GB RAM) no soporta MySQL + PostgreSQL + MongoDB juntos.
**Solución:** cambiar la MV de datos a `t3.small` (2 GB) y hacer *stack update*.

### 6.5 `Host '10.0.1.11' is not allowed to connect` / `Access denied for root`
**Causa:** el usuario `cloud_user` solo existía para `localhost`; la contraseña de root no
coincidía tras el reinicio de la instancia.
**Solución:** resetear con `skip-grant-tables` y crear `cloud_user`@`%`:

```bash
docker exec cloudshop-mysql bash -c 'printf "[mysqld]\nskip-grant-tables\n" > /etc/mysql/conf.d/reset.cnf'
docker restart cloudshop-mysql
# ALTER USER root; CREATE USER cloud_user@'%'; GRANT ALL ON cloudshop_catalogo.*
# quitar reset.cnf y reiniciar
```

### 6.6 `Unknown database 'cloudshop_catalogo'`
**Causa:** la inicialización de MySQL falló en el primer arranque (por el OOM), así que la BD
nunca se creó.
**Solución:** crearla y cargar el esquema manualmente:

```bash
docker exec cloudshop-mysql mysql -uroot -pCloudShopRoot2026! -e "CREATE DATABASE cloudshop_catalogo ..."
docker cp init.sql cloudshop-mysql:/tmp/init.sql
docker exec cloudshop-mysql bash -c 'mysql -uroot -p... cloudshop_catalogo < /tmp/init.sql'
```

### 6.7 Datos no cargados
**Solución:** ejecutar el loader en la MV de datos:

```bash
pip3 install pymysql "psycopg[binary]" pymongo python-dotenv
cd ~/cloudshop
MYSQL_HOST=localhost MYSQL_USER=cloud_user MYSQL_PASSWORD='CloudShop2026!' \
MYSQL_DATABASE=cloudshop_catalogo \
DATABASE_URL='postgresql+psycopg2://cloud_user:CloudShop2026!@localhost:5432/cloudshop_usuarios' \
MONGO_URI='mongodb://localhost:27017/cloudshop_ventas' \
python3 Data/scripts/src/scripts/load_csv_bd.py
```

> El loader es idempotente (`DELETE` antes de insertar): correrlo dos veces da el mismo resultado.

### 6.8 Frontend: `Loading module ... blocked because of a disallowed MIME type (text/html)`
**Causa 1:** el `cd Frontend/frontend` del `preBuild` persistía al `build`, que volvía a hacer
`cd` y fallaba.
**Solución 1:** dejar el `cd` solo en `preBuild`; en `build` solo `npm run build`.

**Causa 2:** la regla de rewrite `/<*>` → `/index.html` (200) capturaba también los `.js`/`.css`.
**Solución 2:** regex que reescribe solo rutas sin extensión:

```json
[{ "source": "</^[^.]+$|\\.(?!(css|gif|ico|jpg|js|png|txt|svg|woff|ttf|map|json)$)([^.]+$)/>",
   "status": "200", "target": "/index.html" }]
```

### 6.9 `404` en `/usuarios`, `/ordenes`, `/ventas`
**Causa:** las reglas del ALB usaban `/usuarios/*`, que **no** coincide con `/usuarios?page=1`
(sin sub-ruta).
**Solución:** cambiar los patrones a `/usuarios*`, `/ordenes*`, `/ventas*`, `/analitica*`.
(`/api/catalogo/*` se dejó igual porque siempre lleva sub-ruta.)

### 6.10 `500` en `/usuarios` (CORS header missing)
**Causa:** algunos emails del CSV tenían tildes/espacios
(`josé manuel.toledo62@...`) que el validador de FastAPI rechazaba. El 500 no lleva headers CORS,
por eso el navegador también reportaba CORS.
**Solución:** normalizar los emails inválidos en PostgreSQL:

```sql
UPDATE usuarios SET email = 'user' || id || '@cloudshop.pe'
WHERE email ~ '[^\x00-\x7F]' OR email ~ ' ';
```

### 6.11 Paginación solo mostraba 100 registros
**Causa:** `ventasService.todas()` devolvía un array (`res.data`) en vez de `{data, total}`,
así que la UI no sabía cuántas páginas había.
**Solución:** devolver el objeto completo `{data, total}` y agregar controles de página en
`AdminUsuarios` y `AdminOrdenes`.

---

## 7. Estado final

- ✅ 5 microservicios corriendo y `healthy` en app-1 (y app-2 réplica)
- ✅ 3 bases de datos con datos cargados en la MV de datos
- ✅ ALB enrutando por path + API Gateway terminando HTTPS
- ✅ Frontend en Amplify consumiendo el API Gateway
- ✅ Ingesta → S3 → Glue → Athena para MS5
- ✅ Swagger UI en `:8081` (MS1, MS4, MS5 vía OpenAPI de FastAPI)

### Credenciales de acceso (demo)

- **Admin:** `admin@cloudshop.pe` (registrarse en la app; nace con rol admin por `ADMIN_EMAILS`)

---

## 8. Notas de seguridad

- El `cloudshop-infra.yaml` contiene credenciales de Docker Hub y BD *hardcodeadas*; el token
  de GitHub se reemplazó por un placeholder antes de versionar. **Nunca** subir el token real.
- Los puertos de BD (3306/5432/27017) solo se abren desde `sg-app` y `sg-ingesta`, nunca a
  `0.0.0.0/0`.
- Regla temporal de SSH directo a la MV de datos: eliminar de `sg-bd` tras terminar el debug.
