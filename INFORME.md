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
- Aplicar tecnicas de validacion en caliente mediante sondas de salud (*health checks*) en puertos alternos antes de la promocion a produccion en el puerto 80.
- Establecer politicas de versionamiento dual mediante etiquetas mutables (`latest`) y etiquetas inmutables referenciadas al identificador unico del commit (`github.sha`).
- Ejecutar procedimientos controlados de retroceso (*rollback*) ante incidentes en produccion aprovechando el historial inmutable de imagenes.

---

## 2. Requisitos y Entorno de Trabajo

Para el desarrollo satisfactorio de la practica se dispuso de las siguientes tecnologias, herramientas y recursos de infraestructura:

- **Estacion de Trabajo Local:** Sistema Operativo Linux Ubuntu x86_64, shell interactiva Bash, Git version 2.43+.
- **Motor de Contenedores:** Docker Community Edition version 29.1+, Docker Buildx y cliente Docker CLI local.
- **Registro de Imagenes:** Docker Hub bajo la cuenta de usuario `ciscoad`, autenticada mediante Personal Access Token (PAT) con permisos de lectura y escritura (*Read & Write*).
- **Infraestructura de Nube:** Instancia de Amazon Web Services (AWS) EC2 tipo `t3.micro` con sistema operativo Ubuntu Server.
- **Direccionamiento Fijo de Red:** Asignacion de una IP Elastica de AWS (`18.213.236.16`) asociada a la interfaz de red de la instancia EC2 para impedir la modificacion de la IP publica tras detenciones o reinicios.
- **Seguridad Perimetral (AWS Security Groups):** Reglas de entrada (*Inbound Rules*) configuradas para permitir trafico en el puerto 22 (SSH) y en el puerto 80 (HTTP) abierto al publico general.
- **Autenticacion SSH:** Par de llaves criptograficas RSA con archivo `.pem` de permisos restringidos (`chmod 400`).
- **Servidor de Aplicacion:** Node.js version 20 LTS, Express version 4/5, suites de pruebas Jest y Supertest.

---

## 3. Desarrollo de los Ejercicios Guiados

---

### Ejercicio 3.1: Creacion del Dockerfile y prueba local

En esta primera etapa se preparo el repositorio para soportar la contenerizacion de la aplicacion backend Node.js mediante un archivo `Dockerfile` con arquitectura multi-etapa (*multi-stage build*), acompanado de su correspondiente archivo de exclusion `.dockerignore`.

#### 1. Diseno del Dockerfile multi-stage

El patron multi-stage permite separar el entorno de compilacion del entorno final de ejecucion. En la primera etapa (`builder`) se instalan las dependencias de produccion (`npm ci --only=production`), mientras que en la segunda etapa (`runtime`) se parte de una imagen limpia de `node:20-alpine`, copiando unicamente los artefactos estrictamente necesarios.

![Dockerfile multi-stage creado en VS Code](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-11-40.png)

#### 2. Configuracion de exclusion con .dockerignore

Se definio el archivo `.dockerignore` para evitar el envio de archivos innecesarios al contexto del demonio de Docker, optimizando los tiempos de transferencia y el tamano de la imagen final.

![Archivo .dockerignore configurado en VS Code](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-11-52.png)

#### 3. Construccion de la imagen localmente

Se ejecuto el comando de construccion de la imagen etiquetada localmente como `mi-app:local`. El proceso concluyo exitosamente en 13 pasos, resolviendo las capas intermedias y produciendo el artefacto final.

```bash
docker build -t mi-app:local .
```

![Compilacion exitosa de la imagen Docker en la terminal](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-13-16.png)

#### 4. Ejecucion y prueba del contenedor local

Se instancio el contenedor en segundo plano enlazando el puerto `3000` y se verifico el endpoint `/health` mediante `curl`. La aplicacion respondio con el estado esperado `{"status":"ok"}`.

```bash
docker run -d -p 3000:3000 --name app-local mi-app:local
curl http://127.0.0.1:3000/health
```

![Ejecucion del contenedor y respuesta exitosa del endpoint /health](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-15-36.png)

#### 5. Limpieza de recursos locales

Finalizada la prueba local, se detuvo y elimino el contenedor de prueba correctamente.

```bash
docker stop app-local && docker rm app-local
```

![Detencion y eliminacion exitosa del contenedor app-local](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-16-27.png)

#### 6. Envio de archivos al repositorio remoto

Se anadieron el `Dockerfile` y `.dockerignore` al control de versiones y se enviaron a la rama `main` del repositorio `Lab5.2` en GitHub.

