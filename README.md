# Plataforma de Microservicios para Comercio Electronico (Products Launcher)

## 1. Descripcion del Proyecto

Este repositorio (`03-products-launcher`) funciona como el orquestador principal y lanzador central de una arquitectura de microservicios para un sistema de comercio electronico construido con NestJS.

El proyecto consolida multiples microservicios desacoplados a traves de **Git Submodules**, facilitando tanto el desarrollo independiente de cada componente como la orquestacion conjunta del ecosistema mediante **Docker Compose** y manifiestos de **Kubernetes (Helm)**.

La comunicacion interservicios es orientada a eventos y asincrona mediante un broker de mensajeria **NATS**, mientras que las aplicaciones cliente o frontend consumen los servicios a traves de un **API Gateway** centralizado que expone endpoints REST.

---

## 2. Arquitectura y Componentes

El ecosistema esta integrado por los siguientes servicios y componentes:

- **NATS Server (`nats-server`)**:
  Servidor de mensajeria de alto rendimiento. Conecta todos los microservicios actuando como canal de transporte para comunicacion Request-Response y Event-Driven. Expone internamente el puerto 4222 y en desarrollo el puerto 8222 para inspeccion y monitoreo.

- **Client Gateway (`client-gateway`)**:
  API Gateway REST que sirve como punto unico de entrada para las aplicaciones cliente. Sus responsabilidades incluyen:
  - Enrutamiento de peticiones hacia los microservicios adecuados a traves del broker NATS.
  - Validacion de datos de entrada mediante DTOs (Data Transfer Objects).
  - Manejo de autenticacion y guardias de seguridad (JWT) para rutas protegidas.
  - Transformacion y gestion de respuestas y excepciones RPC.

- **Microservicio de Autenticacion (`auth-ms`)**:
  Gestiona la identidad de usuarios, registro, inicio de sesion y generacion/verificacion de tokens JWT.
  - Persistencia: Base de datos MongoDB mediante Prisma ORM.
  - Transporte: NATS.

- **Microservicio de Productos (`products-ms`)**:
  Administra el catalogo de productos (creacion, consulta, actualizacion, eliminacion y disponibilidad de inventario).
  - Persistencia: Base de datos SQLite (`dev.db`) mediante Prisma ORM.
  - Transporte: NATS.

- **Microservicio de Ordenes (`orders-ms`)**:
  Gestiona el ciclo de vida de los pedidos (estados: PENDING, PAID, DELIVERED, CANCELLED).
  - Valida productos y existencias comunicandose con `products-ms`.
  - Solicita sesiones de cobro a `payments-ms`.
  - Escucha el evento `payment.succeeded` para actualizar el estado del pedido a PAID y almacenar el comprobante.
  - Persistencia: Base de datos PostgreSQL mediante Prisma ORM (ejecuta migraciones automaticamente al iniciar).
  - Transporte: NATS.

- **Microservicio de Pagos (`payments-ms`)**:
  Procesa operaciones con la pasarela de pagos Stripe.
  - Genera sesiones de Stripe Checkout para pedidos entrantes.
  - Expone un endpoint HTTP para recibir webhooks de eventos de Stripe (`/payments/webhook`).
  - Emite eventos hacia NATS cuando un cobro ha sido exitoso para que otros servicios actuen en consecuencia.
  - Transporte: HTTP (Webhooks y redirecciones) y NATS (comunicacion interna).

- **Base de Datos de Ordenes (`orders-db`)**:
  Contenedor de PostgreSQL 16.2 provisto para el entorno de desarrollo local, mapeado al puerto 5433 en el host.

- **Despliegue en Kubernetes (`k8s/tienda`)**:
  Chart de Helm estructurado para empaquetar y desplegar los microservicios dentro de un cluster de Kubernetes.

---

## 3. Flujo de Trabajo

1. El usuario interactua con el API REST a traves de `client-gateway`.
2. Para operaciones autenticadas, el gateway valida el token JWT contra `auth-ms`.
3. Al crear una orden, el gateway emite un mensaje a `orders-ms`.
4. `orders-ms` consulta a `products-ms` para validar existencias y precios, crea el registro en PostgreSQL y solicita a `payments-ms` una sesion de Stripe.
5. El usuario realiza el pago a traves de Stripe Checkout.
6. Stripe notifica a `payments-ms` a traves del webhook HTTP.
7. `payments-ms` emite el evento `payment.succeeded` en NATS.
8. `orders-ms` intercepta el evento, marca la orden como pagada y asocia el recibo correspondiente.

---

## 4. Requisitos Previos

Antes de ejecutar el proyecto, asegurese de tener instalado:

- Git (version 2.20 o superior para soporte completo de submodulos)
- Docker Desktop o Docker Engine con Docker Compose (v2)
- Node.js (version 20 o superior, requerido si desea ejecutar comandos de Prisma o NestJS localmente)
- Una cuenta en Stripe con claves de prueba (API Secret y Webhook Secret)
- Una cadena de conexion a MongoDB (por ejemplo, mediante una instancia en MongoDB Atlas)

---

## 5. Pasos para Levantar el Proyecto

### Paso 1: Clonar el Repositorio con Submodulos

Dado que los microservicios residen en repositorios individuales vinculados como submodulos, es indispensable clonar tanto el repositorio raiz como el contenido de los submodulos.

Opcion 1: Clonar y descargar submodulos en un solo comando:
```bash
git clone --recurse-submodules <URL_DEL_REPOSITORIO>
cd 03-products-launcher
```

Opcion 2: Si ya clono el repositorio de forma simple:
```bash
git submodule update --init --recursive
```

### Paso 2: Configurar las Variables de Entorno

