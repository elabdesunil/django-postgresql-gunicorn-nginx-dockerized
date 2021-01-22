# Dockerizing Django with Postgres, Gunicorn, and Nginx

github: https://github.com/sunilale0/django-postgresql-gunicorn-nginx-dockerized.git

- [Dockerizing Django with Postgres, Gunicorn, and Nginx](#dockerizing-django-with-postgres-gunicorn-and-nginx)
  - [Introduction](#introduction)
  - [Project Setup](#project-setup)
  - [Docker](#docker)
  - [Postgres](#postgres)
      - [Get the following error?](#get-the-following-error)
      - [after solving error](#after-solving-error)
    - [Notes](#notes)
  - [Gunicorn](#gunicorn)

## Introduction

This is a step-by-step tutorial that details how to configure Django to run on Docker with Postgres. For production environments, Nginx and Gunicorn will be added. We'll also take a look at how to serve Django static and media files via Nginx.

Dependencies:

1. Django v3.0.7
2. Docker v19.03.8
3. Python v3.8.3

## Project Setup

Create a new project directory along with a new Django project:

```cmd
mkdir django-on-docker && cd django-on-docker
mkdir app && cd app
python3.8 -m venv env
source env/bin/activate

pip install django==3.0.7
django-admin.py startproject hello_django .
python manage.py migrate
python manage.py runserver
```

Navigate to http://localhost:8000/ to view the Django welcome screen. Kill the server and exit from the virtual environment once done. We now have a simple Django project to work with.

Create a `requirements.txt` file in the `app` directory and add Django as a dependency:

```
Django==3.0.7
```

Since we'll be moving to Postgres, go ahead and remove the `db.sqlite3` file from the `app` directory.

Your project directory should look like:

```
└── app
    ├── hello_django
    │   ├── __init__.py
    │   ├── asgi.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    ├── manage.py
    └── requirements.txt
```

## Docker

Add a `Dockerfile` to the `app` directory

```Dockerfile
# pull official base image
FROM python:3.8.3-alpine

# set work directory
WORKDIR /usr/src/app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN pip install -r requirements.txt

# copy project
COPY . .
```

So, we started with an Alpine-based Docker image for Python 3.8.3. We then set a working directory with two environment variables:

1. `ENV PYTHONDONTWRITEBYTECODE`: Prevents python from writing pyc files to disc (equivalent to `python -B` [option](https://docs.python.org/3/using/cmdline.html#id1))
2. `PYTHONUNBUFFERED`: Prevents Python from buffering stdout and stderr (equivalent to `python -u` option)

Finally, we updated Pip, copied over the `requirements.txt` file, installed the dependencies, and copied over the Django project itself.

Review Docker for Python Developers [here](https://mherman.org/presentations/dockercon-2018) for more on structuring Dockerfiles as well as some best practices for configuring Docker for Python-based development.

Next, add a `docker-compose.yml` file to the project root:

```Dockerfile
version: '3.7'

services:
    web:
        build: ./app
        command: python manage.py runserver 0.0.0.0:8000
        volumes:
            - ./app/:/usr/src/app
        ports:
            - 8000:8000
        env_file:
            - ./.env.dev
```

Review the compose file [reference](https://docs.docker.com/compose/compose-file/) for info on how this file works.

Update the `SECRET_KEY`, `DEBUG`, AND `ALLOWED_HOSTS` variables in `settings.py`:

```python
SECRET_KEY = os.environ.get("SECRET_KEY")

DEBUG = int(os.environ.get("DEBUG", default=0))

# 'DJANGO_ALLOWED_HOSTS' should be a single string of hosts with a space between each.
# For example: 'DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]'
ALLOWED_HOSTS = os.environ.get("DJANGO_ALLOWED_HOSTS").split(" ")
```

Then, create a `.env.dev` file in the project root to store environment variables for development:

```
DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS= localhost 127.0.0.1 [::1]
```

Build the image:

```
docker-compose build
```

Once the image is build, run the container:

```
docker-compose up -d
```

Navigate to http://localhost:8000/ to again view the welcome screen.

Check for errors in the logs if this doesn't work via `docker-compose logs -f`.

## Postgres

To configure Postgres, we'll need to add a new service to the `docker-compose.yml` file, update the Django settings, and install `Psycopg2`.

First, add a new service called `db` to `docker-compose.yml`:

```Dockerfile
version: '3.7'

services:
    web:
        build: ./app
        command: python manage.py runserver 0.0.0.0:8000
        volumes:
            - ./app/:/usr/src/app
        ports:
            - 8000:8000
        env_file:
            - ./.env.dev
        depends_on:
            - db
    db:
        image: postgres:12.0-alpine
        volumes:
            - postgres_data:/var/lib/postgresql/data/
        environment:
            - POSTGRES_USER=hello_django
            - POSTGRES_PASSWORD=hello_django
            - POSTGRES_DB=hello_django_dev

volumes:
    postgres_data:
```

To persist the data beyond the life of the container we configured a volume. This config will bind `postgres_data` to the "/var/lib/postgresql/data/" directory in the container.

We also added an environment key to define a name for the default database and set a username and password.

Review the "Environment Variables" section of the Postgres Docker Hub [page](https://hub.docker.com/_/postgres) for more info.

We'll also need some new environment variables for the `web` service as well, so update `.env.dev` like so:

```
DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_dev
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
```

Update the `DATABASES` dict in `settings.py`:

```python
DATABASES = {
    "default": {
        "ENGINE": os.environ.get("SQL_ENGINE", "django.db.backends.sqlite3"),
        "NAME": os.environ.get("SQL_DATABASE", os.path.join(BASE_DIR, "db.sqlite3")),
        "USER": os.environ.get("SQL_USER", "user"),
        "PASSWORD": os.environ.get("SQL_PASSWORD", "password"),
        "HOST": os.environ.get("SQL_HOST", "localhost"),
        "PORT": os.environ.get("SQL_PORT", "5432"),
    }
}
```

Here, the database is configured based on the environment variables that we just defined. Take note of the default values.

Update the Dockerfile to install the appropriate packages required for Psycopg2:

```Dockerfile
# pull official base image
FROM python:3.8.3-alpine

# set work directory
WORKDIR /usr/src/app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN apk update \
    && apk add postgresql-dev gcc python3-dev musl-dev

# install dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN pip install -r requirements.txt

# copy project
COPY . .
```

Add Psycopg2 to `requirements.txt`:

```
Django==3.0.7
psycopg2-binary==2.8.5
```

Review [this](https://github.com/psycopg/psycopg2/issues/684) GitHub issue for more more info on installing Psycopg2 in an Alpine-based Docker Image.

Build the new image and spin up the two containers:

```
docker-compose up -d --build
```

Run the migrations:

```
docker-compose exec web python manage.py migrate --noinput
```

#### Get the following error?

```
django.db.utils.OperationalError: FATAL: database "hello_django_dev" does not exist
```

Run `docker-compose down -v` to remove the volumes along with the containers. Then, re-build the images, run the containers, and apply the migrations.

#### after solving error

Ensure the default Django tables are created:

```
$ docker-compose exec db psql --username=hello_django --dbname=hello_django_dev

psql (12.0)
Type "help" for help.

hello_django_dev=# \l
                                          List of databases
       Name       |    Owner     | Encoding |  Collate   |   Ctype    |       Access privileges
------------------+--------------+----------+------------+------------+-------------------------------
 hello_django_dev | hello_django | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres         | hello_django | UTF8     | en_US.utf8 | en_US.utf8 |
 template0        | hello_django | UTF8     | en_US.utf8 | en_US.utf8 | =c/hello_django              +
                  |              |          |            |            | hello_django=CTc/hello_django
 template1        | hello_django | UTF8     | en_US.utf8 | en_US.utf8 | =c/hello_django              +
                  |              |          |            |            | hello_django=CTc/hello_django
(4 rows)

hello_django_dev=# \c hello_django_dev
You are now connected to database "hello_django_dev" as user "hello_django".

hello_django_dev=# \dt
                     List of relations
 Schema |            Name            | Type  |    Owner
--------+----------------------------+-------+--------------
 public | auth_group                 | table | hello_django
 public | auth_group_permissions     | table | hello_django
 public | auth_permission            | table | hello_django
 public | auth_user                  | table | hello_django
 public | auth_user_groups           | table | hello_django
 public | auth_user_user_permissions | table | hello_django
 public | django_admin_log           | table | hello_django
 public | django_content_type        | table | hello_django
 public | django_migrations          | table | hello_django
 public | django_session             | table | hello_django
(10 rows)

hello_django_dev=# \q
```

You can check that the volume was created as well by running:

```
docker volume inspect django-on-docker_postgres_data

[
    {
        "CreatedAt": "2020-06-13T18:43:56Z",
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "django-on-docker",
            "com.docker.compose.version": "1.25.4",
            "com.docker.compose.volume": "postgres_data"
        },
        "Mountpoint": "/var/lib/docker/volumes/django-on-docker_postgres_data/_data",
        "Name": "django-on-docker_postgres_data",
        "Options": null,
        "Scope": "local"
    }
]
```

Next, add an `entrypoint.sh` file to the "app" directory to verify that Postgres is healthy before applying the migrations and running the Django development server:

```shell
#!/bin/sh

if ["$DATABASE" = "postgres"]
then
    echo "Waiting for PostgreSQL..."

    while ! nc -z $SQL_HOST $SQL_PORT; do
        sleep 0.1
    done

    echo "PostgreSQL started"
fi

python manage.py flush --no-input
python manage.py migrate

exec "$@"
```

Update the file permissions locally:

```
chomod +x app/entrypoint.sh
```

Then, update the Dockerfile to copy over the `entrypoint.sh` file and run it as the Docker [entrypoint](https://docs.docker.com/engine/reference/builder/#entrypoint) command:

```Dockerfile
# pull official base image
FROM python:3.8.3-alpine

# set work directory
WORKDIR /usr/src/app

# set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# install psycopg2 dependencies
RUN apk update \
    && apk add postgresql-dev gcc python3-dev musl-dev

# install dependencies
RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN pip install -r requirements.txt

# copy entrypoint.sh
COPY ./entrypoint.sh .

# copy project
COPY . .

# run entrypoint.sh
ENTRYPOINT ["/usr/src/app/entrypoint.sh"]
```

Add the `DATABASE` environment variable to `.env.dev`:

```
DEBUG=1
SECRET_KEY=foo
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_dev
SQL_USER=hello_django
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
```

Test it out again:

1. Re-build the images
2. Run the containers
3. Try http://localhost:8000/

Remove the docker-compose including volumes using:

```
docker-compose down -v
```

### Notes

First, despite adding Postgres, we can still create an independent Docker image for Django as long as the `DATABASE` environment variable is not set to `postgres`. To test, build a new image and then run a new container:

```
docker build -f ./app/Dockerfile -t hello_django:latest ./app
docker run -d \
    -p 8006:8000 \
    -e "SECRET_KEY=please_change_me" -e "DEBUG=1" -e "DJANGO_ALLOWED_HOSTS=*" \
    hello_django python /usr/src/app/manage.py runserver 0.0.0.0:8000
```

You should be able to view the welcome page at http://localhost:8006

After done, remove the docker image by using:

```
docker rm -f <container-id>
```

Second, you may want to comment out the database flush and migrate commands in the `entrypoint.sh` script so they don't run on every start or re-start:

```shell
#!/bin/sh

if ["$DATABASE"="postgres"]
then
    echo "Waiting for postgreSQL..."

    while ! nc -z $SQL_HOST $SQL_PORT; do
        sleep 0.1
    done

    echo "PostgreSQL started"
fi

# python manage.py flush --no-input
# python manage.py migrate

exec "$@"
```

Instead, you can run them manually, after the containers spin up, like so:

```
docker-compose exec web python manage.py flush --no-input
docker-compose exec web python manage.py migrate
```

## Gunicorn

Moving along, for production environments, let's add [Gunicorn](https://gunicorn.org/), a production-grade WSGI server, to the requirements file:

```
Django==3.0.7
gunicorn==20.0.4
psycopg2-binary==2.8.5
```

Curious about WSGI and Gunicorn? Review the [WSGI](https://testdriven.io/courses/python-web-framework/wsgi/) chapter for the Building your own web framework course [here](https://testdriven.io/courses/python-web-framework/);

Since we still want to use Django's built-in server in development, create a new compose file called `docker-compose.prod.yml` for production:

```
version: '3.7'

services:
    web:
        build: ./app
        command: gunicorn hello_django.wsgi:application --bind 0.0.0.0:8000
        ports:
            - 8000:8000
        env_file:
            - ./.env.prod
        depends_on:
            - db
    db:
        image: postgres:12.0-alpine
        volumes:
            - postgres_data:/var/lib/postgresql/data/
        env_file:
            - ./.env.prod.db

volumes:
    postgres_data:
```

> If you have multiple environments, you may want to look at using a `docker-compose.override.yml` configuration file. With this approach, you'd add your base config to a `docker-compose.yml` file and then us a `docker-compose.override.yml` file to override those config settings based on the environment. More on extending a base compose file [here](https://docs.docker.com/compose/extends/).

Take note of the default `command`. We're running Gunicorn rather than the Django development server. We also removed the volume from the `web` service since we don't need it in production. Finally, we're using separate [environment](https://docs.docker.com/compose/env-file/) variable files to define environment variables for both services that will be passed to the container at runtime.

.env.prod:

```
DEBUG=0
SECRET_KEY=change_me
DJANGO_ALLOWED_HOSTS=localhost 127.0.0.1 [::1]
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=hello_django_prod
SQL_PASSWORD=hello_django
SQL_HOST=db
SQL_PORT=5432
DATABASE=postgres
```

.env.prod.db:

```
POSTGRES_USER=hello_django
POSTGRES_PASSWORD=hello_django
POSTGRES_DB=hello_django_prod
```

Add the two files to the project root. You'll probably want to keep them out of version control, so add them to a `.gitignore` file.

Bring down the development containers (and the associated volumes with the `-v` flag):

```
docker-compose down -v
```

Then, build the production images and spin up the containers:

```
docker-compose -f docker-compose.prod.yml up -d --build
```

Verify that the `hello_django_prod` database was created along with the default Django Tables. Test out the admin page at http://localhost:8000/admin. The static files are not being loaded anymore. This is expected since Debug mode is off. We'll fix this shortly.

> How? Remember the commands `docker-compose exec db psql --username=hello_django --dbname=hello_django_prod`, `\L`, `\c hello_django_prod`, `\dt` and `\q`.

> Again, if the container fails to start, check for errors in the logs via `docker-compose -f docker-compose.prod.yml logs -f`.
