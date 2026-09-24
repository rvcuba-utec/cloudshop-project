# CloudShop — Guion de Sustentación para la Exposición

> Documento para preparar la exposición. Cada sección explica **qué es**, **por qué se eligió**
> (la justificación que debes defender) y **puntos clave** para decir en voz alta.
> Al final hay una batería de posibles preguntas del jurado con su respuesta.

---

## 0. Pitch de apertura (30 segundos)

> "CloudShop es un e-commerce con **arquitectura de microservicios** sobre AWS. Son 5
> microservicios en 3 lenguajes distintos, con 3 motores de base de datos, desplegados de
> forma **totalmente automatizada** con un único archivo de CloudFormation. El usuario final
> entra por un frontend React en Amplify, que consume el backend a través de un API Gateway y
> un balanceador de carga que reparte el tráfico entre dos servidores de aplicación."

**Idea fuerza:** *heterogeneidad tecnológica* (cada microservicio usa la mejor herramienta para
su problema) + *automatización total* (infraestructura como código).

---

## 1. Red — VPC

**Qué es:** una red privada virtual `10.0.0.0/16` con dos subredes públicas, un Internet
Gateway y una tabla de rutas.

**Por qué así:**
- **Sin NAT Gateway ni subred privada estricta.** Todas las VMs viven en subred pública porque
  todas necesitan salida a internet (`docker pull`, `boto3` a S3). El NAT Gateway cuesta dinero
  y en un entorno académico (AWS Academy) no aporta.
- **El aislamiento de datos lo hacen los Security Groups**, no la topología de red. Esto es una
  decisión consciente: más simple y suficiente para el alcance del proyecto.
- **Dos subredes en AZ distintas** porque el ALB de AWS **exige** mínimo dos zonas de
  disponibilidad.

**Puntos clave:**
- "Preferimos simplicidad controlada: sin NAT, pero con Security Groups estrictos."

---

## 2. Seguridad — Security Groups

**Qué es:** 4 firewalls virtuales, uno por rol (`alb`, `app`, `bd`, `ingesta`).

**Por qué así (la regla de oro):**
- Los puertos de base de datos (**3306 / 5432 / 27017**) **nunca** se exponen a internet.
  Solo aceptan tráfico desde el Security Group de app y el de ingesta.
- El ALB es el único que acepta el puerto 80 desde `0.0.0.0/0`.
- Los microservicios (8000-8003, 8080) aceptan tráfico del ALB; y también de internet en
  nuestro caso, para que **Swagger UI** pueda leer los `/openapi.json` desde el navegador.

**Puntos clave:**
- "La MV de datos no tiene ninguna puerta abierta a internet en sus puertos de BD; su única
  protección es el Security Group, y es suficiente porque las reglas referencian *grupos*, no IPs."

---

## 3. Cómputo — 4 Máquinas Virtuales EC2

**Qué es:** cuatro instancias Ubuntu 22.04.

| VM | Rol | Tipo |
|---|---|---|
| `cloudshop-mv-datos` | 3 bases de datos | **t3.small** (2 GB) |
| `cloudshop-mv-app-1` | 5 microservicios + swagger | t3.micro |
| `cloudshop-mv-app-2` | réplica de app-1 | t3.micro |
| `cloudshop-mv-ingesta` | 3 contenedores Python | t3.micro |

**Por qué así:**
- **Dos servidores de aplicación idénticos** → alta disponibilidad y balanceo. Si uno cae, el
  ALB sigue enrutando al otro.
- **La MV de datos es t3.small y no t3.micro** porque 1 GB de RAM no alcanza para correr MySQL +
  PostgreSQL + MongoDB simultáneamente (el kernel mataba MySQL por falta de memoria — *out of
  memory*). Este es un aprendizaje real del despliegue.
- **Separar datos de aplicación** permite escalar cada capa por separado y proteger las BD.

**Puntos clave:**
- "Las VMs de aplicación **no compilan código**: descargan imágenes ya construidas desde Docker
  Hub. Compilar en el servidor sería lento y frágil."

---

## 4. Contenedores — Docker + Docker Compose