Genere el archivo de entorno `.env` en la raiz del proyecto copiando la plantilla `.env.template`:

En sistemas Unix / Linux / macOS:
```bash
cp .env.template .env
```

En Windows (PowerShell):
```powershell
Copy-Item .env.template .env
```

Edite el archivo `.env` configurando los valores correspondientes:

| Variable | Descripcion | Valor sugerido / Ejemplo |
|---|---|---|
| `CLIENT_GATEWAY_PORT` | Puerto de escucha publico del API Gateway | `3000` |
| `PAYMENT_MS_PORT` | Puerto de escucha publico del servicio de pagos (para webhooks de Stripe) | `3003` |
| `STRIPE_SECRET` | Clave secreta de la API de Stripe | `sk_test_...` |
| `STRIPE_CANCEL_URL` | URL de redireccion si el pago en Stripe es cancelado | `http://localhost:3003/payments/cancel` |
| `STRIPE_SUCCESS_URL` | URL de redireccion cuando el pago en Stripe es exitoso | `http://localhost:3003/payments/success` |
| `STRIPE_ENDPOINT_SECRET` | Clave secreta para validar la firma de los webhooks de Stripe | `whsec_...` |
| `AUTH_DATABASE_URL` | Cadena de conexion a MongoDB para el microservicio `auth-ms` | `mongodb+srv://<usuario>:<password>@<cluster>.mongodb.net/<dbname>` |
| `JWT_SECRET` | Clave secreta utilizada para firmar y validar tokens JWT | `UnaClaveSecretaRobusta` |
| `ORDERS_DATABASE_URL` | Cadena de conexion PostgreSQL externa (utilizada en produccion o base remota) | `postgresql://<user>:<pass>@<host>:<port>/<db>` |

Nota: En el entorno de desarrollo local con `docker-compose.yml`, el microservicio de ordenes se conecta automaticamente al contenedor PostgreSQL local (`orders-db`), por lo que no es estrictamente necesario configurar `ORDERS_DATABASE_URL` para desarrollo local a menos que desee utilizar una base de datos externa como Neon o RDS.

### Paso 3: Iniciar los Servicios en Modo Desarrollo

Para compilar las imagenes y levantar todos los microservicios en entorno de desarrollo con recarga en caliente (hot reload):

```bash
docker compose up --build
```

Si prefiere ejecutar los contenedores en segundo plano:
```bash
docker compose up -d --build
```

Durante el arranque:
- Se inicializara el broker NATS en los puertos 4222 y 8222.
- Se levantara el contenedor de base de datos PostgreSQL (`orders_database`) en el puerto 5433.
- Se compilaran y ejecutaran en modo watch los contenedores de `client-gateway`, `products-ms`, `orders-ms`, `payments-ms` y `auth-ms`.
- En `orders-ms`, se ejecutaran automaticamente las migraciones pendientes de Prisma (`npx prisma migrate deploy`).

### Paso 4: Comprobacion de los Servicios

Una vez inicializados los contenedores, puede verificar el funcionamiento accediendo a:

- **Client Gateway (API REST)**: `http://localhost:3000`
- **Dashboard y Monitoreo de NATS**: `http://localhost:8222`
- **Endpoints de Pagos y Webhooks**: `http://localhost:3003`
- **Base de Datos PostgreSQL (Orders)**: `localhost:5433` (Credenciales por defecto: usuario `postgres`, clave `123456`, base `ordersdb`)

### Paso 5: Iniciar los Servicios en Modo Produccion (Opcional)

Si desea ejecutar el sistema utilizando las imagenes optimizadas multi-stage (`dockerfile.prod`):

```bash
docker compose -f docker-compose.prod.yml up --build
```

Para detener y retirar todos los contenedores:
```bash
docker compose down
```

Si desea eliminar tambien los volumenes asociados:
```bash
docker compose down -v
```

---

## 6. Administracion de Git Submodules

Dado que cada microservicio corresponde a un repositorio independiente, tenga en cuenta las siguientes pautas:

### Actualizar las referencias de los submodulos
Para sincronizar las carpetas locales con los ultimos commits de las ramas remotas de cada microservicio:
```bash
git submodule update --remote
```

### Flujo de confirmacion de cambios (Commits)
Si realiza modificaciones en el codigo de cualquiera de los microservicios:
1. Acceda al directorio del submodulo y confirme sus cambios primero:
   ```bash
   cd <nombre-submodulo>
   git add .
   git commit -m "Descripcion del cambio"
   git push
   ```
2. Vuelva a la raiz del repositorio lanzador y confirme la actualizacion del puntero del submodulo:
   ```bash
   cd ..
   git add <nombre-submodulo>
   git commit -m "Actualiza referencia de <nombre-submodulo>"
   git push
   ```

Si realiza este proceso en orden inverso, el repositorio principal apuntara a referencias no existentes en el remoto y se generaran conflictos de sincronizacion.

### Agregar un nuevo submodulo al proyecto
Para incorporar un nuevo microservicio al lanzador:
```bash
git submodule add <url_del_repositorio> <nombre_carpeta>
git add .
git commit -m "Agrega nuevo submodulo"
git push
```

---

## 7. Despliegue en Kubernetes

El proyecto incluye en el directorio `k8s/tienda/` la configuracion de Helm para el despliegue de las cargas de trabajo en Kubernetes:

- `k8s/tienda/Chart.yaml`: Definicion y metadatos del chart.
- `k8s/tienda/values.yaml`: Configuracion parametrizable de réplicas, imagenes, puertos y variables de entorno para cada microservicio.
- `k8s/tienda/templates/`: Plantillas de Deployments, Services e Ingress.