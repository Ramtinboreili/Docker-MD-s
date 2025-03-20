# Running WordPress with Docker

## ![WordPress with Docker](../images/wordpress.png)

## Step 1: Pull Required Docker Images
```bash
docker pull wordpress:latest
docker pull nginx:stable-alpine
docker pull mysql:8.0
```

## Step 2: Create a Network
```bash
docker network create --driver bridge wp-net
docker network ls
docker inspect wp-net
```

## Step 3: Create Volumes
```bash
docker volume create --name wp-data
docker volume create --name db-data
docker volume ls
docker inspect wp-data
docker inspect db-data
```

## Step 4: Run MySQL Container
```bash
docker run -d --name mysql --hostname mysql \
  --network=wp-net --restart=unless-stopped \
  --mount=source=db-data,target=/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=strongpassword \
  -e MYSQL_DATABASE=wordpress \
  -e MYSQL_USER=wp_user \
  -e MYSQL_PASSWORD=wp_pass \
  mysql:8.0
```

### Check MySQL Status
```bash
docker ps
docker logs mysql
docker exec -i mysql mysql -u root -pstrongpassword <<< "SHOW DATABASES;"
```

## Step 5: Run WordPress Container
```bash
docker run -d --name wordpress --hostname wordpress \
  --network=wp-net --restart=unless-stopped \
  --mount=source=wp-data,target=/var/www/html/ \
  -e WORDPRESS_DB_HOST=mysql:3306 \
  -e WORDPRESS_DB_USER=wp_user \
  -e WORDPRESS_DB_NAME=wordpress \
  -e WORDPRESS_DB_PASSWORD=wp_pass \
  wordpress:latest
```

### Check WordPress Status
```bash
docker ps
docker logs wordpress
```

## Step 6: Create Nginx Configuration
### Create Required Directories
```bash
mkdir -p ./nginx/conf.d
mkdir -p ./nginx/certs
tree ./nginx
```

### Create Nginx Config File
```bash
cat <<EOF > ./nginx/conf.d/wordpress.conf
server {
  listen 443 ssl;
  server_name wp.example.com;

  ssl_certificate /etc/nginx/certs/cert.pem;
  ssl_certificate_key /etc/nginx/certs/key.pem;

  location / {
    proxy_pass http://wordpress:80;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}

server {
  listen 80;
  server_name wp.example.com;
  return 301 https://$host$request_uri;
}
EOF
```

### Generate SSL Certificate
```bash
CERT_LOCATION=./nginx/certs
openssl req -x509 -nodes -newkey rsa:4096 -days 365 \
  -keyout $CERT_LOCATION/key.pem \
  -out $CERT_LOCATION/cert.pem \
  -subj "/C=US/ST=State/L=City/O=Company/OU=IT/CN=wp.example.com"
```

### Run Nginx Container
```bash
docker run -d --name nginx --hostname nginx \
  --network=wp-net --restart=unless-stopped \
  -v ${PWD}/nginx/conf.d:/etc/nginx/conf.d \
  -v ${PWD}/nginx/certs:/etc/nginx/certs \
  -p 80:80 -p 443:443 \
  nginx:stable-alpine
```

### Check Nginx Status
```bash
docker ps
docker logs nginx
curl -I -L -k http://127.0.0.1
```

## Step 7: Backup WordPress Database
```bash
docker exec -i mysql mysqldump -u root -pstrongpassword --all-databases > full-backup-$(date +%F).sql
ls -lh full-backup-*.sql
du -sh full-backup-*.sql
```

## Step 8: Running WordPress with Docker Compose
### Create `.env` File
```bash
echo "MYSQL_ROOT_PASSWORD=strongpassword" > .env
echo "MYSQL_DATABASE=wordpress" >> .env
echo "MYSQL_USER=wp_user" >> .env
echo "MYSQL_PASSWORD=wp_pass" >> .env
```

### Create `docker-compose.yml`
```yaml
version: '3.8'

networks:
  wp-net:

volumes:
  wp-data:
  db-data:

services:
  db:
    image: mysql:8.0
    container_name: mysql
    volumes:
      - db-data:/var/lib/mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    networks:
      - wp-net

  wordpress:
    image: wordpress:latest
    container_name: wordpress
    volumes:
      - wp-data:/var/www/html/
    depends_on:
      - db
    restart: unless-stopped
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: ${MYSQL_USER}
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}
    ports:
      - 8000:80
    networks:
      - wp-net

  nginx:
    image: nginx:stable-alpine
    container_name: nginx
    restart: unless-stopped
    depends_on:
      - wordpress
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/certs:/etc/nginx/certs
    ports:
      - 80:80
      - 443:443
    networks:
      - wp-net
```

### Start Docker Compose
```bash
docker-compose --env-file .env up -d
```

### Check Docker Compose Logs
```bash
docker-compose logs -f
```

