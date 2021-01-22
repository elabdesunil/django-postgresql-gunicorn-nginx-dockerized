# Dockerizing Django with Postgres, Gunicorn, and Nginx

github: https://github.com/sunilale0/django-postgresql-gunicorn-nginx-dockerized.git

- [Dockerizing Django with Postgres, Gunicorn, and Nginx](#dockerizing-django-with-postgres-gunicorn-and-nginx)
  - [Introduction](#introduction)
  - [Project Setup](#project-setup)
  - [Docker](#docker)

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