```bash
git add Dockerfile .dockerignore
git commit -m "feat: agrega Dockerfile multi-stage para contenerizacion"
git push origin main
```

![Commit y push del Dockerfile y .dockerignore a GitHub](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-17-10.png)

![Repositorio Lab5.2 en GitHub con Dockerfile y .dockerignore confirmados](./Imagenes3.1/Screenshot%20From%202026-11-18%2010-17-45.png)

---

### Ejercicio 3.2: Configuracion de secretos en GitHub

Para que GitHub Actions pueda comunicarse de manera desatendida con Docker Hub y la maquina virtual en AWS EC2, se aprovisionaron los secretos y variables de entorno del repositorio bajo el estandar de minimo privilegio.

#### 1. Verificacion de la instancia EC2 en AWS

Se confirmo el estado *En ejecucion* de la instancia `Lab.5.2` en la consola de AWS EC2 con IP publica `54.145.241.163` (posteriormente asociada a la IP Elastica `18.213.236.16`).

![Resumen detallado de la instancia EC2 en AWS con estado En ejecucion](./Imagenes3.2/Screenshot%20From%202026-11-18%2011-15-28.png)

![Lista de instancias EC2 con comprobaciones 3/3 satisfactorias](./Imagenes3.2/Screenshot%20From%202026-11-18%2011-24-06.png)

#### 2. Generacion del Personal Access Token en Docker Hub

En la seccion de seguridad de Docker Hub (`Account Settings > Security`) se generaron y gestionaron los tokens de acceso personal con permisos *Read, Write, Delete*, evitando el uso de la contrasena directa de la cuenta.

![Tokens de acceso personal activos en la cuenta de Docker Hub](./Imagenes3.3/Screenshot%20From%202026-11-18%2010-24-16.png)

#### 3. Registro de los cinco secretos en GitHub Actions

En **Settings > Secrets and variables > Actions > Secrets**, se configuraron los cinco secretos requeridos: `DOCKER_PASSWORD`, `DOCKER_USERNAME`, `SSH_HOST`, `SSH_PRIVATE_KEY` y `SSH_USER`.

![Los cinco secretos de repositorio configurados en GitHub Actions](./Imagenes3.2/Screenshot%20From%202026-11-18%2011-20-39.png)

#### 4. Creacion de variable de repositorio no secreta

En la pestana **Variables** se configuro la variable `IMAGE_NAME` con el valor `ciscoad/mi-app`, parametrizando el nombre de la imagen en los scripts de despliegue sin exponer informacion sensible.

![Variable IMAGE_NAME registrada con valor ciscoad/mi-app](./Imagenes3.2/Screenshot%20From%202026-11-18%2011-26-26.png)

---

### Ejercicio 3.3: Workflow de Build, Push y Deploy

En este ejercicio se implemento el pipeline principal de Despliegue Continuo en el archivo `.github/workflows/cd.yml`, compuesto por dos jobs secuenciales.

#### 1. Diseno del archivo cd.yml

El archivo de workflow define el job `build-and-push` (autenticacion en Docker Hub, compilacion y publicacion con etiquetas `latest` y `${{ github.sha }}`) y el job `deploy` (conexion SSH a la EC2, validacion de salud en puerto `3001` y promocion al puerto `80`).

![Archivo cd.yml con la estructura del pipeline de CD en VS Code](./Imagenes3.3/Screenshot%20From%202026-11-18%2011-37-30.png)

#### 2. Envio del workflow al repositorio remoto

Se confirmo y envio el archivo a la rama `main` disparando automaticamente la primera ejecucion del pipeline.

```bash
git add .github/workflows/cd.yml
git commit -m "cd: agrega pipeline de CD con Docker y despliegue remoto"
git push origin main
```

![Commit y push exitoso del workflow cd.yml a la rama main](./Imagenes3.3/Screenshot%20From%202026-11-18%2011-38-12.png)

#### 3. Ejecucion de los jobs en GitHub Actions

Al detectarse el evento `push`, GitHub Actions ejecuto los dos trabajos de forma secuencial. El job `build-and-push` finalizo correctamente compilando y publicando la imagen en Docker Hub.

![GitHub Actions: job build-and-push en ejecucion con todos los pasos completados](./Imagenes3.3/Screenshot%20From%202026-11-18%2012-03-17.png)

A continuacion el job `deploy` se conecto via SSH a la EC2, descargo la imagen, valido el health check en el puerto `3001` y activo el contenedor en el puerto `80`.