**Qué es:** cada microservicio y cada base de datos corre en su propio contenedor.

**Por qué así:**
- **Portabilidad:** la misma imagen corre igual en la laptop, en app-1 y en app-2.
- **Flujo build → push → pull:** se construye una vez en la laptop, se sube a Docker Hub, y las
  VMs solo hacen `pull`. Reproducible y rápido.
- **`restart: unless-stopped`:** si un contenedor se cae (por ejemplo, la BD todavía no estaba
  lista), Docker lo reintenta solo.

**Puntos clave:**
- "3 lenguajes (Python, Go, Node) conviviendo sin conflicto gracias a Docker: cada uno lleva su
  runtime dentro de su imagen."

---

## 5. Bases de datos — Políglota (2 SQL + 1 NoSQL)

**Qué es y por qué cada una:**

| Motor | Microservicio | Por qué esta BD |
|---|---|---|
| **MySQL** | Catálogo (MS2) | Datos relacionales con transacciones de inventario (`SELECT ... FOR UPDATE` para reservar stock sin condiciones de carrera) |
| **PostgreSQL** | Usuarios (MS1) | Relacional robusto; buen soporte para integridad referencial usuarios↔direcciones |
| **MongoDB** | Ventas/Reseñas (MS3) | Documento flexible: una venta lleva sus `items[]` **embebidos**, se lee todo el pedido en una sola consulta |

**Puntos clave:**
- "No forzamos una sola base para todo: **persistencia políglota**. El pedido en Mongo guarda sus
  ítems embebidos, evitando *joins*; el catálogo en MySQL necesita transacciones ACID para el
  stock."
- "Cada microservicio es **dueño de su base de datos** — nadie más la toca directamente. Esto es
  el principio de *database per service*."

---

## 6. Microservicios — 5 servicios, 3 lenguajes

| # | Servicio | Lenguaje | Responsabilidad |
|---|---|---|---|
| MS1 | Usuarios | Python/FastAPI | Registro, login, JWT, direcciones, roles |
| MS2 | Catálogo | Go/Gin | Productos, categorías, inventario, reserva de stock |
| MS3 | Ventas/Reseñas | Node/Express | Pedidos y reseñas (MongoDB) |
| MS4 | Órdenes | Python/FastAPI | **Orquestador** sin BD propia |
| MS5 | Analítica | Python/FastAPI | Consultas a Athena (solo admin) |

**Por qué así:**
- **MS4 es un orquestador puro:** no tiene base de datos. Coordina el flujo de compra —
  reserva stock en MS2 → crea la venta en MS3 → confirma stock → marca confirmada. Si algo falla
  a mitad, **compensa** liberando lo reservado (patrón *Saga*).
- **MS2 en Go** porque el catálogo es el servicio de más tráfico (lecturas de productos) y Go da
  alto rendimiento con bajo consumo.
- **MS3 en Node/Express** (no NestJS) por decisión del equipo: mismo lenguaje, menos *boilerplate*.

**Roles (admin/usuario):**
- No hay un microservicio de autorización aparte. El rol viaja como un **claim `rol` dentro del
  mismo JWT** que emite MS1. Cada servicio lo verifica con la librería de su lenguaje **sin
  consultar la base de usuarios**. Esto es clave: los servicios son *stateless* respecto a la
  sesión.

**Puntos clave:**
- "El JWT es el contrato común entre servicios escritos en 3 lenguajes distintos: uno lo emite,
  los demás lo verifican con el mismo secreto compartido."

---

## 7. Balanceador — Application Load Balancer (ALB)

**Qué es:** un balanceador con un listener HTTP :80 y **enrutamiento por path** hacia 5 target
groups (uno por microservicio).

**Por qué así:**
- **Un solo punto de entrada** para todo el backend. El frontend usa una sola URL base.
- **Enrutamiento por path:** el ALB lee la ruta (`/api/catalogo/*`, `/usuarios*`, etc.) y la
  envía al microservicio correcto.
- **Las reglas se ordenan por prioridad:** las más específicas
  (`/usuarios/*/ventas` → MS3) van antes que las genéricas (`/usuarios*` → MS1), para que MS1 no
  capture rutas que en realidad son de MS3.
