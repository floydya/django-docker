# This is a tutorial on how to run Django with Docker

Includes:
- django application
- celery 
- best practices for development 
    - faster-dev (volumes)
    - debugging
- Dockerfile best practices
- some production best practices
    - do not run as root
    - use gunicorn and serve static content


# Workshop

1. Setup VirtualEnv and install Django
    ```
    virtualenv -p $(which python3) venv
    . ./venv/bin/activate
    pip install "Django>=3.0,<4"
    ```
1. Create django project and initial setup
    ```
    django-admin startproject django_demo .
    python django_demo/manage.py makemigrations
    python django_demo/manage.py migrate
    python django_demo/manage.py createsuperuser --username admin --email ""
    ```
    You may test it with `python django_demo/manage.py runserver 0.0.0.0:8000` and open the browser at http://120.0.0.1:8000
1. Setup a first draft of a Dockerfile, so we can see what is bad in this draft
    ```Dockerfile
    FROM python
    COPY django_demo/ /app/
    RUN pip install --no-cache-dir "Django>=3.0,<4"
    ENV PYTHONUNBUFFERED=1
    CMD python /app/manage.py runserver 0.0.0.0:8000
    ```
    and test it
    ```
    docker build -t django-docker-demo .
    docker run -it --rm -p8000:8000 django-docker-demo
    ``` 

    build for the first time: 2:18<br>
    build after minor change: 0:22

    problems:
    * COPY files before 
    * use docker images tags (alpine if possible)
    * beware of context (.dockerignore)

1. Fix problems in Dockerfile
    ```Dockerfile
    FROM python:3.7-alpine
    RUN pip install --no-cache-dir "Django>=3.0,<4"
    COPY django_demo/ /app/
    ENV PYTHONUNBUFFERED=1
    CMD python /app/manage.py runserver 0.0.0.0:8000
    ```
    and `.dockerignore`
    ```
    venv/
    __pycache__/
    db.sqlite3
    ```
    From `Sending build context to Docker daemon  42.42MB` to `...156.7kB`

    build for the first time: 1:37<br>
    build after minor change: 0:03
1. Let's create our own Django content
    create `django_demo/django_demo/views.py`
    ```python
    import os
    from django.http import HttpResponse

    def hello_world(request):
        output_string = "<h1>Hello People from {}".format(os.environ.get("HOSTNAME", "no_host"))
        return HttpResponse(output_string)
    ```
    change `django_demo/django_demo/urls.py`
    ```python
    from django.contrib import admin
    from django.urls import path
    from . import views

    urlpatterns = [
        path('admin/', admin.site.urls),
        path('', views.hello_world),
    ]
    ```

1. Dispite the fast building time, I don't want to rebuild+restart every time I change something
    add volume on docker run
    ```
    docker run -it --rm -p8000:8000 -v "$(pwd)/django_demo/:/app/" django-docker-demo
    ``` 

    now, every time you change some file, django wil reload itself. Very useful for developmemt.
