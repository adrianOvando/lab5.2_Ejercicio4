# Laboratorio 5.2: Configuracion de un Pipeline de Despliegue Continuo con Docker

**Universidad San Francisco Xavier de Chuquisaca**  
**Facultad de Tecnologia**  
**Carrera:** Ingenieria en Ciencias de la Computacion  
**Asignatura:** Trabajando en la Nube (COM610)  
**Docente:** Ing. Marcelo Quispe Ortega  
**Estudiante:** Jesus Adrian Ovando  
**Semestre:** 2/2026  
**Repositorio Ejercicios Guiados (3.1 - 3.4):** https://github.com/adrianOvando/Lab5.2.git  
**Repositorio Practica Individual (4):** https://github.com/adrianOvando/lab5.2_Ejercicio4.git  
**Direccion IP Elastica EC2 (AWS):** http://18.213.236.16  

---

## 1. Objetivos del Laboratorio

Al finalizar el desarrollo de la presente practica de laboratorio, se alcanzaron y consolidaron los siguientes objetivos tecnicos y conceptuales:

- Comprender el ciclo integral de Integracion y Despliegue Continuo (CI/CD) orquestado mediante contenedores Docker.
- Disenar y construir imagenes de Docker optimizadas aplicando el patron de construccion de multiples etapas (*multi-stage build*) con base en distribuciones ultralivianas (Alpine Linux).
- Automatizar la compilacion, etiquetado y publicacion de imagenes de contenedores hacia el registro publico Docker Hub mediante flujos de trabajo en GitHub Actions.
- Gestionar credenciales y secretos de forma segura en GitHub Actions, protegiendo llaves privadas SSH, tokens de acceso personal y direcciones de infraestructura de red.
- Implementar despliegue automatizado hacia un servidor virtual en la nube (instancia AWS EC2 con Ubuntu Server) ejecutando comandos remotos sobre SSH.
- Aplicar tecnicas de validacion en caliente mediante sondas de salud (*health checks*) en puertos alternos antes de la promocion a produccion en el puerto 80 (tecnica de despliegue con tiempo de inactividad cero o *near-zero downtime*).
- Establecer politicas de versionamiento dual mediante etiquetas mutables (`latest`) y etiquetas inmutables referenciadas al identificador unico del commit (`github.sha`).
- Ejecutar procedimientos controlados de retroceso (*rollback*) ante incidentes en produccion aprovechando el historial inmutable de imagenes.
- Incorporar analisis estatico de seguridad y escaneo de vulnerabilidades en imagenes de contenedores mediante la integracion de herramientas especializadas (Trivy).

---

## 2. Requisitos y Entorno de Trabajo

Para el desarrollo satisfactorio de la practica se dispuso de las siguientes tecnologias, herramientas y recursos de infraestructura:

- **Estacion de Trabajo Local:** Sistema Operativo Linux Ubuntu x86_64, shell interactiva Bash, Git version 2.43+.
- **Motor de Contenedores:** Docker Community Edition version 29.1+, Docker Buildx y cliente Docker CLI local.
- **Registro de Imagenes:** Docker Hub bajo la cuenta de usuario `ciscoad`, autenticada mediante Personal Access Token (PAT) con permisos de lectura y escritura (*Read & Write*).
- **Infraestructura de Nube:** Instancia de Amazon Web Services (AWS) EC2 tipo `t2.micro` con sistema operativo Ubuntu Server 24.04 LTS.
- **Direccionamiento Fijo de Red:** Asignacion de una IP Elastica de AWS (`18.213.236.16`) asociada a la interfaz de red de la instancia EC2 para impedir la modificacion de la IP publica tras detenciones o reinicios.
- **Seguridad Perimetral (AWS Security Groups):** Reglas de entrada (*Inbound Rules*) configuradas para permitir trafico en el puerto 22 (SSH) y en el puerto 80 (HTTP) abierto al publico general (`0.0.0.0/0`).
- **Autenticacion SSH:** Par de llaves criptograficas RSA con archivo `.pem` de permisos restringidos (`chmod 400`) para acceso exclusivo del usuario `ubuntu`.
- **Servidor de Aplicacion:** Node.js version 20 LTS, Express version 4/5, suites de pruebas Jest y Supertest.

---

## 3. Desarrollo de los Ejercicios Guiados

