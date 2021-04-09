---
layout: post
title:  "Run a Flask application as a Docker container"
date:   2021-04-09 18:00:00 +0200
categories: howto web
---
In this post you will learn how to run a Python [Flask](https://flask.palletsprojects.com/en/1.1.x/) application as a [Docker](https://en.wikipedia.org/wiki/Docker_(software)) container.

I have created a [GitHub repository](https://github.com/guidoman/dockerized-flask-app) containing the full source code of this example.

# Prerequisites
First of all you need to know the basics of Web application development with Flask.

You also need to understand __how Docker works__, and the advantages of deploying and running a Flask application as a container.

I also use some additional tools (__pyenv__, __pipenv__ and __git__), that, even if not mandatory, will make your life easier as a Python software developer.

# Setup project environment with __pipenv__
I will use pipenv to create a local directory for my new Flask application. Let's start by creating an empty folder, `cd` to it and run:
```
pyenv install 3.8.6
pyenv local 3.8.6
```

These commands will download and install the specific Python version I'm using in this example (3.8.6), and will create a file named `.python-version` containing string `3.8.6`. Feel free to use any other recent version of Python.

To check if pyenv is configured correctly, run `python --version`. The following output should be printed:
```
Python 3.8.6
```
Now I create a virtual environment and install all the required dependencies, i.e., Python modules needed to run the application. Let's use pipenv to do this. Run in the terminal:
```
pipenv install --python `which python` flask 
```

You should see something like the following:
```
Creating a virtualenv for this project…
Pipfile: /Users/guido/GitHub/dockerized-flask-app/Pipfile
Using /Users/guido/.pyenv/shims/python (3.8.6) to create virtualenv…
...
```

and then:
```
✔ Success!
```

You now have two new files, `Pipfile` and `Pipfile.lock`. Your Pipfile will look like this:
```
[[source]]
name = "pypi"
url = "https://pypi.org/simple"
verify_ssl = true

[dev-packages]

[packages]
flask = "*"

[requires]
python_version = "3.8"
```

`Pipfile.lock` is much longer and more complicated. If you don't know how pipenv works, you can safely ignore it.

# Flask application
Now create [a Hello World Flask application](https://github.com/guidoman/dockerized-flask-app/blob/main/my_flask_app/main.py) inside directory `my_flask_app`.

To execute it in debug mode and check if it's working run:
```
pipenv run python my_flask_app/main.py
```

If you open your browser to address `http://127.0.0.1:5000/` (5000 is the default Flask port), you should see the "Hello, World" text.

# Docker container

## Build the image
In general, to run a Flask application in production, you need an HTTP server like [gunicorn](https://gunicorn.org/). Install it now with pipenv:
```
pipenv install gunicorn
```

To build the Docker image for your container, you need [this Dockerfile](https://github.com/guidoman/dockerized-flask-app/blob/main/Dockerfile).

To actually build the image, run:
```
docker build -t my_flask_app .
```

When the previous command completes without errors, if you execute `docker images`, you should see in the output one line for your new image named `my_flask_app`.


## Dockerfile description
I'll explain here line by line how the Dockerfile works.

```
FROM python:3.8
```

In this example I'm using Python 3.8, so the Docker image will be derived from the default Python image for this version.

```
RUN pip install pipenv
```

Using the pip command included in the default image, I install the pipenv tool.

```
ENV PROJECT_DIR /app
```

The `/app` directory is the destination path where the application source code will be copied. For clarity, I assign this value to an environment variable,  used in the lines below.

```
COPY . /${PROJECT_DIR}
```

This command copies recursively all files from the current directory (the one from where you run `docker build`) to the `/app` directory in the image.

```
WORKDIR ${PROJECT_DIR}
```

This changes the working directory. Every command executed after this line will use `/app` as their working directory.

```
RUN pipenv install --system --deploy
```

This command installs dependencies defined in the Pipfile system-wide. Besides, the `--deploy` parameter forces the installation to fail if the Pipfile.lock is out–of–date, ensuring that the image will contain exactly the dependencies we want.

The last line runs the application with gunicorn using the Docker `CMD` command. I'll explain each element of the command line:

- `gunicorn`: Docker will run a gunicorn process. All arguments after this one will be passed to gunicorn.
- `--graceful-timeout 5`: when stopping the application, this is the time gunicorn has to wait for the application to shut down properly, before terminating it. I've set this timeout to a value shorter than the default Docker equivalent to force that the gunicorn process is always shut down properly by Docker.
- `--chdir my_flask_app`: this changes the working directory. All modules names will be resolved starting from this directory.
- `main:app`: the Flask application is named `app` inside module `main`.
- `-w 4`: gunicorn will run with 4 workers. You should choose the best number of workers for your server hardware.
- `-b 0.0.0.0:8080`: gunicorn will bind on all interfaces on port 8080. For this scenario this is OK.

## Start the container

Run the container:
```
docker run -d -p 8080:8080 --name my_flask_app my_flask_app
```

The container is named `my_flask_app`, the same as your image. You may want to choose a more descriptive name.

Check that your container is running with `docker ps`. You should see a line for your container.

Finally, open [http://localhost:8080/](http://localhost:8080/) in your browser. You will see the same "Hello World!" text as before, but now from a gunicorn server running inside a Docker container!

# Conclusions
I have described in a few steps how to setup a Python Flask project with pyenv, pipenv and how to create a Docker container from it.

Next steps:
- publish your image to a Docker hub (public or private)
- use Kubernetes