- **Health checks:** cada target group consulta `/health`; si un contenedor no responde, el ALB
  deja de enviarle tráfico.

**Puntos clave:**
- "El ALB reparte carga entre app-1 y app-2, y hace de *reverse proxy* inteligente según la URL."

---

## 8. API Gateway — la pieza que resuelve HTTPS

**Qué es:** un HTTP API que recibe HTTPS y reenvía todo (`ANY /{proxy+}`) al ALB por HTTP.

**Por qué existe (justificación importante):**
- Amplify sirve el frontend por **HTTPS**. El navegador **prohíbe** que una página HTTPS haga
  llamadas a un backend HTTP (error de *mixed content*).
- El ALB solo tiene HTTP (poner HTTPS requeriría un dominio propio + certificado ACM).
- El API Gateway **termina el HTTPS** con un certificado gestionado por AWS y reenvía por HTTP al
  ALB, que está dentro de la VPC. Así el frontend habla HTTPS y todo funciona.

**Comparación con la alternativa (que preguntará el jurado):**
- Otro equipo usó **5 API Gateways, uno por microservicio, sin ALB**. Funciona, pero el frontend
  tendría que conocer 5 URLs distintas.
- Nosotros: **1 API Gateway + 1 ALB**. El frontend usa una sola URL y el ALB hace el ruteo
  interno. Es más limpio y centraliza el enrutamiento en un solo lugar.

**Puntos clave:**
- "El API Gateway no reemplaza al ALB: lo complementa. API Gateway = capa HTTPS de borde; ALB =
  enrutamiento y balanceo interno."

---

## 9. Data Lake — S3 + Glue + Athena + MS5

**Qué es:** un pipeline analítico separado del transaccional.

**El flujo:**
1. **Ingesta** (3 contenedores Python) lee el 100% de cada BD con credenciales de **solo lectura**
   y sube archivos CSV/JSON a **S3** (un prefijo por tabla).
2. **Glue** (crawler) escanea S3 y registra el esquema de cada tabla en un catálogo de datos.
3. **Athena** ejecuta SQL directamente sobre los archivos en S3 (sin cargar a ninguna BD).
4. **MS5 Analítica** llama a Athena con `boto3` y expone 6 consultas de negocio (solo admin).

**Por qué así:**
- **Separar analítica de la operación:** las consultas pesadas (ticket promedio, productos más
  vendidos) corren sobre S3, **sin tocar las bases de producción**. No degradan la tienda.
- **Un prefijo por tabla en S3:** Glue define una tabla por carpeta; mezclar archivos con
  columnas distintas en una carpeta rompería el esquema.
- **Credenciales de solo lectura para la ingesta:** el proceso analítico no puede modificar nada.
- **Sin claves AWS en el código:** la ingesta usa el **IAM Role** de la instancia (`boto3` lo
  toma automáticamente).

**Puntos clave:**
- "Es un patrón de *data lake*: los datos crudos viven en S3 baratísimo, y Athena consulta bajo
  demanda pagando solo por lo que escanea. La operación y la analítica quedan desacopladas."

---

## 10. Frontend — React SPA en Amplify

**Qué es:** una *Single Page Application* en React (Vite) desplegada en AWS Amplify, conectada
al repositorio de GitHub.

**Por qué así:**
- **Amplify hace CI/CD:** cada `push` a `main` dispara build + deploy automático.
- **HTTPS gratis** y CDN global de AWS.
- **Regla de rewrite** para SPA: cualquier ruta desconocida devuelve `index.html` (para que el
  routing de React funcione), pero **sin capturar** los archivos `.js`/`.css` (eso se resolvió
  con una expresión regular específica).

**Puntos clave:**
- "El frontend consume el backend por una sola variable, `VITE_API_BASE_URL`, que apunta al API
  Gateway. Cambiar de entorno es cambiar una variable."

---

## 11. Infraestructura como Código — CloudFormation

**Qué es:** un único archivo `cloudshop-infra.yaml` que describe **toda** la infraestructura.