---

### Ejercicio 3.1: Creacion del Dockerfile y prueba local

En esta primera etapa se preparo el repositorio para soportar la contenerizacion de la aplicacion backend Node.js mediante un archivo `Dockerfile` con arquitectura multi-etapa (*multi-stage build*), acompanado de su correspondiente archivo de exclusion `.dockerignore`.

#### 1. Diseno del Dockerfile multi-stage
El patron multi-stage permite separar el entorno de compilacion del entorno final de ejecucion. En la primera etapa (`builder`) se descargan e instalan las dependencias de produccion (`npm ci --only=production`), mientras que en la segunda etapa (`runtime`) se parte de una imagen limpia de `node:20-alpine`, copiando unicamente la carpeta `node_modules` y los archivos indispensables de la aplicacion (`app.js`, `server.js`, `package.json`), descartando herramientas de compilacion o caches temporales.

Adicionalmente, se configuro la exposicion del puerto `3000` y la ejecucion del proceso a traves de `node server.js`.

![Creacion del Dockerfile y estructura del proyecto](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-11-40.png)

#### 2. Configuracion de exclusion con .dockerignore
Para evitar el envio de archivos innecesarios al contexto del demonio de Docker y optimizar los tiempos de transferencia, se definio el archivo `.dockerignore`:

![Configuracion del archivo .dockerignore](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-11-52.png)

#### 3. Construccion de la imagen localmente
Se ejecuto el comando de construccion de la imagen etiquetada localmente como `mi-app:local`:

```bash
docker build -t mi-app:local .
```

El proceso concluyo exitosamente, resolviendo las capas intermedias y produciendo el artefacto final optimizado.

![Compilacion exitosa de la imagen Docker](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-13-16.png)

#### 4. Ejecucion y prueba del contenedor local
Se instancio un contenedor en segundo plano enlazando el puerto local `3000` con el puerto del contenedor:

```bash
docker run -d -p 3000:3000 --name app-local mi-app:local
```

Posteriormente se verifico la salud de la aplicacion mediante una peticion HTTP al endpoint `/health`:

```bash
curl http://localhost:3000/health
```

La aplicacion respondio satisfactoriamente con el estado esperado: `{"status":"ok"}`.

![Ejecucion del contenedor y validacion con curl](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-15-36.png)

#### 5. Limpieza de recursos locales
Finalizada la prueba de concepto local, se detuvo y elimino el contenedor de prueba:

```bash
docker stop app-local && docker rm app-local
```

![Detencion y eliminacion del contenedor local](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-16-27.png)

#### 6. Versionamiento y envio al repositorio remoto
Se anadieron los archivos `Dockerfile` y `.dockerignore` al control de versiones de Git y se enviaron a la rama `main` del repositorio remoto en GitHub:

```bash
git add Dockerfile .dockerignore
git commit -m "feat: agrega Dockerfile multi-stage para contenerizacion"
git push origin main
```

![Confirmacion de cambios en Git](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-17-10.png)

![Envio de los cambios a GitHub origin main](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-17-45.png)

---

### Ejercicio 3.2: Configuracion de secretos en GitHub

Para que GitHub Actions pueda comunicarse de manera desatendida tanto con Docker Hub como con la maquina virtual en AWS EC2, se procedio a aprovisionar los secretos y variables de entorno del repositorio bajo el estandar de minimo privilegio y encriptacion asimetrica.

#### 1. Aprovisionamiento de la instancia EC2 en AWS
Se comprobo la ejecucion de la instancia virtual en la consola de AWS EC2, verificando el estado *Running* y sus grupos de seguridad con trafico habilitado para SSH (puerto 22) y HTTP (puerto 80).

![Instancia EC2 en ejecucion en la consola de AWS](./Imagenes3.2/Screenshot%20From%202026-11-18%2011-15-28.png)

#### 2. Generacion del Personal Access Token en Docker Hub
En la configuracion de seguridad de Docker Hub (`Account Settings > Security`) se genero un token de acceso personal con permisos *Read, Write, Delete*, evitando el uso de la contrasena personal de la cuenta.

![Generacion de Personal Access Token en Docker Hub](./Imagenes3.2/Screenshot%20From%202026-11-18%2011-20-39.png)

