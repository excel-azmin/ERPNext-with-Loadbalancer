# ERPNext-with-Loadbalancer

# Install Docker 

```
apt update && apt upgrade -y
apt install curl
curl -fsSL get.docker.com -o get-docker.sh
CHANNEL=stable sh get-docker.sh
rm get-docker.sh
docker swarm init
docker ps
//Post-installation steps for Linux: Manage Docker as a non-root user
sudo groupadd docker
sudo usermod -aG docker $USER
```

# Install Docker Compose 

```
sudo apt-get install docker-compose
```

# Setup Portainer 

```
docker volume create portainer_data
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:lts
docker ps
```

# Set Admin password on portaienr 

* Visit : https://localhost:9443

# Expose Portainer with Loadbalancer 

* `nano /etc/nginx/nginx.conf`

```
 # Upstream Portaienr
    upstream lms_portainer {
        server 192.168.10.103:9443 max_fails=1 fail_timeout=30s;
    }
```
* ` nano /etc/nginx/sites-available/carepronginx.arcapps.org`

```
server {
    server_name lms-portainer.arcapps.org;

    # Let’s Encrypt challenge support
    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/html;
        allow all;
    }

    # Main reverse proxy to Portainer backend (HTTPS 9443)
    location / {
        proxy_pass https://lms_portainer;
        proxy_ssl_verify off;  # Use only if Portainer uses self-signed cert
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/lms-portainer.arcapps.org/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/lms-portainer.arcapps.org/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
```

* test nginx config `nginx -t`
* restart nginx `sudo systemctl restart nginx`
* Access Portainer with the doamin `lms-portainer.arcapps.org`

# Deploy ERPNext

* Create a new stack `carepro-lms`

* update the yml file