1. Let's add some more dependencies: Celery (**This is a major step**)

    We are going to use Redis as a message broker, for simplicity

    - Install Celery 
        (as we are getting more dependencies, let's keep a `requirements.txt`)
        ```Dockerfile
        Django>=3.0,<4
        Celery>=4.3.0,<4.4
        redis>=3.3<3.4
        ```
        and update `Dockerfile` (now we don't need to update this with every dependency change)
        ```
        FROM python:3.7-alpine

        COPY django_demo/requirements.txt /app/requirements.txt
        RUN pip install --no-cache-dir -r /app/requirements.txt
        COPY django_demo/ /app/

        ENV PYTHONUNBUFFERED=1
        WORKDIR /app

        CMD python /app/manage.py runserver 0.0.0.0:8000
        ```
    - Create a `docker-compose.yml` so we can keep 3 running services: django + redis + celery_worker
        ```yaml
        version: "3.4"
        services:
            django:
                image: django-docker-demo
                ports:
                    - 8000:8000
                volumes:
                    - ./django_demo/:/app/
            celery-worker:
                image: django-docker-demo
                volumes:
                    - ./django_demo/:/app/
                command: "celery -A django-worker.tasks worker --loglevel=info"
            redis:
                image: redis
        ```
    - Update python code with a test

        create `django_demo/django_demo/tasks.py`
        ```python
        from celery import Celery
        import os
        app = Celery('tasks', broker='redis://redis:6379', backend='redis://redis:6379')

        @app.task
        def hello(caller_host):
            return "Hi {}! This is {}.".format(caller_host, os.environ.get("HOSTNAME", 'celery_worker_hostname'))
        ```
        add to `django_demo/django_demo/views.py`
        ```python
        from . import tasks
        def test_task(request):
            task = tasks.hello.delay(os.environ.get("HOSTNAME", "no_host"))
            output_string = task.get(timeout=5)
            return HttpResponse(output_string)
        ```
        update `django_demo/django_demo/urls.py`
        ```python
        urlpatterns = [
            path('admin/', admin.site.urls),
            path('', views.hello_world),
            path('task', views.test_task),
        ]
        ```
    - Finally, we can test our setup

        - run `docker-compose up` (you can see the logs and terminate with ctrl+c. to run in background, add `-d`)
        - open http://127.0.0.1:8000/task


    Now that things got a little confusing, it gets worse.

1. What do you need to develop (not an exaustive list)
    
    | tool | what for | example |
    | --- | --- | --- |
    | a good IDE | auto-completion, debugging  | vscode |
    | good IDE plugins | framework specifics (django) | `ms-python.python`, `batisteo.vscode-django` |
    | linter | you need to have a real-time feedback about what is wrong | mypy |
    | formatter | it's great for code sharing | black |
    | unit tests | you should really not test your code. Make your computer do it | pytest |


    **note**: keep a `requirements-dev.txt` with those development packages (`pip freeze` may help)
    so you can install them without impact the docker image.

    **tip**: for automatic linting and syntax checking, edit `.vscode/settings.json`:
    ```
    {
        "python.pythonPath": "venv/bin/python3.7",
        "editor.formatOnSave": true,
        "python.formatting.provider": "black",
        "python.linting.pylintEnabled": false,
        "python.linting.mypyEnabled": true,
        "python.linting.enabled": true
    }
    ```

1. Debugging a Django application

    example with vscode, which uses ptvsd for debugging, `.vscode/launch.json`
    ```json
    { 
        "version": "0.2.0",
        "configurations": [
            {
                "name": "Python: Current File",
                "type": "python",
                "request": "launch",
                "program": "${file}",
            }, 
            {
                "name": "Python: Debug Django",
                "type": "python",
                "request": "launch",
                "program": "${workspaceFolder}/django_demo/manage.py",
                "args": [
                    "runserver",
                    "--nothreading"
                ],
                "subProcess": true,
            }
        ]
    }
    ```

    The first one is good enough for python scripts, but not so nice for django applications.
    The second one is very good, but it runs locally (no docker)

    or...

    - You could be running a debugger with docker

        - change the `Dockerfile` in order to include `requirements-dev.txt` instead of `requirements.txt` (we need development tools for debugging) and some other dependencies
            ```dockerfile
            FROM python:3.7-alpine

            RUN apk add --update --no-cache \
                    bash \
                    build-base

            COPY django_demo/requirements-dev.txt /app/requirements.txt
            RUN pip install --no-cache-dir -r /app/requirements.txt
            COPY django_demo/ /app/

            ENV PYTHONUNBUFFERED=1
            WORKDIR /app

            CMD python /app/manage.py runserver 0.0.0.0:8000
            ```
            and build it again 
            ```
            docker build -t django-docker-demo .
            ```
        - expose port 5678 on django service inside `docker-compose.yml`
            ```yaml
            version: "3.4"
            services:
                django:
                    image: django-docker-demo:latest
                    ports:
                        - "8000:8000"
                        - "5678:5678"
                    volumes:
                        - "./django_demo/:/app/"
                celery-worker:
                    image: django-docker-demo:latest
                    volumes:
                        - "./django_demo/:/app/"
                    command: "celery -A django_demo.tasks worker --loglevel=info"
                redis:
                    image: redis:5.0-alpine
            ```
        - modify `django_demo/manage.py` (add it inside main, before `execute_from_command_line(sys.argv)`)
            ```python
            from django.conf import settings
            if settings.DEBUG:
                if (  # as reload relauches itself, workaround for it
                    "--noreload" not in sys.argv
                    and os.environ.get("INTERNAL_FLAG", "nop") == "nop"
                ):
                    os.environ["INTERNAL_FLAG"] = "yes"
                else:
                    import ptvsd

                    ptvsd.enable_attach()
            ```
        - add a remote debugger on your IDE. For vscode add a configuration to `.vscode/launch.json`
            ```json
            {
                "name": "Python: Debug Django attach Docker",
                "type": "python",
                "request": "attach",
                "localRoot": "${workspaceFolder}/django_demo",
                "remoteRoot": "/app",
                "host": "127.0.0.1",
                "port": 5678,
            },
            ```
        - and test it
            ```
            docker-compose up
            ```
            After adding a breakpoint inside `django_demo.views.hello_world()` reload your browser.

1. Improve `Dockerfile`

    - do not run as root.
        ```Dockerfile
        RUN adduser -D user
        USER user
        ```
    - remove unnecessary dependencies. Keep different images for different uses (use multi-stage builds)
        ```Dockerfile
        FROM python:3.7-alpine AS base
        ...
        FROM base AS dev
        ...
        FROM base AS final
        ...
        ```
    - clean your logs
        ```yaml
        services:
            app:
                ...
                logging:
                    options:
                        max-size: "10m"
                        max-file: "3"
        ```
    - next level caching (with Buildkit `DOCKER_BUILDKIT=1`)
        ```Dockerfile
        # syntax=docker/dockerfile:experimental
        ...
        ENV HOME /app
        WORKDIR /app
        RUN --mount=type=cache,uid=0,target=/app/.cache/pip,from=base \
            pip install -r requirements.txt
        ```
        **note**: `# syntax=docker/dockerfile:experimental` on the first line of your `Dockerfile` is mandatory for using BUILDKIT new features
    - quality as part of the pipeline
        ```Dockerfile
        FROM dev AS qa
        RUN black --target-version=py37 --check --diff . 
        RUN mypy --python-version=3.7 --pretty --show-error-context .
        RUN coverage run django_demo/manage.py test
        RUN django_demo/manage.py check --deploy --fail-level WARNING
        ```

    - Prepare Django for production
        - Prepare Django for production - https://docs.djangoproject.com/en/3.0/howto/deployment/checklist/ (out of the scope for this)
        - Use a decent webserver <br>
            > from: https://docs.djangoproject.com/en/3.0/ref/django-admin/#runserver <br>
            DO NOT USE THIS SERVER IN A PRODUCTION SETTING. It has not gone through security audits or performance tests. 
            (And that’s how it’s gonna stay. We’re in the business of making Web frameworks, not Web servers, 
            so improving this server to be able to handle a production environment is outside the scope of Django.)
        - build static files (you will need it for the new webserver)
            ```Dockerfile
            FROM dev AS staticfiles
            RUN python manage.py collectstatic --noinput

            FROM nginx:1.17-alpine
            COPY conf/nginx.conf /etc/nginx/conf.d/default.conf
            COPY --from=staticfiles /app/static /staticfiles/static
            ```
            and create the corresponding `conf/nginx.conf` (just a minimal example. Don't use in production)
            ```
            server {
                listen 80 default_server;
                server_name _;

                location /static {
                    root /staticfiles;
                }

                location / {
                    proxy_pass http://django:8000;
                }
            }
            ```
     
        - build a common entrypoint (so you don't have to change dockerfile later)
            ```bash
            #!/bin/sh
            python manage.py migrate

            case $DJANGO_DEBUG in
                true|True|TRUE|1)
                    echo "================= Starting debugger =================="
                    python manage.py runserver 0.0.0.0:8000
                    ;;
                *)
                    gunicorn --worker-class gevent -b 0.0.0.0:8000 django_demo.wsgi
                    ;;
            esac
            ```
        - you probably want to use a different database

1. Update your docker-compose and run
    `docker-compose.yml`
    ```
    version: "3.4"
    services:
        webserver:
            image: django-docker-demo:webserver
            build:
                dockerfile: Dockerfile
                target: webserver
                context: .
            ports:
                - 8080:80
        django:
            image: django-docker-demo:dev
            build:
                dockerfile: Dockerfile
                target: dev
                context: .
            environment:
                - DJANGO_DEBUG=yes
            ports:
                - 8000:8000
                - 5678:5678
            volumes:
                - ./django_demo/:/app/
        celery-worker:
            image: django-docker-demo:latest
            build:
                dockerfile: Dockerfile
                context: .
            volumes:
                - ./django_demo/:/app/
            command: "celery -A django_demo.tasks worker --loglevel=info"
        redis:
            image: redis:5.0-alpine
    ```