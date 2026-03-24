# Guía técnica de implementación de balanceadores de carga web

## Índice

1. Introducción
2. Fundamentos teóricos
   2.1 Balanceo de carga
   2.2 Alta disponibilidad
   2.3 Escalabilidad horizontal
   2.4 Stateless y estado compartido
   2.5 Algoritmos de balanceo
3. Arquitectura del sistema
   3.1 Descripción general
   3.2 Componentes
4. Implementación
   4.1 Instalación de Docker Engine
   4.2 Base de datos y Redis
   4.3 NFS
   4.4 Nodos Moodle
   4.5 NGINX
5. Conclusión

---

## 1. Introducción

El presente documento describe la implementación de una arquitectura de balanceo de carga web para un Entorno Virtual de Aprendizaje (EVA) basado en Moodle, utilizando tecnologías open source y contenedores Docker.

El objetivo es proporcionar una solución escalable, altamente disponible y tolerante a fallos, que permita distribuir la carga entre múltiples instancias de aplicación, manteniendo consistencia en sesiones, datos y archivos.

---

## 2. Fundamentos teóricos

### 2.1 Balanceo de carga

El balanceo de carga es una técnica que permite distribuir el tráfico de red entre múltiples servidores para mejorar el rendimiento, la disponibilidad y la escalabilidad del sistema.

### 2.2 Alta disponibilidad

Se refiere a la capacidad del sistema de mantenerse operativo ante fallos. En esta arquitectura se logra mediante redundancia de nodos y mecanismos de failover automático.

### 2.3 Escalabilidad horizontal

Consiste en añadir más nodos al sistema para soportar mayor carga sin modificar la arquitectura base.

### 2.4 Stateless y estado compartido

En arquitecturas distribuidas, los nodos deben ser stateless. El estado se externaliza en servicios compartidos:

* Base de datos: persistencia
* Redis: sesiones y caché
* NFS: almacenamiento de archivos

### 2.5 Algoritmos de balanceo

* Round Robin: distribución secuencial
* IP Hash: persistencia por IP
* Least Connections: asigna al nodo con menor carga
* Hash genérico: basado en variables

---

## 3. Arquitectura del sistema

### 3.0 Diagramas de referencia

A continuación se incluyen los diagramas de la arquitectura implementada:

![Diagrama de arquitectura general](img/Gemini_Generated_Image_92g24d92g24d92g2.png)

![Flujo de balanceo de carga](docs/flujo_balanceo.png)

Estos diagramas representan la distribución de componentes, el flujo de tráfico y el comportamiento del balanceador ante diferentes escenarios.

### 3.1 Descripción general

La arquitectura implementada sigue el siguiente flujo:

Usuarios / Internet
│
▼
NGINX (Balanceador de carga)
│
▼
┌───────────────────────────────┐
│         Nodos Moodle          │
│   ┌───────────┐ ┌───────────┐ │
│   │  Nodo 1   │ │  Nodo 2   │ │
│   └───────────┘ └───────────┘ │
└───────────────────────────────┘
│               │
▼               ▼
┌───────┐     ┌────────┐
│ Redis │     │ MariaDB│
└───────┘     └────────┘
\           /
\         /
▼       ▼
┌────────────┐
│ NFS Server │
└────────────┘

### 3.2 Componentes

NGINX:

* Balanceador de carga
* Reverse proxy
* Manejo de failover

Moodle:

* Aplicación web
* Ejecutada en contenedores

Redis:

* Manejo de sesiones
* Caché de aplicación

MariaDB:

* Base de datos central

NFS:

* Almacenamiento compartido

---

## 4. Implementación

### 4.1 Instalación de Docker Engine

* Antes de poder instalar Docker Engine, se debe desinstalar todos los paquetes en conflicto.

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

* Configurar el repositorio

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

* Instalar los paquetes de Docker