![GitHub Actions: job deploy completado exitosamente con Desplegar en servidor remoto](./Imagenes3.3/Screenshot%20From%202026-11-18%2012-03-26.png)

#### 4. Verificacion en el navegador

Finalizada la ejecucion, se abrio el navegador y se consulto el endpoint `/health` mediante la IP publica de la instancia EC2. La respuesta confirmo que el contenedor estaba operativo en la nube.

![Navegador mostrando la respuesta en vivo del endpoint /health en la IP de la EC2](./Imagenes3.3/Screenshot%20From%202026-11-18%2012-03-51.png)

---

### Ejercicio 3.4: Verificacion del despliegue y rollbacks

Este ejercicio demostro la capacidad de actualizacion automatica del pipeline, la resiliencia ante despliegues defectuosos y la estrategia tecnica de rollback.

#### 1. Actualizacion de version a 2.0.0

Se modifico `app.js` incorporando el campo `version` en la respuesta del endpoint `/health`:

```javascript
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok', version: '2.0.0' });
});
```

![Modificacion en app.js con version 2.0.0 en VS Code](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-08-05.png)

Se realizo el commit y push correspondiente:

```bash
git add app.js
git commit -m "feat: actualiza mensaje de health check"
git push origin main
```

![Commit y push de la actualizacion de version al repositorio remoto](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-11-38.png)

El pipeline se disparo automaticamente y al refrescar la URL se confirmo la version `2.0.0` en produccion:

![Navegador mostrando la respuesta actualizada con version 2.0.0 en la EC2](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-14-21.png)

#### 2. Simulacion de fallo intencional

Para verificar que el mecanismo de proteccion del health check funciona correctamente, se introdujo un fallo intencional alterando el puerto del servidor en `server.js` para que el contenedor no respondiera en el puerto esperado.

![server.js modificado con puerto 5000 para provocar el fallo del health check](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-15-28.png)

```bash
git add .
git commit -m "fallo aproposito"
git push origin main
```

![Commit y push del codigo defectuoso para disparar el fallo del pipeline](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-16-14.png)

El job `build-and-push` concluyo correctamente (la imagen se construyo), pero el job `deploy` fallo de forma controlada al detectar que el health check en el puerto `3001` no respondio satisfactoriamente. El contenedor anterior en el puerto `80` continuo operando sin interrupcion.

![GitHub Actions: pipeline con job build-and-push exitoso y job deploy FALLIDO en rojo](./Imagenes3.4/Screenshot%20From%202026-11-18%2015-17-58.png)

#### 3. Procedimiento de Rollback manual

Gracias al etiquetado inmutable por SHA, es posible revertir a cualquier version anterior en segundos. El procedimiento operativo consiste en conectarse via SSH a la EC2 y ejecutar:

```bash
# Reemplaza SHA_ANTERIOR por el SHA del commit estable anterior
docker pull ciscoad/mi-app:SHA_ANTERIOR
docker stop mi-app && docker rm mi-app
docker run -d --name mi-app --restart unless-stopped -p 80:3000 ciscoad/mi-app:SHA_ANTERIOR
```

Este mecanismo garantiza tiempos de recuperacion minimos sin necesidad de recompilar una nueva imagen.

---

## 4. Practica Individual (Ejercicio 4)

Para la practica individual se desarrollo y contenerizo una **API RESTful de Gestion de Productos** en el repositorio `lab5.2_Ejercicio4`, implementando un pipeline de CD integral con escaneo de seguridad automatizado y direccionamiento elastico sobre AWS.

### 1. Endpoints de la API de Productos

La aplicacion expone los siguientes servicios RESTful:

- `GET /health`: Estado del servicio con `uptime` y marca temporal ISO.
- `GET /api/products`: Coleccion completa de productos.
- `GET /api/products/:id`: Busqueda por identificador.
- `POST /api/products`: Creacion con validacion (HTTP 201).
- `PUT /api/products/:id`: Actualizacion de atributos.
- `DELETE /api/products/:id`: Eliminacion con confirmacion.

### 2. Configuracion de Secretos en el Nuevo Repositorio

Se configuraron los cinco secretos en el repositorio `lab5.2_Ejercicio4` con los mismos datos de acceso a la instancia EC2 y Docker Hub.

![Secretos configurados en el repositorio lab5.2_Ejercicio4](./Imagenes4/Screenshot%20From%202026-11-18%2015-43-43.png)

### 3. Inicializacion del Proyecto y Primer Push

