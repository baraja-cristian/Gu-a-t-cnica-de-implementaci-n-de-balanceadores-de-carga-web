# Guía técnica de implementación de balanceadores de carga web para Moodle (UPEC)

> Arquitectura distribuida con NGINX, Docker, Redis, MariaDB y NFS

---

## Índice

1. [Introducción](#1-introducción)
2. [Fundamentos teóricos](#2-fundamentos-teóricos)
   - 2.1 [Balanceo de carga](#21-balanceo-de-carga)
   - 2.2 [Alta disponibilidad](#22-alta-disponibilidad)
   - 2.3 [Escalabilidad horizontal](#23-escalabilidad-horizontal)
   - 2.4 [Arquitecturas stateless y estado compartido](#24-arquitecturas-stateless-y-estado-compartido)
   - 2.5 [Algoritmos de balanceo](#25-algoritmos-de-balanceo)
   - 2.6 [Contenedores y Docker](#26-contenedores-y-docker)
   - 2.7 [Proxy inverso](#27-proxy-inverso)
3. [Arquitectura del sistema](#3-arquitectura-del-sistema)
   - 3.1 [Descripción general](#31-descripción-general)
   - 3.2 [Flujo de tráfico](#32-flujo-de-tráfico)
   - 3.3 [Componentes y responsabilidades](#33-componentes-y-responsabilidades)
   - 3.4 [Tabla de servidores e IPs](#34-tabla-de-servidores-e-ips)
4. [Implementación](#4-implementación)
   - 4.1 [Instalación de Docker Engine (todos los servidores)](#41-instalación-de-docker-engine-todos-los-servidores)
     - 4.1.1 [Desinstalación de paquetes en conflicto](#411-desinstalación-de-paquetes-en-conflicto)
     - 4.1.2 [Configuración del repositorio oficial](#412-configuración-del-repositorio-oficial)
     - 4.1.3 [Instalación de paquetes Docker](#413-instalación-de-paquetes-docker)
     - 4.1.4 [Habilitación del servicio](#414-habilitación-del-servicio)
     - 4.1.5 [Verificación de la instalación](#415-verificación-de-la-instalación)
     - 4.1.6 [Configuración de permisos de usuario](#416-configuración-de-permisos-de-usuario)
     - 4.1.7 [Posibles errores en la instalación de Docker](#417-posibles-errores-en-la-instalación-de-docker)
   - 4.2 [Despliegue de base de datos y Redis (Servidor 1)](#42-despliegue-de-base-de-datos-y-redis-servidor-1---ip-172202081122)
     - 4.2.1 [Estructura del directorio](#421-estructura-del-directorio)
     - 4.2.2 [Configuración de docker-compose.yml](#422-configuración-de-docker-composeyml)
     - 4.2.3 [Despliegue de los contenedores](#423-despliegue-de-los-contenedores)
     - 4.2.4 [Verificación del estado](#424-verificación-del-estado)
     - 4.2.5 [Posibles errores en el despliegue de DB y Redis](#425-posibles-errores-en-el-despliegue-de-db-y-redis)
   - 4.3 [Configuración del almacenamiento NFS (Servidor 2)](#43-configuración-del-almacenamiento-nfs-servidor-2---ip-17220208167)
     - 4.3.1 [Instalación de NFS](#431-instalación-de-nfs)
     - 4.3.2 [Habilitación del servicio NFS](#432-habilitación-del-servicio-nfs)
     - 4.3.3 [Creación del directorio compartido](#433-creación-del-directorio-compartido)
     - 4.3.4 [Configuración de exports](#434-configuración-de-exports)
     - 4.3.5 [Verificación de exports](#435-verificación-de-exports)
     - 4.3.6 [Configuración del firewall y SELinux](#436-configuración-del-firewall-y-selinux)
     - 4.3.7 [Posibles errores en la configuración NFS](#437-posibles-errores-en-la-configuración-nfs)
   - 4.4 [Despliegue de nodos Moodle (Servidores 4 y 5)](#44-despliegue-de-nodos-moodle-servidores-4-y-5---ips-172202081173-y-172202081169)
     - 4.4.1 [Preparación del entorno](#441-preparación-del-entorno)
     - 4.4.2 [Configuración de docker-compose.yml](#442-configuración-de-docker-composeyml)
     - 4.4.3 [Dockerfile de Moodle](#443-dockerfile-de-moodle)
     - 4.4.4 [Archivo config.php de Moodle](#444-archivo-configphp-de-moodle)
     - 4.4.5 [Despliegue del contenedor](#445-despliegue-del-contenedor)
     - 4.4.6 [Restricción de acceso por firewall](#446-restricción-de-acceso-por-firewall)
     - 4.4.7 [Posibles errores en los nodos Moodle](#447-posibles-errores-en-los-nodos-moodle)
   - 4.5 [Configuración del balanceador NGINX (Servidor 3)](#45-configuración-del-balanceador-nginx-servidor-3---ip-17220208174)
     - 4.5.1 [Estructura de archivos](#451-estructura-de-archivos)
     - 4.5.2 [Configuración de docker-compose.yml](#452-configuración-de-docker-composeyml)
     - 4.5.3 [Archivo nginx.conf](#453-archivo-nginxconf)
     - 4.5.4 [Despliegue del contenedor](#454-despliegue-del-contenedor)
     - 4.5.5 [Posibles errores en NGINX](#455-posibles-errores-en-nginx)
5. [Verificación del sistema completo](#5-verificación-del-sistema-completo)
   - 5.1 [Comprobación de conectividad entre servicios](#51-comprobación-de-conectividad-entre-servicios)
   - 5.2 [Validación del balanceo de carga](#52-validación-del-balanceo-de-carga)
   - 5.3 [Prueba de failover](#53-prueba-de-failover)
6. [Conclusión](#6-conclusión)

---

## 1. Introducción

El presente documento describe la implementación de una arquitectura de balanceo de carga web para el Entorno Virtual de Aprendizaje (EVA) de la Universidad Politécnica Estatal del Carchi (UPEC), basada en Moodle, utilizando tecnologías open source y contenedores Docker.

Una plataforma educativa que concentra la actividad de cientos o miles de usuarios simultáneos no puede depender de un único servidor. Cuando ese servidor falla, toda la institución pierde acceso al sistema. Cuando la demanda supera su capacidad, el rendimiento se degrada para todos los usuarios. La solución a ambos problemas es la misma: distribuir la carga entre múltiples instancias de la aplicación, coordinadas por un componente central que decide a cuál de ellas enviar cada petición entrante.

Esta guía cubre la implementación completa de dicha solución, incluyendo:

- El balanceador de carga NGINX como punto de entrada único.
- Múltiples instancias dockerizadas de Moodle como nodos de aplicación.
- Redis como almacén centralizado de sesiones y caché.
- MariaDB como base de datos relacional compartida.
- NFS como sistema de archivos compartido para el directorio `moodledata`.

Todos los servicios, con excepción de NFS, se ejecutan dentro de contenedores Docker, lo que garantiza reproducibilidad, portabilidad y facilidad de mantenimiento.

---

## 2. Fundamentos teóricos

### 2.1 Balanceo de carga

El balanceo de carga (*load balancing*) es la técnica que distribuye el tráfico de red entrante entre un conjunto de servidores backend, denominados colectivamente *upstream pool* o *backend pool*. El componente que realiza esta distribución se denomina balanceador de carga (*load balancer*).

El balanceador actúa como intermediario entre el cliente y los servidores de aplicación. Desde la perspectiva del usuario, existe un único punto de acceso (una IP o un nombre de dominio). Internamente, el balanceador selecciona el servidor destino según el algoritmo configurado y reenvía la petición.

Los beneficios principales son:

- **Rendimiento:** múltiples servidores atienden las peticiones en paralelo, reduciendo los tiempos de respuesta.
- **Disponibilidad:** si un servidor falla, el balanceador deja de enviarle tráfico y lo redistribuye entre los restantes.
- **Capacidad:** agregar servidores al pool incrementa la capacidad total del sistema sin modificar su arquitectura.

### 2.2 Alta disponibilidad

La alta disponibilidad (*High Availability*, HA) es la propiedad de un sistema de permanecer operativo ante la ocurrencia de fallos en uno o más de sus componentes. Se mide habitualmente como porcentaje de tiempo de actividad (*uptime*): 99.9 % implica menos de 9 horas de inactividad al año; 99.99 % implica menos de 53 minutos.

En esta arquitectura se logra mediante:

- **Redundancia activa de nodos:** dos instancias de Moodle atienden peticiones simultáneamente. Si una falla, la otra continúa operando.
- **Failover automático en NGINX:** los parámetros `max_fails` y `fail_timeout` en la directiva `upstream` permiten a NGINX detectar backends no disponibles y excluirlos del pool de forma automática, sin intervención manual.
- **Persistencia de estado en servicios externos:** Redis y MariaDB centralizan el estado de la aplicación, de modo que la pérdida de un nodo de aplicación no implica pérdida de datos ni de sesiones activas.

### 2.3 Escalabilidad horizontal

La escalabilidad horizontal (*horizontal scaling* o *scale-out*) consiste en añadir más instancias de un servicio para aumentar la capacidad del sistema, en contraposición a la escalabilidad vertical (*scale-up*), que consiste en aumentar los recursos (CPU, RAM) de un único servidor.

La escalabilidad horizontal es preferible en entornos de producción porque:

- No tiene un límite físico tan rígido como la capacidad máxima de un servidor.
- Permite agregar y retirar nodos sin interrumpir el servicio.
- Distribuye el riesgo de fallo entre múltiples máquinas independientes.

Para que sea posible, los nodos de aplicación deben ser **stateless** (sin estado local), condición descrita en la siguiente sección.

### 2.4 Arquitecturas stateless y estado compartido

Un servicio es *stateless* (sin estado) cuando no almacena información de sesión o datos persistentes localmente entre peticiones. Cada petición se puede atender de forma independiente por cualquier nodo del pool, sin que el resultado dependa de cuál nodo la atendió anteriormente.

Moodle, en su configuración predeterminada, almacena las sesiones en el sistema de archivos local del servidor. Esto lo hace *stateful* y lo incompatible con el balanceo de carga: si el usuario A inicia sesión contra el Nodo 1, su sesión existe solo en el disco de ese nodo; si la siguiente petición la atiende el Nodo 2, este no encuentra la sesión y obliga al usuario a autenticarse nuevamente.

La solución es externalizar el estado a servicios compartidos accesibles por todos los nodos:

| Tipo de estado | Servicio | Justificación |
|---|---|---|
| Sesiones de usuario | Redis | Almacén en memoria de alta velocidad, soporte nativo en Moodle mediante `session_handler_class = redis` |
| Datos relacionales | MariaDB | Base de datos centralizada única para todos los nodos |
| Archivos subidos | NFS | Sistema de archivos de red montado en todos los nodos bajo el mismo path |

### 2.5 Algoritmos de balanceo

NGINX open source soporta los siguientes algoritmos en el bloque `upstream`:

| Algoritmo | Directiva | Comportamiento |
|---|---|---|
| Round Robin | (predeterminado, sin directiva) | Distribuye las peticiones de forma cíclica secuencial entre todos los servidores disponibles. Apropiado cuando los servidores tienen capacidad homogénea y las peticiones tienen costos similares. |
| Least Connections | `least_conn;` | Envía cada nueva petición al servidor con el menor número de conexiones activas en ese momento. Recomendado cuando las peticiones tienen tiempos de procesamiento variables, como ocurre en Moodle con operaciones de carga de archivos, ejecución de cuestionarios o generación de informes. |
| IP Hash | `ip_hash;` | Calcula un hash de la IP de origen del cliente y lo mapea siempre al mismo servidor. Garantiza que un cliente específico sea atendido siempre por el mismo nodo, lo que elimina problemas de sesión sin necesidad de un almacén externo. No es la opción elegida en esta arquitectura, dado que Redis centraliza las sesiones de forma más robusta. |
| Generic Hash | `hash $variable;` | Permite definir el hash sobre cualquier variable NGINX (URL, cabecera, cookie). Ofrece mayor flexibilidad para casos de uso específicos. |

En esta implementación se utiliza **Least Connections** (`least_conn`), dado que Moodle genera peticiones de costo computacional heterogéneo y este algoritmo distribuye la carga de forma más equitativa bajo esas condiciones.

### 2.6 Contenedores y Docker

Un contenedor es una unidad de software que empaqueta el código de una aplicación junto con todas sus dependencias (bibliotecas, runtime, configuración) en un entorno aislado y reproducible. A diferencia de las máquinas virtuales, los contenedores comparten el kernel del sistema operativo anfitrión, lo que los hace considerablemente más ligeros en consumo de recursos.

Docker es la plataforma de contenedorización más utilizada en el ecosistema open source. Sus componentes relevantes para esta guía son:

- **Docker Engine:** el demonio que gestiona la creación, ejecución y detención de contenedores.
- **Dockerfile:** archivo de texto que describe, paso a paso, cómo construir una imagen de contenedor personalizada.
- **Docker Compose:** herramienta para definir y gestionar aplicaciones multi-contenedor mediante un archivo `docker-compose.yml`, que describe los servicios, redes y volúmenes de la aplicación.
- **Volúmenes:** mecanismo para persistir datos generados por los contenedores fuera de su sistema de archivos efímero.

### 2.7 Proxy inverso

Un proxy inverso (*reverse proxy*) es un servidor que recibe peticiones en nombre de uno o más servidores backend y las reenvía a estos últimos. El cliente solo conoce la dirección del proxy inverso, no la de los servidores backend.

NGINX actúa en esta arquitectura simultáneamente como proxy inverso y como balanceador de carga. Esta combinación permite:

- Centralizar la terminación TLS/SSL (si se configura HTTPS).
- Agregar cabeceras HTTP que informen a Moodle sobre la IP real del cliente (`X-Real-IP`, `X-Forwarded-For`).
- Controlar tiempos de espera y comportamiento ante fallos de los backends.
- Registrar el acceso en un formato de log enriquecido que indica a cuál backend se envió cada petición.

---

## 3. Arquitectura del sistema

### 3.1 Descripción general

La arquitectura implementada sigue el principio de separación de responsabilidades: cada servidor cumple una función específica y bien delimitada. Esto facilita el mantenimiento, la depuración y la escalabilidad futura.

El flujo de una petición es el siguiente:

1. El usuario accede a la URL de Moodle desde su navegador.
2. La petición llega al servidor NGINX (Servidor 3, balanceador).
3. NGINX selecciona el nodo Moodle con menor número de conexiones activas.
4. La petición es reenviada al nodo seleccionado.
5. El nodo Moodle consulta MariaDB para recuperar datos relacionales y Redis para verificar la sesión del usuario.
6. Si la operación requiere acceso al directorio `moodledata` (archivos subidos, caché de archivos), el nodo accede al volumen NFS montado localmente.
7. La respuesta generada por Moodle es devuelta al usuario a través de NGINX.

### 3.2 Flujo de tráfico

```
Usuarios / Internet
        |
        v
+-------------------------+
| NGINX - Balanceador     |  Servidor 3 - 172.20.208.174
| (Reverse Proxy / LB)    |
+-------------------------+
         |          |
         v          v
  +----------+  +----------+
  |  Nodo 1  |  |  Nodo 2  |   Servidores 4 y 5
  |  Moodle  |  |  Moodle  |   172.20.208.173 / .169
  +----------+  +----------+
         |          |
         +----+-----+
              |
     +--------+---------+
     |                  |
     v                  v
+----------+      +----------+
|  MariaDB |      |  Redis   |   Servidor 1 - 172.20.208.122
+----------+      +----------+
     |
     v
+----------+
|   NFS    |                    Servidor 2 - 172.20.208.167
+----------+
```

### 3.3 Componentes y responsabilidades

**NGINX**
Actua como punto de entrada unico para todo el trafico HTTP. Implementa el algoritmo de balanceo `least_conn`, gestiona el failover automatico mediante deteccion de backends caidos (`max_fails`, `fail_timeout`), agrega las cabeceras necesarias para que Moodle identifique correctamente la IP del cliente, y registra a que backend fue enviada cada peticion.

**Nodos Moodle**
Cada nodo ejecuta una instancia identica de Moodle dentro de un contenedor Docker basado en `php:8.1-apache`. Los nodos son completamente stateless: no almacenan sesiones localmente (delegadas a Redis) ni archivos de datos localmente (delegados a NFS). La identidad de configuracion entre nodos se garantiza usando el mismo `Dockerfile` y el mismo `config.php` en todos.

**MariaDB**
Motor de base de datos relacional que almacena toda la informacion de la plataforma: usuarios, cursos, actividades, calificaciones, configuracion del sistema, etc. Es el unico punto de escritura para datos persistentes relacionales y es compartido por todos los nodos.

**Redis**
Almacen de datos en memoria que gestiona las sesiones activas de todos los usuarios. Cuando un usuario se autentica contra el Nodo 1 y su siguiente peticion es atendida por el Nodo 2, este encuentra la sesion en Redis y no requiere nueva autenticacion. Tambien puede actuar como backend de cache de la aplicacion Moodle.

**NFS (Network File System)**
Servidor de archivos de red que exporta el directorio `/srv/moodledata`. Este directorio es montado por todos los nodos Moodle bajo la ruta `/var/moodledata` dentro del contenedor. Esto garantiza que los archivos subidos por los usuarios (materiales de curso, envios de tareas, archivos de recursos) sean accesibles desde cualquier nodo, independientemente de cual atendio la peticion de subida.

### 3.4 Tabla de servidores e IPs

| Servidor | Rol | IP |
|---|---|---|
| Servidor 1 | Base de datos (MariaDB) + Cache (Redis) | 172.20.208.122 |
| Servidor 2 | Almacenamiento compartido (NFS) | 172.20.208.167 |
| Servidor 3 | Balanceador de carga (NGINX) | 172.20.208.174 |
| Servidor 4 | Nodo de aplicacion Moodle 1 | 172.20.208.173 |
| Servidor 5 | Nodo de aplicacion Moodle 2 | 172.20.208.169 |

---

## 4. Implementación

### 4.1 Instalación de Docker Engine (todos los servidores)

Docker Engine debe instalarse en todos los servidores que vayan a ejecutar contenedores: Servidor 1 (DB/Redis), Servidor 3 (NGINX), Servidor 4 y Servidor 5 (nodos Moodle). El Servidor 2 (NFS) no requiere Docker.

Los pasos que siguen corresponden a la instalación sobre sistemas basados en RHEL/AlmaLinux/Rocky Linux usando el gestor de paquetes `dnf`.

#### 4.1.1 Desinstalación de paquetes en conflicto

Antes de instalar Docker Engine desde el repositorio oficial, es necesario eliminar cualquier versión previa o paquetes alternativos (como Podman o runc) que puedan causar conflictos. Distribuciones como AlmaLinux incluyen Podman instalado por defecto, cuyo demonio entraría en conflicto con el de Docker.

```bash
sudo dnf remove docker \
                docker-client \
                docker-client-latest \
                docker-common \
                docker-latest \
                docker-latest-logrotate \
                docker-logrotate \
                docker-engine \
                podman \
                runc
```

> Este comando no produce error si los paquetes no están instalados; simplemente indica que no hay nada que eliminar. Es seguro ejecutarlo en sistemas limpios.

#### 4.1.2 Configuración del repositorio oficial

Se añade el repositorio oficial de Docker para RHEL/AlmaLinux. Esto garantiza que `dnf` instale la versión oficial y mantenida por Docker, Inc., en lugar de una versión de la distribución que puede estar desactualizada.

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

#### 4.1.3 Instalación de paquetes Docker

Se instalan los componentes del ecosistema Docker:

- `docker-ce`: Docker Community Edition (el motor principal).
- `docker-ce-cli`: interfaz de línea de comandos.
- `containerd.io`: runtime de bajo nivel para la gestión de contenedores.
- `docker-buildx-plugin`: extensión para la construcción de imágenes multiplataforma.
- `docker-compose-plugin`: integración de Docker Compose como subcomando de `docker`.

```bash
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### 4.1.4 Habilitación del servicio

Se habilita el servicio Docker para que se inicie automáticamente con el sistema operativo (`enable`) y se arranca inmediatamente (`--now`).

```bash
sudo systemctl enable --now docker
```

#### 4.1.5 Verificación de la instalación

Se descarga y ejecuta una imagen de prueba oficial. Si Docker está correctamente instalado, el contenedor imprime un mensaje de confirmación y termina.

```bash
sudo docker run hello-world
```

#### 4.1.6 Configuración de permisos de usuario

Por defecto, el socket de Docker solo es accesible por `root`. Para ejecutar comandos `docker` sin `sudo`, se añade el usuario actual al grupo `docker`. Esta configuración tiene implicaciones de seguridad en entornos de producción; evaluar según la política de la organización.

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

> El comando `newgrp docker` aplica el cambio de grupo en la sesión actual sin necesidad de cerrar sesión. En sesiones SSH, puede ser necesario reconectar.

#### 4.1.7 Posibles errores en la instalación de Docker

**Error: `Cannot connect to the Docker daemon`**

```
Cannot connect to the Docker daemon at unix:///var/run/docker.sock
```

El servicio Docker no está en ejecución. Verificar su estado e iniciarlo:

```bash
sudo systemctl status docker
sudo systemctl start docker
```

**Error: `Permission denied` al ejecutar `docker` sin sudo**

El usuario no pertenece al grupo `docker` o la sesión no ha sido reiniciada tras añadirlo. Ejecutar `newgrp docker` o reconectar la sesión SSH.

**Error: conflicto de paquetes con Podman**

```
Error: Transaction test error: file /usr/bin/... conflicts between attempted installs
```

Asegurarse de haber ejecutado el paso de desinstalación de paquetes en conflicto antes de la instalación.

**Error: `docker-compose` no reconocido como comando**

En la versión moderna de Docker, Compose se invoca como `docker compose` (sin guion), no como `docker-compose`. Verificar que el plugin está instalado:

```bash
docker compose version
```

---

### 4.2 Despliegue de base de datos y Redis (Servidor 1 - IP 172.20.208.122)

Este servidor centraliza dos servicios críticos: MariaDB, que almacena todos los datos relacionales de Moodle, y Redis, que gestiona las sesiones de usuario y la caché. Ejecutarlos en el mismo servidor reduce la latencia de red entre ambos y simplifica la administración, aunque en instalaciones de mayor escala pueden separarse en servidores dedicados.

Se utiliza Docker Compose para gestionar ambos contenedores como una unidad, con sus redes y volúmenes definidos declarativamente.

#### 4.2.1 Estructura del directorio

```bash
mkdir dbRedis-moodle
cd dbRedis-moodle
```

#### 4.2.2 Configuración de docker-compose.yml

El archivo `docker-compose.yml` define los dos servicios. Los volúmenes con nombre (`dbdata`, `redisdata`) garantizan que los datos persistan aunque los contenedores sean eliminados y recreados.

La opción `restart: always` instruye a Docker para que reinicie automáticamente los contenedores en caso de fallo o reinicio del sistema operativo, lo que es esencial para la disponibilidad del servicio.

```bash
nano docker-compose.yml
```

```yaml
version: "3.9"

services:
  mariadb:
    image: mariadb:10.6
    container_name: moodle_db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: moodle
      MYSQL_USER: moodle
      MYSQL_PASSWORD: secret
    volumes:
      - dbdata:/var/lib/mysql
    ports:
      - "3306:3306"

  redis:
    image: redis:alpine
    container_name: moodle_redis
    restart: always
    command: redis-server --appendonly yes --requirepass redispass
    volumes:
      - redisdata:/data
    ports:
      - "6379:6379"

volumes:
  dbdata:
  redisdata:
```

**Notas de configuración:**

- `--appendonly yes` en Redis habilita la persistencia en disco mediante AOF (*Append Only File*). Sin esta opción, Redis opera exclusivamente en memoria y los datos se pierden ante un reinicio.
- `--requirepass redispass` protege el acceso a Redis con contraseña. Esta contraseña debe coincidir exactamente con el valor de `session_redis_auth` en `config.php` de Moodle.
- El puerto 3306 de MariaDB y 6379 de Redis se publican en la interfaz del servidor para que los nodos Moodle puedan conectarse desde sus propias IPs.

#### 4.2.3 Despliegue de los contenedores

```bash
docker compose up -d --build
```

La opción `-d` (*detached*) ejecuta los contenedores en segundo plano. `--build` fuerza la construcción de imágenes locales si existiese un `Dockerfile`, aunque en este caso se usan imágenes oficiales del registry.

#### 4.2.4 Verificación del estado

```bash
docker ps
```

Ambos contenedores (`moodle_db` y `moodle_redis`) deben aparecer con estado `Up`.

Para verificar conectividad desde los nodos Moodle antes de continuar:

```bash
# Desde los Servidores 4 o 5, verificar acceso a MariaDB
nc -zv 172.20.208.122 3306

# Verificar acceso a Redis
nc -zv 172.20.208.122 6379
```

#### 4.2.5 Posibles errores en el despliegue de DB y Redis

**El contenedor de MariaDB se reinicia constantemente (`Restarting`)**

Revisar los logs para identificar el problema:

```bash
docker logs moodle_db
```

Causa frecuente: el directorio de datos ya contiene una base de datos incompatible con la versión de imagen especificada. Eliminar el volumen y volver a crear:

```bash
docker compose down -v
docker compose up -d
```

> Precaución: `-v` elimina los volúmenes y con ellos todos los datos. Usar solo en instalaciones nuevas o cuando los datos no son necesarios.

**Redis rechaza conexiones con error `NOAUTH`**

Indica que el cliente intenta conectarse sin contraseña o con una contraseña incorrecta. Verificar que el valor de `--requirepass` en el `docker-compose.yml` coincide con `session_redis_auth` en `config.php`.

**Error de puerto ya en uso: `bind: address already in use`**

Otro proceso está ocupando el puerto 3306 o 6379. Identificarlo:

```bash
sudo ss -tlnp | grep 3306
```

---

### 4.3 Configuración del almacenamiento NFS (Servidor 2 - IP 172.20.208.167)

NFS (*Network File System*) es un protocolo que permite a un servidor exportar un directorio de su sistema de archivos local para que otros sistemas lo monten como si fuera un directorio local. En esta arquitectura, NFS resuelve el problema de consistencia de archivos en entornos multi-nodo: los archivos subidos por los usuarios desde cualquier nodo se almacenan en un único lugar accesible por todos.

El directorio compartido es `/srv/moodledata`, que corresponde al `dataroot` de Moodle (la variable `$CFG->dataroot` en `config.php`). Este directorio contiene archivos de cursos, envíos de tareas, caché de archivos y otros datos que Moodle gestiona fuera de la base de datos.

#### 4.3.1 Instalación de NFS

```bash
sudo dnf install nfs-utils -y
```

El paquete `nfs-utils` incluye el servidor NFS, las herramientas de gestión de exports y los utilitarios de diagnóstico.

#### 4.3.2 Habilitación del servicio NFS

```bash
sudo systemctl enable --now nfs-server
sudo systemctl status nfs-server
```

`enable --now` configura el inicio automático del servicio en cada arranque del sistema y lo inicia inmediatamente.

#### 4.3.3 Creación del directorio compartido

```bash
mkdir -p /srv/moodledata
chown -R nobody:nogroup /srv/moodledata
chmod -R 777 /srv/moodledata
```

El directorio se asigna al usuario `nobody` y grupo `nogroup`, que son las cuentas sin privilegios usadas por NFS cuando la opción `no_root_squash` no está activa para clientes específicos. Los permisos `777` garantizan que los contenedores Moodle (que ejecutan como `www-data`) puedan leer y escribir sin restricciones.

> En entornos de producción con requerimientos de seguridad estrictos, se recomienda afinar los permisos y usar `root_squash` con UIDs mapeados explícitamente.

#### 4.3.4 Configuración de exports

El archivo `/etc/exports` define qué directorios se exportan y a qué clientes, con qué permisos.

```bash
nano /etc/exports
```

```
/srv/moodledata 172.20.208.0/24(rw,sync,no_subtree_check,no_root_squash)
/srv/moodledata 172.20.208.173(rw,sync,no_subtree_check,no_root_squash)
/srv/moodledata 172.20.208.169(rw,sync,no_subtree_check,no_root_squash)
```

**Significado de las opciones:**

| Opción | Descripción |
|---|---|
| `rw` | Permite lectura y escritura desde el cliente. |
| `sync` | Las escrituras se confirman al disco antes de responder al cliente. Garantiza consistencia ante fallos, a costa de algo de rendimiento. |
| `no_subtree_check` | Deshabilita la verificación de que el archivo solicitado pertenece al árbol exportado. Mejora el rendimiento y evita problemas con archivos renombrados. |
| `no_root_squash` | El usuario root del cliente conserva privilegios de root en el servidor NFS. Necesario para que Docker pueda crear directorios y gestionar permisos correctamente. |

#### 4.3.5 Verificación de exports

Aplicar los cambios sin reiniciar el servicio y verificar los exports activos:

```bash
sudo exportfs -rv
```

La salida debe mostrar el directorio exportado con las opciones configuradas para cada cliente.

#### 4.3.6 Configuración del firewall y SELinux

El firewall debe permitir los servicios NFS, mountd y rpc-bind, que son los componentes del protocolo NFS sobre TCP/UDP.

```bash
sudo systemctl enable firewalld
sudo systemctl start firewalld
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --reload
```

SELinux puede bloquear el acceso NFS en sistemas RHEL. Se configura en modo permisivo para evitar conflictos con el montaje desde Docker:

```bash
nano /etc/selinux/config
```

Modificar la línea:

```
SELINUX=permissive
```

```bash
reboot
```

> SELinux en modo `permissive` registra las violaciones de política pero no las bloquea, permitiendo el funcionamiento del sistema mientras se analizan las políticas específicas a aplicar.

#### 4.3.7 Posibles errores en la configuración NFS

**El montaje NFS desde los nodos Moodle falla con `access denied` o `permission denied`**

Verificar que la IP del cliente está incluida en `/etc/exports`, que el servicio NFS está activo y que el firewall permite el tráfico:

```bash
sudo exportfs -v
sudo systemctl status nfs-server
sudo firewall-cmd --list-services
```

**El contenedor Moodle no puede escribir en el directorio montado**

Verificar los permisos del directorio en el servidor NFS:

```bash
ls -la /srv/moodledata
```

Si el propietario no es `nobody` o los permisos no son `777`, corregir:

```bash
chmod -R 777 /srv/moodledata
```

**Error `Stale file handle` en el cliente NFS**

Indica que el servidor NFS fue reiniciado sin que el cliente remontara el volumen. El contenedor Docker debe ser reiniciado para que el volumen NFS sea remontado:

```bash
docker compose down
docker compose up -d
```

---

### 4.4 Despliegue de nodos Moodle (Servidores 4 y 5 - IPs 172.20.208.173 y 172.20.208.169)

Cada nodo Moodle ejecuta una instancia idéntica de la aplicación dentro de un contenedor Docker personalizado. La imagen se construye a partir de un `Dockerfile` que instala PHP 8.1 con Apache y todas las extensiones requeridas por Moodle, configura los parámetros de PHP para un entorno de producción, y clona el código fuente de Moodle directamente desde el repositorio oficial.

Los pasos de esta sección deben repetirse en cada servidor que opere como nodo Moodle.

#### 4.4.1 Preparación del entorno

Instalar NFS utils en el nodo para que Docker pueda montar volúmenes NFS, y habilitar el firewall:

```bash
sudo dnf install nfs-utils -y
sudo systemctl enable firewalld
sudo systemctl start firewalld
```

Crear el directorio de trabajo:

```bash
mkdir moodle
cd moodle
```

#### 4.4.2 Configuración de docker-compose.yml

El bloque `volumes` define un volumen de tipo `local` con opciones NFS. Docker gestiona el montaje automáticamente al iniciar el contenedor, sin necesidad de configurar `/etc/fstab`. La dirección `addr=172.20.208.167` debe corresponder a la IP del Servidor 2 (NFS).

```bash
nano docker-compose.yml
```

```yaml
services:
  moodle:
    build: .
    container_name: moodle-app
    restart: always
    ports:
      - "80:80"
    volumes:
      - moodledata:/var/moodledata

volumes:
  moodledata:
    driver: local
    driver_opts:
      type: "nfs"
      o: "addr=172.20.208.167,rw,nolock,soft"
      device: ":/srv/moodledata"
```

**Notas sobre las opciones NFS del volumen:**

| Opción | Descripción |
|---|---|
| `nolock` | Deshabilita el bloqueo de archivos NFS (NLM). Adecuado para aplicaciones que gestionan sus propios bloqueos. |
| `soft` | Si el servidor NFS no responde, las operaciones retornan error en lugar de bloquear indefinidamente el proceso. Mejora la resiliencia ante fallos del servidor NFS. |

#### 4.4.3 Dockerfile de Moodle

El `Dockerfile` construye la imagen personalizada de Moodle. Cada bloque tiene una justificación técnica:

```bash
nano Dockerfile
```

```dockerfile
FROM php:8.1-apache

# Instalar dependencias del sistema y extensiones PHP requeridas por Moodle.
# Moodle requiere: gd (imágenes), mysqli/pdo_mysql (base de datos), zip (archivos),
# intl (internacionalización), soap (servicios web), opcache (caché de código),
# ldap (autenticación corporativa), redis (sesiones).
RUN apt-get update && apt-get install -y \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libxml2-dev \
    libzip-dev \
    libicu-dev \
    libldap2-dev \
    libonig-dev \
    git \
    unzip \
    curl \
    pkg-config \
    default-libmysqlclient-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) \
        gd \
        mysqli \
        pdo \
        pdo_mysql \
        zip \
        intl \
        soap \
        opcache \
        ldap \
    && pecl install redis \
    && docker-php-ext-enable redis \
    && a2enmod rewrite headers expires \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Configuración de PHP optimizada para Moodle en producción.
# memory_limit elevado para soportar operaciones intensivas (exportación de informes, etc.).
# opcache mejora significativamente el rendimiento al cachear el bytecode compilado de PHP.
# opcache.validate_timestamps=0 deshabilita la revalidación de timestamps en producción
# para maximizar el rendimiento; requiere reiniciar el contenedor tras actualizar el código.
RUN echo "memory_limit = 1024M" > /usr/local/etc/php/conf.d/moodle.ini && \
    echo "upload_max_filesize = 200M" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "post_max_size = 200M" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "max_execution_time = 300" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "max_input_vars = 10000" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "opcache.enable=1" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "opcache.memory_consumption=512" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "opcache.max_accelerated_files=20000" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "opcache.validate_timestamps=0" >> /usr/local/etc/php/conf.d/moodle.ini

# Configuración mínima de Apache para evitar advertencias en los logs.
RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf

WORKDIR /var/www/html

# Clonar Moodle 4.3 (rama estable) desde el repositorio oficial.
# --depth=1 reduce drásticamente el tiempo de descarga al traer solo el último commit.
RUN git clone --depth=1 --branch MOODLE_403_STABLE https://github.com/moodle/moodle.git .

# Copiar la configuración personalizada de Moodle al directorio raíz de la aplicación.
COPY config.php /var/www/html/config.php

# Asignar la propiedad de todos los archivos al usuario de Apache (www-data).
# Sin esto, Moodle no puede escribir en directorios temporales ni leer ciertos archivos.
RUN chown -R www-data:www-data /var/www/html

EXPOSE 80
```

#### 4.4.4 Archivo config.php de Moodle

El archivo `config.php` es el punto central de configuración de Moodle. En esta arquitectura, contiene la configuración de conexión a MariaDB, a Redis para sesiones, y las rutas y URLs del sistema.

```bash
nano config.php
```

```php
<?php
unset($CFG);
global $CFG;
$CFG = new stdClass();

/* =========================
 * Base de Datos
 * ========================= */
$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = '172.20.208.122';   // IP del Servidor 1 (MariaDB)
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodle';
$CFG->dbpass    = 'secret';
$CFG->prefix    = 'mdl_';

$CFG->dboptions = array(
  'dbpersist' => 0,
  'dbsocket'  => 0,
  'dbport'    => 3306,
  'dbcollation' => 'utf8mb4_unicode_ci',
);

/* =========================
 * Configuración General
 * ========================= */
$CFG->wwwroot   = 'http://172.20.208.174';   // IP del balanceador NGINX (Servidor 3)
$CFG->dataroot  = '/var/moodledata';          // Path del volumen NFS dentro del contenedor
$CFG->directorypermissions = 02777;

$CFG->reverseproxy = false;
// Activar las siguientes líneas cuando se implemente HTTPS con dominio:
//$CFG->sslproxy = true;
//$CFG->trustedproxy = ['172.20.208.174'];

/* =========================
 * Redis - Sesiones
 * ========================= */
$CFG->session_handler_class = '\core\session\redis';
$CFG->session_redis_host     = '172.20.208.122';   // IP del Servidor 1 (Redis)
$CFG->session_redis_port     = 6379;
$CFG->session_redis_database = 0;
$CFG->session_redis_prefix   = 'moodle_';
$CFG->session_redis_auth     = 'redispass';

require_once(__DIR__ . '/lib/setup.php');
```

**Aspectos críticos de esta configuración:**

- `$CFG->wwwroot` debe apuntar siempre a la IP o dominio del balanceador NGINX, nunca a la IP de un nodo individual. Moodle usa esta variable para construir todas las URLs de la aplicación.
- `$CFG->dataroot` debe ser accesible con permisos de escritura por el usuario `www-data`. Si el volumen NFS no está correctamente montado, Moodle fallará al intentar escribir en este directorio.
- `$CFG->session_handler_class = '\core\session\redis'` es la directiva que delega la gestión de sesiones a Redis. Sin esta línea, Moodle almacenará sesiones en el sistema de archivos local del contenedor, rompiendo el comportamiento entre nodos.

#### 4.4.5 Despliegue del contenedor

La primera vez se construye la imagen desde el `Dockerfile`. El flag `--no-cache` fuerza la reconstrucción completa, útil para asegurarse de que se obtiene el código más reciente de Moodle desde git.

```bash
docker compose build --no-cache
docker compose up -d
```

#### 4.4.6 Restricción de acceso por firewall

Los nodos Moodle no deben ser accesibles directamente desde Internet o desde la red general. Solo NGINX debe poder enviarles tráfico en el puerto 80. Esta restricción refuerza la seguridad y garantiza que el tráfico pase siempre por el balanceador.

```bash
# Permitir tráfico HTTP solo desde el balanceador NGINX (172.20.208.174)
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="172.20.208.174" port port="80" protocol="tcp" accept'

# Eliminar la regla general que permite HTTP desde cualquier origen
firewall-cmd --permanent --zone=public --remove-service=http

# Aplicar los cambios
firewall-cmd --reload

# Verificar el estado de las reglas
firewall-cmd --list-all
```

#### 4.4.7 Posibles errores en los nodos Moodle

**El contenedor no puede montar el volumen NFS al iniciar**

```
driver failed programming external connectivity: Error starting userland proxy
```

O bien:

```
VolumeDriver.Mount: exit status 32
```

Verificar que el paquete `nfs-utils` está instalado en el host (no en el contenedor), que el servidor NFS está activo y que el firewall del Servidor 2 permite el tráfico NFS desde la IP del nodo.

**Moodle muestra error de base de datos al acceder por primera vez**

Verificar que la base de datos `moodle` existe en MariaDB y que el usuario `moodle` tiene permisos de acceso desde la IP del nodo:

```bash
docker exec -it moodle_db mysql -u root -prootpass -e "SHOW GRANTS FOR 'moodle'@'%';"
```

**La sesión se pierde al cambiar de nodo (el usuario es desconectado)**

Indica que Redis no está siendo usado como almacén de sesiones. Verificar que `config.php` contiene la directiva `session_handler_class` correcta y que el contenedor puede alcanzar Redis:

```bash
docker exec -it moodle-app php -r "
\$r = new Redis();
\$r->connect('172.20.208.122', 6379);
\$r->auth('redispass');
echo \$r->ping() . PHP_EOL;
"
```

La salida debe ser `+PONG`.

**Error `The wwwroot is misconfigured` en Moodle**

Ocurre cuando la URL con la que se accede al sistema no coincide con `$CFG->wwwroot`. Verificar que los usuarios acceden a través de la IP del balanceador y que `wwwroot` apunta a esa misma IP.

---

### 4.5 Configuración del balanceador NGINX (Servidor 3 - IP 172.20.208.174)

NGINX es el punto de entrada de toda la arquitectura. Su configuración determina el algoritmo de balanceo, el comportamiento ante fallos de backend, los tiempos de espera y el formato de los logs.

#### 4.5.1 Estructura de archivos

```bash
mkdir nginxMoodle
cd nginxMoodle
```

#### 4.5.2 Configuración de docker-compose.yml

Se publican dos puertos: el 80 para el tráfico de usuarios y el 8080 como puerto de administración o monitoreo.

```bash
nano docker-compose.yml
```

```yaml
services:
  nginx:
    image: nginx:stable
    container_name: moodle_lb
    restart: always
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

La opción `:ro` monta el archivo de configuración en modo solo lectura dentro del contenedor, lo que previene modificaciones accidentales desde dentro del contenedor.

#### 4.5.3 Archivo nginx.conf

```bash
nano nginx.conf
```

```nginx
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    # Formato de log extendido que registra a qué backend fue enviada cada petición.
    # upstream_addr es especialmente útil para verificar que el balanceo funciona correctamente
    # y para diagnosticar problemas de acceso a backends específicos.
    log_format upstreamlog '$remote_addr - $remote_user [$time_local] '
                           '"$request" $status $body_bytes_sent '
                           '"$http_referer" "$http_user_agent" '
                           'upstream: $upstream_addr';

    upstream moodle_backend {
        # least_conn: envía cada nueva conexión al servidor con menos conexiones activas.
        # Recomendado para Moodle por la variabilidad en el tiempo de procesamiento de peticiones.
        least_conn;

        # max_fails=3: si un backend falla 3 veces en la ventana de fail_timeout, se marca como no disponible.
        # fail_timeout=30s: ventana de tiempo para contabilizar los fallos. Pasados 30s, NGINX
        # intentará de nuevo enviar tráfico al backend para verificar si se ha recuperado.
        server 172.20.208.173:80 max_fails=3 fail_timeout=30s;
        server 172.20.208.169:80 max_fails=3 fail_timeout=30s;

        # keepalive: mantiene hasta 32 conexiones persistentes con cada backend,
        # reduciendo la sobrecarga de establecer nuevas conexiones TCP para cada petición.
        keepalive 32;
    }

    server {
        listen 80;
        server_name 172.20.208.174;

        access_log /var/log/nginx/access.log upstreamlog;

        location / {
            proxy_pass         http://moodle_backend;
            proxy_http_version 1.1;

            # Cabeceras necesarias para que Moodle identifique correctamente
            # la IP real del cliente y el protocolo usado.
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host  $host;
            proxy_set_header X-Forwarded-Server $host;

            # Tiempos de espera elevados para soportar operaciones largas de Moodle
            # como importación/exportación de cursos, generación de informes, etc.
            proxy_connect_timeout 300;
            proxy_send_timeout    300;
            proxy_read_timeout    300;

            # proxy_buffering off: desactiva el buffer de respuesta en NGINX.
            # Recomendado para Moodle para reducir la latencia percibida en respuestas largas.
            proxy_buffering off;
            proxy_redirect  off;
        }
    }
}
```

#### 4.5.4 Despliegue del contenedor

```bash
docker compose up -d
```

Verificar que el contenedor está activo:

```bash
docker ps
docker logs moodle_lb
```

Los logs no deben contener errores de sintaxis en la configuración de NGINX. Un inicio exitoso muestra únicamente el proceso iniciado sin mensajes de error.

#### 4.5.5 Posibles errores en NGINX

**Error de sintaxis en la configuración**

```
nginx: [emerg] invalid parameter "least_conn;" in /etc/nginx/nginx.conf
```

Verificar la sintaxis del archivo de configuración antes de (re)iniciar el contenedor:

```bash
docker run --rm -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro nginx:stable nginx -t
```

**NGINX arranca pero los backends no responden (502 Bad Gateway)**

Verificar que los nodos Moodle están activos y accesibles desde el Servidor 3:

```bash
curl -I http://172.20.208.173
curl -I http://172.20.208.169
```

También verificar que las reglas de firewall de los nodos permiten tráfico desde 172.20.208.174.

**Las peticiones siempre van al mismo nodo**

Si se usa `ip_hash`, todas las peticiones desde la misma IP irán al mismo nodo. Verificar que se está usando `least_conn` o el método Round Robin predeterminado. Confirmar el balanceo revisando los logs:

```bash
docker exec moodle_lb tail -f /var/log/nginx/access.log
```

La columna `upstream:` debe alternar entre las IPs de los dos nodos bajo carga.

**Error al montar el archivo de configuración: `no such file or directory`**

El archivo `nginx.conf` debe existir en el directorio desde el que se ejecuta `docker compose up`. Verificar la ruta:

```bash
ls -la nginx.conf
```

---

## 5. Verificación del sistema completo

### 5.1 Comprobación de conectividad entre servicios

Antes de realizar la instalación web de Moodle, verificar que todos los componentes se comunican correctamente.

Desde cualquier nodo Moodle (Servidor 4 o 5), verificar:

```bash
# Conectividad con MariaDB
nc -zv 172.20.208.122 3306

# Conectividad con Redis
nc -zv 172.20.208.122 6379

# Montaje del volumen NFS (verificar que el directorio existe y es escribible)
docker exec moodle-app ls -la /var/moodledata
docker exec moodle-app touch /var/moodledata/test_write && echo "NFS escribible"
```

### 5.2 Validación del balanceo de carga

Acceder a la URL del balanceador desde un navegador:

```
http://172.20.208.174
```

Moodle debe presentar el asistente de instalación (en la primera ejecución) o la página de acceso.

Para confirmar que el balanceo distribuye las peticiones entre ambos nodos, monitorear los logs de NGINX en tiempo real:

```bash
docker exec moodle_lb tail -f /var/log/nginx/access.log
```

Realizar varias peticiones desde el navegador. La columna `upstream:` debe mostrar ambas IPs (`.173` y `.169`) en distintas peticiones.

### 5.3 Prueba de failover

Para verificar el comportamiento ante la caída de un nodo:

```bash
# En el Servidor 4 (Nodo 1), detener el contenedor
docker compose stop

# Desde el navegador, verificar que Moodle sigue respondiendo a través del balanceador
# (puede tardar hasta fail_timeout=30s en detectar el fallo la primera vez)

# Revisar los logs de NGINX para confirmar que el tráfico fue redirigido al Nodo 2
docker exec moodle_lb tail -20 /var/log/nginx/access.log

# Restaurar el Nodo 1
docker compose start
```

NGINX detectará automáticamente la recuperación del nodo y comenzará a enviarle tráfico de nuevo dentro del siguiente ciclo de `fail_timeout`.

---

## 6. Conclusión

La arquitectura implementada resuelve los problemas fundamentales de una plataforma educativa en producción: disponibilidad, rendimiento bajo carga y consistencia de datos en un entorno distribuido.

La separación de responsabilidades entre componentes (NGINX para enrutamiento, MariaDB para persistencia relacional, Redis para sesiones, NFS para archivos, Docker para aislamiento y portabilidad) permite que cada elemento sea escalado, actualizado o reemplazado de forma independiente sin afectar al conjunto del sistema.

El uso exclusivo de tecnologías open source de alta madurez garantiza la sostenibilidad del proyecto a largo plazo, sin dependencias de licencias comerciales y con una comunidad activa de soporte.

Como pasos de evolución futura se recomiendan:

- Implementación de HTTPS mediante certificados TLS con Let's Encrypt o una PKI interna, activando `$CFG->sslproxy` en `config.php`.
- Configuración de un sistema centralizado de logs (ELK Stack o Loki/Grafana) para monitoreo de todos los componentes.
- Implementación de alta disponibilidad también en el nivel de base de datos, mediante replicación MariaDB o Galera Cluster.
- Automatización del despliegue mediante herramientas de infraestructura como código (Ansible, Terraform).
