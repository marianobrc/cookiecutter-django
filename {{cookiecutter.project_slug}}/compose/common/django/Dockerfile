ARG PYTHON_VERSION=3.9-slim-bullseye

{% if cookiecutter.js_task_runner == 'Gulp' -%}
FROM node:16-bullseye-slim as client-builder

ARG APP_HOME=/app
WORKDIR ${APP_HOME}

COPY ./package.json ${APP_HOME}
RUN npm install && npm cache clean --force
COPY . ${APP_HOME}
RUN npm run build

{%- endif %}

# define an alias for the specfic python version used in this file.
FROM python:${PYTHON_VERSION} as python

# Python build stage common to local and production
FROM python as python-build-stage-base

# Install apt packages
RUN apt-get update && apt-get install --no-install-recommends -y \
  # dependencies for building Python packages
  build-essential \
  # psycopg2 dependencies
  libpq-dev

# Requirements are installed here to ensure they will be cached.
COPY ./requirements .

# Python build stage for local
FROM python-build-stage-base as python-build-stage-local
ARG BUILD_ENVIRONMENT=local
# Create Python Dependency and Sub-Dependency Wheels.
RUN pip wheel --wheel-dir /usr/src/app/wheels  \
  -r ${BUILD_ENVIRONMENT}.txt

# Python build stage for production
FROM python-build-stage-base as python-build-stage-production
ARG BUILD_ENVIRONMENT=production
# Create Python Dependency and Sub-Dependency Wheels.
RUN pip wheel --wheel-dir /usr/src/app/wheels  \
  -r ${BUILD_ENVIRONMENT}.txt


# Python 'run' base stage
FROM python as python-run-stage-base

ARG APP_HOME=/app

ENV PYTHONUNBUFFERED 1
ENV PYTHONDONTWRITEBYTECODE 1

WORKDIR ${APP_HOME}

RUN addgroup --system django \
    && adduser --system --ingroup django django


# Install required system dependencies
RUN apt-get update && apt-get install --no-install-recommends -y \
  # psycopg2 dependencies
  libpq-dev \
  # Translations dependencies
  gettext \
  # cleaning up unused files
  && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
  && rm -rf /var/lib/apt/lists/*


COPY --chown=django:django ./compose/common/django/entrypoint /entrypoint
RUN sed -i 's/\r$//g' /entrypoint
RUN chmod +x /entrypoint

# copy application code to WORKDIR
COPY --chown=django:django . ${APP_HOME}

# make django owner of the WORKDIR directory as well.
RUN chown django:django ${APP_HOME}

USER django

ENTRYPOINT ["/entrypoint"]


# Python 'run' stage for local development
FROM python-run-stage-base as python-run-stage-local

ARG BUILD_ENVIRONMENT=local
ENV BUILD_ENV ${BUILD_ENVIRONMENT}

# All absolute dir copies ignore workdir instruction. All relative dir copies are wrt to the workdir instruction
# copy python dependency wheels from python-build-stage
COPY --from=python-build-stage-local /usr/src/app/wheels  /wheels/

# use wheels to install python dependencies
USER root
RUN pip install --no-cache-dir --no-index --find-links=/wheels/ /wheels/* \
  && rm -rf /wheels/


COPY --chown=django:django ./compose/local/django/start /start
RUN sed -i 's/\r$//g' /start
RUN chmod +x /start

{% if cookiecutter.use_celery == "y" %}
COPY --chown=django:django ./compose/local/django/celery/worker/start /start-celeryworker
RUN sed -i 's/\r$//g' /start-celeryworker
RUN chmod +x /start-celeryworker


COPY --chown=django:django ./compose/local/django/celery/beat/start /start-celerybeat
RUN sed -i 's/\r$//g' /start-celerybeat
RUN chmod +x /start-celerybeat


COPY --chown=django:django ./compose/local/django/celery/flower/start /start-flower
RUN sed -i 's/\r$//g' /start-flower
RUN chmod +x /start-flower

COPY --chown=django:django . ${APP_HOME}

USER django
{% endif %}


# Python 'run' stage for production
FROM python-run-stage-base as python-run-stage-production

ARG BUILD_ENVIRONMENT=production
ENV BUILD_ENV ${BUILD_ENVIRONMENT}

# All absolute dir copies ignore workdir instruction. All relative dir copies are wrt to the workdir instruction
# copy python dependency wheels from python-build-stage
COPY --from=python-build-stage-production /usr/src/app/wheels  /wheels/

# use wheels to install python dependencies
USER root
RUN pip install --no-cache-dir --no-index --find-links=/wheels/ /wheels/* \
  && rm -rf /wheels/

COPY --chown=django:django ./compose/production/django/start /start
RUN sed -i 's/\r$//g' /start
RUN chmod +x /start

{%- if cookiecutter.use_celery == "y" %}
COPY --chown=django:django ./compose/production/django/celery/worker/start /start-celeryworker
RUN sed -i 's/\r$//g' /start-celeryworker
RUN chmod +x /start-celeryworker


COPY --chown=django:django ./compose/production/django/celery/beat/start /start-celerybeat
RUN sed -i 's/\r$//g' /start-celerybeat
RUN chmod +x /start-celerybeat


COPY ./compose/production/django/celery/flower/start /start-flower
RUN sed -i 's/\r$//g' /start-flower
USER root
RUN chmod +x /start-flower
{%- endif %}


# copy application code to WORKDIR
{%- if cookiecutter.js_task_runner == 'Gulp' %}
COPY --from=client-builder --chown=django:django ${APP_HOME} ${APP_HOME}
{% else %}
COPY --chown=django:django . ${APP_HOME}
{%- endif %}

USER django