#### 3. Registro de secretos de repositorio en GitHub Actions
En la seccion **Settings > Secrets and variables > Actions > Secrets**, se definieron los cinco secretos requeridos:

- `DOCKER_USERNAME`: Nombre de usuario de la cuenta en Docker Hub (`ciscoad`).
- `DOCKER_PASSWORD`: Personal Access Token generado en Docker Hub (`dckr_pat_...`).
- `SSH_HOST`: Direccion IP de la instancia EC2.
- `SSH_USER`: Usuario administrador del sistema operativo remoto (`ubuntu`).
- `SSH_PRIVATE_KEY`: Contenido integro y estructurado de la llave privada RSA `.pem`.

![Secretos configurados en GitHub Actions](./Imagenes3.2/Screenshot%20From%202026-11-18%2011-24-06.png)

#### 4. Creacion de variable de repositorio no secreta
En la pestana **Variables** se configuro la variable `IMAGE_NAME` con el valor `ciscoad/mi-app`, permitiendo parametrizar el nombre de la imagen en los scripts de despliegue.

![Variable IMAGE_NAME registrada en GitHub Actions](./Imagenes3.2/Screenshot%20From%202026-11-18%2011-26-26.png)

---

### Ejercicio 3.3: Workflow de Build, Push y Deploy

En este ejercicio se diseno e implemento el pipeline principal de Despliegue Continuo en el archivo `.github/workflows/cd.yml`. El flujo automatizado consta de dos jobs secuenciales: `build-and-push` y `deploy`.

#### 1. Arquitectura del pipeline de CD
- **Job `build-and-push`:**
  - Descarga el codigo fuente con `actions/checkout@v6`.
  - Inicia sesion en Docker Hub mediante `docker/login-action@v3`.
  - Compila la imagen Docker y la publica concurrentemente con dos etiquetas: `ciscoad/mi-app:${{ github.sha }}` (inmutable) y `ciscoad/mi-app:latest` (mutable).
- **Job `deploy`:**
  - Exige el cumplimiento previo del primer job mediante la directiva `needs: build-and-push`.
  - Establece un tunel seguro SSH hacia la instancia EC2 mediante `appleboy/ssh-action@v1.0.0`.
  - Descarga la nueva imagen (`docker pull`).
  - Ejecuta un contenedor temporal de validacion (`mi-app-new`) en el puerto alterno `3001`.
  - Ejecuta una espera pasiva y comprueba la respuesta del endpoint de salud: `curl -sf http://localhost:3001/health`.
  - Si la sonda es exitosa, detiene y elimina el contenedor en produccion anterior, levanta el nuevo contenedor en el puerto `80`, y remueve el contenedor de validacion.
  - Si la sonda falla, cancela el despliegue con salida de error (`exit 1`) manteniendo inalterado el contenedor activo en produccion.

![Archivo cd.yml estructurado en el editor de codigo](./Imagenes3.3/Screenshot%20From%202026-11-18%2011-37-30.png)

#### 2. Confirmacion y envio del pipeline a GitHub
Se anadieron las modificaciones del proyecto y el archivo de workflow, sincronizando los cambios con la rama `main`:

```bash
git add .
git commit -m "cd: agrega pipeline de CD con Docker y despliegue remoto"
git push origin main
```

![Confirmacion y envio del workflow cd.yml](./Imagenes3.3/Screenshot%20From%202026-11-18%2011-38-12.png)

![Verificacion de token de Docker Hub activo](./Imagenes3.3/Screenshot%20From%202026-11-18%2010-24-16.png)

#### 3. Ejecucion automatizada de los jobs en GitHub Actions
Al detectarse el evento `push`, GitHub Actions ejecuto secuencialmente ambos trabajos: primero la etapa de construccion y publicacion de la imagen, seguida del trabajo de despliegue remoto. Ambos culminaron con estado exitoso.

![Ejecucion exitosa de los jobs build-and-push y deploy](./Imagenes3.3/Screenshot%20From%202026-11-18%2012-03-17.png)

#### 4. Inspeccion de los logs del despliegue remoto
Dentro del paso de ejecucion remota via SSH, se constato la descarga de la imagen, la creacion del contenedor en el puerto `3001`, la evaluacion positiva del health check (`Health check exitoso. Activando nuevo contenedor...`) y la reubicacion final hacia el puerto `80`.

