## Self-host, Backup and Restore WordPress

I use WordPress as a CMS backend. One day, no one can access to admin dashboard with the 500 error. I tried all methods found on Google search, but none of these works. So, I had to build a new WP and migrate all data. This blog will share the way I did it.

# Self-host with Docker

+ Create `wordpress` directory and 3 Docker files.

```bash
mkdir wordpress
cd wordpress
touch docker-compose.yml
touch dev.yml
touch prod.yml
```

+ `docker-compose.yml`

We will create 2 containers running WordPress and MariaDB to store data. Note that we use 2 volumes for each container. This makes data persistent for backup and restoration processes.

```yml
services:
  db:
    image: mariadb:10.6.4-focal
    command: "--default-authentication-plugin=mysql_native_password"
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=somewordpress
      - MYSQL_DATABASE=wordpress
      - MYSQL_USER=wordpress
      - MYSQL_PASSWORD=wordpress
    expose:
      - 3306
      - 33060
  wordpress:
    image: wordpress:latest
    volumes:
      - wordpress_data:/var/www/html
    restart: always
    environment:
      - WORDPRESS_DB_HOST=db
      - WORDPRESS_DB_USER=wordpress
      - WORDPRESS_DB_PASSWORD=wordpress
      - WORDPRESS_DB_NAME=wordpress
volumes:
  db_data: {}
  wordpress_data: {}
```

+ `dev.yml` exports dev port, which is 8000, to docker host.

```yaml
services:
  wordpress:
    ports:
      - 8000:80
```

+ `prod.yml` exports prod port, which is 8001, to docker host.

```yaml
services:
  wordpress:
    ports:
      - 8001:80
```

+ Start WordPress

We will start the `dev` WordPress first. After adding some posts, we will try to backup and restore data to the `prod` one.

```bash
docker compose -p dev -f docker-compose.yml -f dev.yml up -d
# ...
# [+] Running 5/5
#  ⠿ Network dev_default         Created
#  ⠿ Volume "dev_wordpress_data" Created
#  ⠿ Volume "dev_db_data"        Created
#  ⠿ Container dev-wordpress-1   Started
#  ⠿ Container dev-db-1          Started

docker ps --format "table {{.Image}}\t{{.Ports}}\t{{.Names}}"
# IMAGE                  PORTS                  NAMES
# wordpress:latest       0.0.0.0:8000->80/tcp   dev-wordpress-1
# mariadb:10.6.4-focal   3306/tcp, 33060/tcp    dev-db-1
```

# Setup and create a post

+ Access to http://localhost:8000/ and setup admin user

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662835274658/xp4G5FnE4.png align="left")

+ Create an example post

![create-post.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662835685738/kXeE_v1tZ.png align="left")

![post.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662836398419/na5Le5SKt.png align="left")

+ Access to http://localhost:8000/ again to see the post

# Backup

The 2 most important things to back up: **content of posts** and **uploaded files**. The posts, configs, users info, etc. are stored in database, so we will dump all data to a `sql` file. And, the uploaded files located in `$WP_ROOT/wp-content/uploads`, so we will copy this directory to a safe place.

## Save database

If you noticed, we cannot access to database from docker host but we can do this for WordPress container. So, we will login into WordPress shell, then backup the database.

```bash
# this is docker host shell
docker exec -it dev-wordpress-1 bash
```

After logging into WordPress shell, we will install `mariadb-client` and use the user info in `docker-compose.yml` to create a database backup.

```bash
# this is docker container shell
apt update
apt install mariadb-client -y

mariadb-dump -h db -u root -psomewordpress wordpress > backup.sql
head -n 3 backup.sql
# -- MariaDB dump 10.19  Distrib 10.5.15-MariaDB, for debian-linux-gnu (aarch64)
# --
# -- Host: db    Database: wordpress

exit
```

We back to the docker host shell and copy `backup.sql` from WordPress container to the docker host.

```bash
# this is docker host shell
docker cp dev-wordpress-1:/var/www/html/backup.sql .
```

## Save uploaded files

```bash
# this is docker host shell
docker cp dev-wordpress-1:/var/www/html/wp-content/uploads .
tree uploads
# uploads
# └── 2022
#     └── 09
#         ├── blog-1-year.png
#         └── ...
```

# Restore

Someday, unrecoverable errors happen, and we have to create a new WordPress with backup data. Start prod WordPress, which is a brand new one running at port 8001.

```bash
docker compose -p prod -f docker-compose.yml -f prod.yml up -d
# [+] Running 5/5
#  ⠿ Network prod_default          Created
#  ⠿ Volume "prod_db_data"         Created
#  ⠿ Volume "prod_wordpress_data"  Created
#  ⠿ Container prod-wordpress-1    Started
#  ⠿ Container prod-db-1           Started

docker ps --format "table {{.Image}}\t{{.Ports}}\t{{.Names}}"
# IMAGE                  PORTS                  NAMES
# wordpress:latest       0.0.0.0:8001->80/tcp   prod-wordpress-1
# mariadb:10.6.4-focal   3306/tcp, 33060/tcp    prod-db-1
# wordpress:latest       0.0.0.0:8000->80/tcp   dev-wordpress-1
# mariadb:10.6.4-focal   3306/tcp, 33060/tcp    dev-db-1
```

+ Change old URL http://locahost:8000/ to new one http://locahost:8001/. Skip this step if you don't change the URL/domain.

![url.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1662838275211/VWymG4uHc.png align="left")

+ Copy `uploads` and `backup.sql` to prod WordPress, then, login to container shell

```bash
# this is docker host shell
docker cp uploads prod-wordpress-1:/var/www/html/wp-content
docker cp backup.sql prod-wordpress-1:/var/www/html
docker exec -it prod-wordpress-1 bash
```

+ Restore database

```bash
# this is docker container shell
apt update
apt install mariadb-client -y
mariadb -h db -u root -psomewordpress wordpress < backup.sql
```

+ Access to http://localhost:8001/ and check the old posts are restored
+ Access to http://localhost:8001/wp-admin and check if you can login with `admin/admin`

*Note: We can backup `plugins`, `themes`, etc. in `wp-content` like `uploads` directory.*

# Clean up

```bash
# dev
docker compose -p dev -f docker-compose.yml -f dev.yml down
docker volume rm dev_wordpress_data dev_db_data

# prod
docker compose -p prod -f docker-compose.yml -f prod.yml down
docker volume rm prod_wordpress_data prod_db_data
```

That's all. Thank you for reading my blog.