**Por qué así (el gran diferenciador):**
- **Reproducibilidad:** borrar el stack y recrearlo levanta exactamente lo mismo.
- **Automatización total:** el `UserData` de cada VM instala Docker, clona el repo, escribe el
  `.env`, levanta los contenedores y —en la de datos— carga los datos. Cero clics manuales.
- **Orden de dependencias:** CloudFormation crea los recursos en el orden correcto
  (el ALB antes que Amplify, porque Amplify necesita el DNS del API Gateway; las VMs de app
  después de Amplify, porque el `.env` de CORS necesita el dominio de Amplify).

**Puntos clave:**
- "Todo el despliegue cabe en un archivo. El operador solo publica las imágenes, sube el YAML y
  ejecuta el crawler. El resto es automático."

---

## 12. Diagrama para la diapositiva

```text
  Usuario ─HTTPS─► Amplify (React) ─HTTPS─► API Gateway ─HTTP─► ALB ─┬─► app-1 (5 MS)
                                                                     └─► app-2 (5 MS)
                                                                            │
                                                                            ▼
                                                                      MV Datos
                                                                   MySQL/PG/Mongo
                                                                            ▲
                                                            Ingesta ────────┘
                                                               │
                                                               ▼
                                                     S3 ─► Glue ─► Athena ─► MS5
```

---

## 13. Posibles preguntas del jurado (y respuestas)

**P: ¿Por qué no usaron Kubernetes?**
R: Para el alcance del proyecto, Docker Compose en dos VMs detrás de un ALB da alta
disponibilidad suficiente sin la complejidad operativa de K8s. La arquitectura permite migrar a
ECS/EKS después sin cambiar el código.

**P: ¿Qué pasa si se cae una VM de aplicación?**
R: El ALB detecta el fallo con el *health check* y enruta todo al otro servidor. El servicio
sigue disponible.

**P: ¿Por qué API Gateway *y* ALB, no uno solo?**
R: El API Gateway resuelve el HTTPS de borde (mixed content con Amplify); el ALB hace el
enrutamiento por path y el balanceo entre las dos VMs. Cada uno cumple un rol distinto.

**P: ¿Cómo aíslan la base de datos si está en subred pública?**
R: Con Security Groups. Los puertos de BD solo aceptan tráfico de los grupos de app e ingesta,
nunca de internet. El aislamiento es a nivel de firewall, no de topología.

**P: ¿Cómo comparten sesión 5 servicios en 3 lenguajes?**
R: Con un JWT HS256. MS1 lo emite; los demás lo verifican con el mismo secreto usando la librería
de cada lenguaje. No hay estado compartido ni consultas cruzadas a la BD de usuarios.

**P: ¿La analítica afecta el rendimiento de la tienda?**
R: No. La analítica corre sobre S3 con Athena, completamente separada de las bases de producción.
La ingesta lee con credenciales de solo lectura.

**P: ¿Por qué MongoDB para ventas y no SQL?**
R: Un pedido es un documento natural: lleva sus ítems embebidos. Se lee/escribe el pedido completo
en una operación, sin *joins*. El modelo documento encaja mejor que el relacional aquí.

**P: ¿Cómo garantizan consistencia en una compra (stock)?**
R: MS4 orquesta con un patrón Saga: reserva stock (MS2, con `SELECT FOR UPDATE`), crea la venta
(MS3), confirma. Si algo falla, compensa liberando el stock reservado antes de responder error.

**P: ¿Dónde guardan los secretos?**
R: En este proyecto están hardcodeados en el YAML por simplicidad académica. En producción irían
en AWS Secrets Manager o SSM Parameter Store. El token de GitHub nunca se versiona.

---

## 14. Cierre (30 segundos)

> "En resumen: CloudShop demuestra una arquitectura de microservicios real —heterogénea,
> desacoplada y con persistencia políglota— desplegada de forma completamente automatizada con
> infraestructura como código. Separamos la capa de presentación (Amplify), la de borde
> (API Gateway), el balanceo (ALB), la lógica (5 microservicios), los datos (3 motores) y la
> analítica (data lake), cada una con la tecnología adecuada a su problema."