Se inicializaron todos los archivos del proyecto (codigo fuente, `Dockerfile`, `.dockerignore`, `package.json`, workflow `cd.yml`) y se realizo el primer commit al repositorio.

```bash
git add .
git commit -m "feat: inicializa api de productos con dockerfile y pipeline cd"
git push -u origin main
```

![Terminal con la salida del primer git push al repositorio lab5.2_Ejercicio4](./Imagenes4/Screenshot%20From%202026-11-18%2015-46-19.png)

### 4. Repositorio en GitHub con Estructura Completa

Tras el primer push, el repositorio en GitHub reflejo la estructura completa del proyecto incluyendo las carpetas `src/`, `tests/`, `Dockerfile`, workflow de CD y demas archivos de configuracion.

![Repositorio lab5.2_Ejercicio4 en GitHub con todos los archivos publicados](./Imagenes4/Screenshot%20From%202026-11-18%2015-49-07.png)

### 5. Ejecucion Exitosa del Pipeline de CD

Al realizar el push, GitHub Actions ejecuto automaticamente el pipeline. Ambos jobs (`build-and-push` y `deploy`) completaron con estado exitoso en 54 segundos totales.

![Pipeline de CD completado con exito: build-and-push y deploy en verde](./Imagenes4/Screenshot%20From%202026-11-18%2015-49-53.png)

### 6. Verificacion de Endpoints en Produccion via IP Elastica

Se confirmo la disponibilidad de los servicios a traves de la IP Elastica `18.213.236.16`:

**Endpoint de salud** (`/health`): retorna estado operativo, tiempo de actividad en segundos y marca temporal:

![Respuesta del endpoint /health con uptime y timestamp desde la EC2](./Imagenes4/Screenshot%20From%202026-11-18%2015-50-07.png)

**Endpoint de productos** (`/api/products`): retorna la coleccion completa con los tres productos del catalogo (Laptop ThinkPad, Mouse Inalambrico, Teclado Mecanico RGB):

![Respuesta JSON completa del endpoint /api/products con los tres productos en produccion](./Imagenes4/Screenshot%20From%202026-11-18%2015-50-24.png)

### 7. Etiquetado Inmutable por SHA en Docker Hub

En el registro de Docker Hub se confirma la publicacion de la imagen `ciscoad/lab5-2-ejercicio4` con dos etiquetas: `latest` (mutable) y el SHA completo del commit `01388e275c88ccc5333fbc82d001db8ee444a2e4` (inmutable). Esta segunda etiqueta es la que permite ejecutar rollbacks exactos a cualquier version historica.

![Docker Hub mostrando la imagen con etiqueta latest y etiqueta SHA del commit](./Imagenes4/Screenshot%20From%202026-11-18%2015-55-08.png)

---

## 5. Estrategia de Versionamiento y Rollback

| Mecanismo | Funcion | Beneficio Principal |
|---|---|---|
| **Health Check en puerto alterno** | Validacion previa del nuevo contenedor en el puerto `3001` antes de sustituir al activo en el puerto `80`. | Impide que una version rota reemplace al contenedor productivo. |
| **Etiquetado inmutable por SHA** | Vinculacion 1:1 entre el commit de Git y la imagen publicada en Docker Hub. | Permite revertir a cualquier version historica exacta en segundos. |

---

## 6. Conclusiones

1. **Eficiencia de los Builds Multi-Etapa:** La implementacion de `Dockerfile` multi-stage produjo imagenes limpias y livianas (46 MB) libres de dependencias de desarrollo, reduciendo el tiempo de descarga en la EC2.
2. **Seguridad en la Gestion de Secretos:** El aislamiento de llaves privadas SSH y tokens de Docker Hub dentro de los secretos de GitHub Actions evito la exposicion de credenciales sensibles en el codigo fuente publico.
3. **Resiliencia mediante Health Checks:** La logica de validacion previa en el puerto `3001` demostro proteger a los usuarios finales de interrupciones de servicio ante despliegues defectuosos, tal como se evidencio durante la simulacion de fallo del Ejercicio 3.4.
4. **Trazabilidad Absoluta:** El etiquetado de artefactos con el SHA de Git establece un enlace directo entre el codigo fuente y el binario ejecutado en produccion, posibilitando auditorias claras y rollbacks inmediatos.
5. **Estabilidad Operativa con IP Elastica:** La incorporacion de una IP Elastica de AWS garantiza la continuidad del servicio y la permanencia de la URL publica frente a cualquier reinicio o mantenimiento de la infraestructura subyacente.
