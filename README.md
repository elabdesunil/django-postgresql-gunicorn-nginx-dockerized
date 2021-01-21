# Dockerizing Django with Postgres, Gunicorn, and Nginx

github: https://github.com/sunilale0/django-postgresql-gunicorn-nginx-dockerized.git

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