![Registro de ejecucion del comando remoto con health check exitoso](./Imagenes3.3/Screenshot%20From%202026-11-18%2012-03-26.png)

#### 5. Verificacion del servicio en la instancia EC2
Se abrio el navegador web y se consulto el endpoint `/health` utilizando la direccion IP publica del servidor en la nube. La aplicacion retorno codigo HTTP 200 con el mensaje JSON esperado:

```json
{"status":"ok"}
```

![Respuesta de la API desplegada en la nube via HTTP](./Imagenes3.3/Screenshot%20From%202026-11-18%2012-03-51.png)

---

### Ejercicio 3.4: Verificacion del despliegue y rollbacks

Este ejercicio tuvo como proposito demostrar la capacidad de actualizacion desatendida del pipeline, la resiliencia ante despliegues defectuosos y la estrategia tecnica de rollback manual.

#### 1. Actualizacion automatizada de version (v2.0.0)
Se modifico el archivo `app.js` para incorporar el campo de version en la respuesta del endpoint de salud:

```javascript
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok', version: '2.0.0' });
});
```

![Modificacion en app.js incorporando version 2.0.0](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-08-05.png)

Se realizo el commit y push correspondiente:

```bash
git add app.js
git commit -m "feat: actualiza mensaje de health check"
git push origin main
```

![Envio de los cambios de version a la rama main](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-11-38.png)

El pipeline de CD se disparo de forma automatica, completando las etapas de build, push y deploy sin intervencion manual:

![Ejecucion del pipeline para el despliegue de la version 2.0.0](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-14-21.png)

Al refrescar la URL publica en el navegador, se confirmo la actualizacion en tiempo real con la respuesta esperada:

```json
{"status":"ok","version":"2.0.0"}
```

![Validacion de la version 2.0.0 reflejada en produccion](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-15-28.png)

#### 2. Simulacion de fallo y prueba de resiliencia del Health Check
Para verificar que el mecanismo de proteccion del pipeline evita la exposicion de versiones defectuosas, se introdujo un fallo intencional en la aplicacion (alterando el endpoint `/health` o generando una referencia no definida).

![Introduccion de fallo intencional en el codigo](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-16-14.png)

Al enviarse el commit al repositorio remoto, el job `build-and-push` concluyo correctamente (la imagen se construyo), pero el job `deploy` **fallo de forma controlada (estado rojo)** durante la validacion en el puerto alterno `3001`:

```text
Health check FALLIDO. Cancelando despliegue y manteniendo version anterior.
```

![Fallo controlado del job deploy en GitHub Actions](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-17-58.png)

A pesar del fallo en el pipeline, al ingresar a la URL publica de la EC2 en el puerto 80, **la aplicacion anterior continuaba operando con normalidad**. Esto evidencia que el contenedor danado nunca llego a reemplazar al contenedor productivo en el puerto 80.

![Comprobacion de que la aplicacion productiva no sufrio caida](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-30-55.png)

#### 3. Etiquetado inmutable por SHA y procedimiento de Rollback manual
Para posibilitar el retorno a cualquier version anterior sin depender exclusivamente de `latest`, el pipeline almacena cada imagen con la etiqueta `${{ github.sha }}`.

En la interfaz de Docker Hub se evidencia la coexistencia de la etiqueta `latest` junto con las etiquetas unicas asociadas a cada commit de Git:

![Docker Hub mostrando imagenes etiquetadas por commit SHA](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-34-26.png)

##### Procedimiento operativo para ejecucion de Rollback:
Si tras un despliegue exitoso se detecta un error de logica de negocio en produccion, el administrador puede revertir el estado del sistema en pocos segundos conectandose via SSH al servidor y ejecutando los siguientes comandos:

```bash
# 1. Definir el identificador SHA del commit estable previo (ej. 7b679de...)
SHA_ESTABLE="ciscoad/mi-app:7b679de"

# 2. Descargar la imagen especifica correspondiente a dicha version
docker pull $SHA_ESTABLE

# 3. Detener y remover el contenedor con fallas en produccion
docker stop mi-app && docker rm mi-app

# 4. Levantar de inmediato el contenedor con la version previa sobre el puerto 80
docker run -d --name mi-app --restart unless-stopped -p 80:3000 $SHA_ESTABLE
```

Este procedimiento garantiza tiempos de recuperacion (RTO) minimos sin requerir la compilacion de una nueva imagen.