```bash
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

* Iniciar Docker Engine.

```bash
sudo systemctl enable --now docker
```

* Verificar instalación

```bash
sudo docker run hello-world
```

* Crear grupo docker

```bash
sudo groupadd docker
```

* Añadir usuario

```bash
sudo usermod -aG docker $USER
```

```bash
newgrp docker
```

---

### 4.2 Base de datos y Redis (Servidor 1)

```bash
mkdir dbRedis-moodle
```

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

```bash
docker compose up -d --build
```

```bash
docker ps
```

---

### 4.3 NFS (Servidor 2)

```bash
sudo dnf install nfs-utils -y
```

```bash
sudo systemctl enable --now nfs-server
sudo systemctl status nfs-server
```

```bash
mkdir -P /srv/moodledata
chown -R nobody:nogroup /srv/moodledata
chmod -R /srv/moodledata
```

```bash
nano /etc/exports
```

```bash
/srv/moodledata 172.20.208.0/24(rw,sync,no_subtree_check,no_root_squash)
/srv/moodledata 172.20.208.173(rw,sync,no_subtree_check,no_root_squash)
/srv/moodledata 172.20.208.169(rw,sync,no_subtree_check,no_root_squash)
```

```bash
sudo exportfs -v
```

```bash
sudo systemctl enable firewalld
sudo systemctl start firewalld
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --reload
nano /etc/selinux/config
SELINUX=permissive
reboot
chmod -R 777 /srv/moodledata
```

---

### 4.4 Nodos Moodle

```bash
sudo dnf install nfs-utils -y
sudo systemctl enable firewalld
sudo systemctl start firewalld
```

```bash
mkdir moodle
```

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

```bash
nano Dockerfile
```

```dockerfile
FROM php:8.1-apache

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

RUN echo "memory_limit = 1024M" > /usr/local/etc/php/conf.d/moodle.ini && \
    echo "upload_max_filesize = 200M" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "post_max_size = 200M" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "max_execution_time = 300" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "max_input_vars = 10000" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "opcache.enable=1" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "opcache.memory_consumption=512" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "opcache.max_accelerated_files=20000" >> /usr/local/etc/php/conf.d/moodle.ini && \
    echo "opcache.validate_timestamps=0" >> /usr/local/etc/php/conf.d/moodle.ini

RUN echo "ServerName localhost" >> /etc/apache2/apache2.conf

WORKDIR /var/www/html

RUN git clone --depth=1 --branch MOODLE_403_STABLE https://github.com/moodle/moodle.git .

COPY config.php /var/www/html/config.php

RUN chown -R www-data:www-data /var/www/html

EXPOSE 80
```

---

### 4.5 NGINX

```bash
mkdir nginxMoodle
```

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

```bash
nano nginx.conf
```

```nginx
worker_processes auto;
events {
    worker_connections 1024;
}
http {
    log_format upstreamlog '$remote_addr - $remote_user [$time_local] '
                          '"$request" $status $body_bytes_sent '
                          '"$http_referer" "$http_user_agent" '
                          'upstream: $upstream_addr';

    upstream moodle_backend {
        least_conn;
        server 172.20.208.173:80 max_fails=3 fail_timeout=30s;
        server 172.20.208.169:80 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    server {
        listen 80;
        server_name 172.20.208.174;

        access_log /var/log/nginx/access.log upstreamlog;

        location / {
            proxy_pass http://moodle_backend;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Server $host;
            proxy_connect_timeout 300;
            proxy_send_timeout 300;
            proxy_read_timeout 300;
            proxy_buffering off;
            proxy_redirect off;
        }
    }
}
```

```bash
docker compose up -d
```

---

## 5. Conclusión

La arquitectura implementada permite desplegar un entorno Moodle altamente disponible, escalable y tolerante a fallos, utilizando tecnologías open source y contenedores. Este enfoque garantiza un uso eficiente de recursos, facilidad de mantenimiento y capacidad de crecimiento según la demanda.