```
version: "3.8"

x-customizable-image: &customizable_image
  image: arcapps/lms:${ERP_VERSION?Variable ERP_VERSION not set}
  deploy:
    update_config:
      parallelism: 1
    rollback_config:
      parallelism: 1

x-backend-defaults: &backend_defaults
  <<: [*customizable_image]
  volumes:
    - sites:/home/frappe/frappe-bench/sites
  deploy:
    replicas: 1
    update_config:
      parallelism: 1
    rollback_config:
      parallelism: 1

services:
  db-master:
    image: mariadb:10.6
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD:?No db password set}
      MYSQL_DATABASE: ${DB_NAME:-frappe}
      MYSQL_USER: ${DB_USER:-frappe}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --skip-character-set-client-handshake
      - --skip-innodb-read-only-compressed # Temporary fix for MariaDB 10.6
    volumes:
      - db-data-master:/var/lib/mysql
    networks:
      - main_erp_network
    deploy:
      replicas: 1

  redis-cache:
    image: redis:6.2-alpine
    volumes:
      - redis-cache-data:/data
    networks:
      - main_erp_network
    deploy:
      replicas: 1

  redis-queue:
    image: redis:6.2-alpine
    volumes:
      - redis-queue-data:/data
    networks:
      - main_erp_network
    deploy:
      replicas: 1

  configurator:
    <<: *backend_defaults
    entrypoint:
      - bash
      - -c
    command:
      - >
        ls -1 apps > sites/apps.txt;
        bench set-config -g db_host $$DB_HOST;
        bench set-config -gp db_port $$DB_PORT;
        bench set-config -g redis_cache "redis://$$REDIS_CACHE";
        bench set-config -g redis_queue "redis://$$REDIS_QUEUE";
        bench set-config -g redis_socketio "redis://$$REDIS_QUEUE";
        bench set-config -gp socketio_port $$SOCKETIO_PORT;
    environment:
      DB_HOST: db-master
      DB_PORT: 3306
      REDIS_CACHE: redis-cache:6379
      REDIS_QUEUE: redis-queue:6379
      SOCKETIO_PORT: 9000
    networks:
      - main_erp_network
    deploy:
      replicas: 1
      restart_policy:
        condition: none


  backend:
    <<: *backend_defaults
    networks:
      - main_erp_network
    deploy:
      replicas: 1

  frontend:
    <<: *customizable_image
    command:
      - nginx-entrypoint.sh
    ports:
      - "3890:8080"
    environment:
      BACKEND: backend:8000
      FRAPPE_SITE_NAME_HEADER: lms-web.arcapps.org
      SOCKETIO: websocket:9000
      UPSTREAM_REAL_IP_ADDRESS: ${UPSTREAM_REAL_IP_ADDRESS:-127.0.0.1}
      UPSTREAM_REAL_IP_HEADER: ${UPSTREAM_REAL_IP_HEADER:-X-Forwarded-For}
      UPSTREAM_REAL_IP_RECURSIVE: ${UPSTREAM_REAL_IP_RECURSIVE:-off}
      PROXY_READ_TIMEOUT: ${PROXY_READ_TIMEOUT:-120}
      CLIENT_MAX_BODY_SIZE: ${CLIENT_MAX_BODY_SIZE:-50m}
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:
      - main_erp_network
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
        delay: 5s
     
    depends_on:
      - backend
      - websocket

  websocket:
    <<: [*customizable_image]
    command:
      - node
      - /home/frappe/frappe-bench/apps/frappe/socketio.js
    volumes:
      - sites:/home/frappe/frappe-bench/sites
    networks:
      - main_erp_network
    deploy:
      replicas: 1

  queue-short:
    <<: *backend_defaults
    command: bench worker --queue short,default
    networks:
      - main_erp_network
    deploy:
      replicas: 1

  queue-long:
    <<: *backend_defaults
    command: bench worker --queue long,default,short
    networks:
      - main_erp_network
    deploy:
      replicas: 1

  scheduler:
    <<: *backend_defaults
    command: bench schedule
    networks:
      - main_erp_network
    deploy:
      replicas: 1

volumes:
  sites:
  db-data-master:
  db-data-slave1:
  db-data-slave2:
  redis-cache-data:
  redis-queue-data:

networks:
  main_erp_network:


```

 * Update the `Environment` as per given refrence

![image](https://github.com/user-attachments/assets/6e5f6310-6211-42fa-8e33-553758c623ed)

 * Deploy the stack 

 
# Create Site 

* Access `Backend` from poratiner
![image](https://github.com/user-attachments/assets/cb253840-6081-47e9-9477-be512cc01bf2)

* Create a new site, It will ask for mariadb and administrator password 
```
bench new-site lms-web.arcapps.org
```

* Install LMS

```
bench --site lms-web.arcapps.org install-app lms
bench --site lms-web.arcapps.org install-app excel_lms
```

* Access the apps with local server IP. Example : `http://192.168.10.103:3890/`

![image](https://github.com/user-attachments/assets/ded16c77-9dc3-4053-92a1-f347c15fb991)

# Configure Loadbalancer for LMS

* `nano /etc/nginx/nginx.conf`

```
 # Upstream k8s
    upstream lms_web {
        server 192.168.10.103:3890 max_fails=1 fail_timeout=30s;
    }

```
* ` nano /etc/nginx/sites-available/carepronginx.arcapps.org`

```
server { 	
    server_name lms-web.arcapps.org;
   # Let’s Encrypt challenge support
    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        root /var/www/html;
        allow all;
    }
	
    location / {
        proxy_pass http://lms_web;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host erp.k8s.com; # Force backend to use expected Host
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/lms-web.arcapps.org/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/lms-web.arcapps.org/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}

```

* test nginx config `nginx -t`
* restart nginx `sudo systemctl restart nginx`
* Access LMS with the doamin `lms-web.arcapps.org`

![image](https://github.com/user-attachments/assets/f0399111-ce4f-4588-b29b-2c142b895b88)
  

 



