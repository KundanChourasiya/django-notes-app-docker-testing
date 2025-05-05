# django-notes-app-docker-testing

![image](https://github.com/user-attachments/assets/b20a3f0a-59ff-4a3f-be90-32798dc61090)


### Dockerfile

```properties
FROM python:3.9

WORKDIR /app/backend

COPY requirements.txt /app/backend
RUN apt-get update \
    && apt-get upgrade -y \
    && apt-get install -y gcc default-libmysqlclient-dev pkg-config \
    && rm -rf /var/lib/apt/lists/*


# Install app dependencies
RUN pip install mysqlclient
RUN pip install --no-cache-dir -r requirements.txt

COPY . /app/backend

EXPOSE 8000
#RUN python manage.py migrate
#RUN python manage.py makemigrations

```

### docker-compose file

```properties
version: "3.8"
services:

  nginx:
    container_name: "nginx_cont"
    build:
      context: ./nginx
    image: nginx
    ports:
      - "80:80"
    networks:
      - django-network
    restart: always
    depends_on:
      - django

  django:
    container_name: "django_cont"
    build:
      context: .
    ports:
      - "8000:8000"
    env_file:
      - ".env"
    depends_on:
      - mysql
    networks:
      - django-network
    command: sh -c "python manage.py migrate --noinput && gunicorn notesapp.wsgi --bind 0.0.0.0:8000"
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8000/admin || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s



  mysql:
    container_name: "db_cont"
    image: mysql
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=test_db
    volumes:
      - ./mysql-django:/var/lib/mysql
    networks:
      - django-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s


volumes:
  mysql-django:

networks:
  django-network:
```

### nginx docker file
```properties
FROM nginx:1.23.3-alpine

COPY ./default.conf /etc/nginx/conf.d/default.conf
```

### nginx/default.conf file
```properties
# conatiner_name
upstream django{
    server django_app:8000;
}

server {
    listen 80;
	#(xyz.com)
    server_name localhost;
    #server_name xyz.com;

    location / {
        proxy_pass http://django_cont:8000;
	proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