---

## 4. Practica Individual (Ejercicio 4)

Para la practica individual, se desarrollo y contenerizo un proyecto de dominio completo correspondiente a una **API RESTful de Gestion de Productos** (`lab5.2_Ejercicio4`), implementando un pipeline de CD integral con escaneo de seguridad automatizado y direccionamiento elastico sobre AWS.

### 1. Arquitectura y Componentes del Proyecto Propio
- **Entidad de Dominio:** Modelo de catalogo de productos tecnologicos con atributos estructurados (`id`, `name`, `description`, `price`, `stock`, `category`, `createdAt`).
- **Endpoints de Servicio:**
  - `GET /health`: Sonda de salud con reporte de estado operativo, tiempo de actividad (`uptime`) y marca temporal en formato ISO.
  - `GET /api/products`: Recuperacion de coleccion completa de productos en formato JSON.
  - `GET /api/products/:id`: Busqueda individual por identificador numerico.
  - `POST /api/products`: Creacion y validacion de nuevos registros con codigo HTTP 201.
  - `PUT /api/products/:id`: Actualizacion de atributos existentes.
  - `DELETE /api/products/:id`: Eliminacion logica y confirmacion de recurso.

### 2. Estructura de Directorios y Dockerfile Multi-Stage
Se adapto el `Dockerfile` para soportar la arquitectura modular con carpeta fuente `src/`:

```dockerfile
# Etapa 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

# Etapa 2: Runtime
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/src ./src
EXPOSE 3000
CMD ["node", "src/server.js"]
```

![Estructura del proyecto propio y configuracion de Dockerfile](./Imagenes4/Screenshot%20From%202026-11-18%2015-46-19.png)

### 3. Configuracion de Secretos y Variables para el Nuevo Repositorio
En el repositorio remoto `https://github.com/adrianOvando/lab5.2_Ejercicio4.git` se registraron los secretos de acceso para Docker Hub y la llave privada SSH de la instancia EC2.

![Configuracion de secretos de repositorio en lab5.2_Ejercicio4](./Imagenes4/Screenshot%20From%202026-11-18%2015-43-43.png)

### 4. Workflow de CD con Escaneo de Vulnerabilidades (Trivy) y Entorno de Produccion
Se configuro el workflow `.github/workflows/cd.yml` incorporando:
- Etiquetado de imagen `ciscoad/lab5-2-ejercicio4:${{ github.sha }}` y `ciscoad/lab5-2-ejercicio4:latest`.
- Paso de analisis de vulnerabilidades mediante **Aqua Security Trivy** (`aquasecurity/trivy-action@master`), inspeccionando el sistema operativo base y librerias en busca de fallos criticos o altos.
- Definicion de entorno de despliegue mediante la clausula `environment: production`.
- Script de despliegue en servidor remoto que valida el health check en el puerto `3001`, detiene cualquier contenedor previo en el puerto `80` (incluyendo contenedores del ejercicio anterior como `mi-app`) y activa el contenedor de la API de productos.

### 5. Ejecucion Automatizada del Pipeline de la Practica Individual
Al realizarse el push hacia la rama `main`, el pipeline ejecuto en su totalidad las etapas de checkout, autenticacion, build, push a Docker Hub, escaneo Trivy y despliegue sobre la maquina virtual EC2.

![Pipeline de CD completado con exito para la API de productos](./Imagenes4/Screenshot%20From%202026-11-18%2015-49-07.png)

### 6. Registro de Despliegue Remoto con Validacion de Salud
En los registros de ejecucion de la accion SSH se verifico la recepcion de la nueva imagen, la inicializacion en el puerto `3001`, la comprobacion afirmativa del health check y la publicacion en el puerto `80`.

![Log detallado del despliegue remoto en la instancia EC2](./Imagenes4/Screenshot%20From%202026-11-18%2015-49-53.png)

### 7. Verificacion de Endpoints en la Nube mediante IP Elastica
Para garantizar la persistencia del acceso sin dependencia de reinicios de la instancia en AWS, se configuro la **IP Elastica `18.213.236.16`**. Se valido la disponibilidad de los servicios en produccion:

- **Endpoint de salud (`http://18.213.236.16/health`):**
  Retorna el estado de disponibilidad del contenedor y su tiempo de actividad en linea:

![Verificacion del endpoint /health de la API propia](./Imagenes4/Screenshot%20From%202026-11-18%2015-50-07.png)

- **Endpoint de productos (`http://18.213.236.16/api/products`):**
  Retorna la coleccion completa de elementos almacenados, demostrando la operatividad de las rutas de negocio del contenedor:

![Respuesta en vivo del endpoint /api/products en produccion](./Imagenes4/Screenshot%20From%202026-11-18%2015-50-24.png)

### 8. Registro de Version e Inmutabilidad en Docker Hub
En el registro publico de Docker Hub se confirma la disponibilidad del artefacto de software publicado bajo `ciscoad/lab5-2-ejercicio4`, exhibiendo su etiqueta mutable `latest` y la etiqueta inmutable que corresponde exactamente con el identificador del commit SHA de Git (`01388e275c88ccc5333fbc82d001db8ee444a2e4`):

![Docker Hub con imagen y tag SHA para lab5-2-ejercicio4](./Imagenes4/Screenshot%20From%202026-11-18%2015-55-08.png)

---

## 5. Analisis de Seguridad y Escaneo de Vulnerabilidades

Como buena practica de DevSecOps, la integracion de herramientas de escaneo como **Trivy** (`aquasecurity/trivy-action`) en el pipeline de Despliegue Continuo aporta las siguientes ventajas:

1. **Deteccion Temprana de CVEs:** Identifica vulnerabilidades conocidas tanto en los paquetes del sistema operativo base (Alpine Linux) como en los paquetes de terceros instalados en `node_modules`.
2. **Cumplimiento de Politicas:** Permite establecer umbrales de severidad (`CRITICAL`, `HIGH`). Si se configurara con `exit-code: '1'`, el pipeline bloquearia de inmediato el despliegue de imagenes vulnerables a produccion.
3. **Minimizacion de la Superficie de Ataque:** Al haber seleccionado `node:20-alpine` como imagen base, el numero de binarios y dependencias se reduce al minimo indispensable, reduciendo drasticamente la presencia de vulnerabilidades en comparacion con imagenes completas como `node:20` (basadas en Debian).

---

## 6. Procedimiento de Rollback y Estrategia de Versionamiento

El despliegue continuo en entornos de produccion requiere mecanismos predecibles de recuperacion ante contingencias. La combinacion de dos estrategias garantiza alta disponibilidad y estabilidad operativa:

| Mecanismo | Funcion | Beneficio Principal |
|---|---|---|
| **Sonda de Salud (*Health Check*)** | Validación previa al cambio en puerto alterno (`3001`). | Impide que una versión rota o incapaz de iniciar reemplace al contenedor activo. |
| **Etiquetado Inmutable por SHA** | Vinculación 1:1 entre el commit de Git y la imagen en Docker Hub. | Permite regresar a cualquier versión histórica exacta en segundos mediante un simple `docker pull`. |

---

## 7. Conclusiones

1. **Eficiencia de los Builds Multi-Etapa:** La implementacion de `Dockerfile` multi-stage permitio construir imagenes limpias, ligeras y libres de dependencias de desarrollo o utilitarios innecesarios, reduciendo el consumo de ancho de banda en la red y acelerando los procesos de `pull` y `run` en los servidores en la nube.
2. **Seguridad y Gestion de Secretos:** El aislamiento de llaves privadas SSH y tokens de acceso dentro de los secretos de GitHub Actions evito la exposicion de credenciales criticas en el codigo fuente publico, satisfaciendo estandares industriales de seguridad en la nube.
3. **Resiliencia Mediante Despliegue Progresivo y Health Checks:** La logica de validacion implementada en el script de despliegue protege a los usuarios finales de experimentar caidas de servicio, verificando la salud del nuevo contenedor antes de enrutar el trafico productivo en el puerto 80.
4. **Trazabilidad Absoluta:** El etiquetado de artefactos con el SHA de Git establece un enlace directo entre el codigo fuente y el binario ejecutado en produccion, lo cual posibilita auditorias claras y rollbacks inmediatos ante incidentes.
5. **Estabilidad Operativa con IP Elastica:** La incorporacion de una IP Elastica de AWS garantiza la continuidad del servicio frente a cualquier reinicio o mantenimiento de la infraestructura subyacente.
