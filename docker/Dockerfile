ARG PYTHON_VERSION="3.9-slim-bookworm"
ARG NODE_VERSION="22-slim"

FROM node:${NODE_VERSION} AS frontend-builder

RUN apt-get update \
   && apt-get install -y --no-install-recommends git python3 make g++ ca-certificates \
   && apt-get clean \
   && rm -rf /var/lib/apt/lists/* \
   && corepack enable \
   && corepack prepare pnpm@latest --activate

WORKDIR /frontend
COPY frontend/package.json frontend/pnpm-lock.yaml* ./

RUN pnpm install --frozen-lockfile

COPY frontend/ ./

ENV PUBLIC_PATH="/static/_nuxt/"

RUN pnpm run build


FROM python:${PYTHON_VERSION} AS backend-builder

RUN apt-get update \
   && apt-get install -y --no-install-recommends \
   netcat-openbsd \
   libpq-dev \
   unixodbc-dev \
   g++ \
   libssl-dev \
   curl \
   && apt-get clean \
   && rm -rf /var/lib/apt/lists/*

ADD https://astral.sh/uv/install.sh /uv-installer.sh

RUN sh /uv-installer.sh && rm /uv-installer.sh

ENV PATH="/root/.local/bin/:$PATH"

# RUN pip install -U --no-cache-dir pip \
#    && pip install --no-cache-dir uv

WORKDIR /backend
COPY backend/pyproject.toml ./
COPY backend/ ./

# RUN uv sync --system pyproject.toml
RUN uv pip install --system . 

FROM python:${PYTHON_VERSION} AS runtime

RUN apt-get update \
   && apt-get install -y --no-install-recommends \
   libpq-dev \
   unixodbc-dev \
   libssl-dev \
   python-is-python3 \
   && apt-get clean \
   && rm -rf /var/lib/apt/lists/*


RUN useradd -ms /bin/sh doccano
RUN mkdir /data \
   && chown doccano:doccano /data

COPY --from=backend-builder /usr/local/lib/python3.9/site-packages /usr/local/lib/python3.9/site-packages
COPY --from=backend-builder /usr/local/bin/celery /usr/local/bin/celery
COPY --from=backend-builder /usr/local/bin/gunicorn /usr/local/bin/gunicorn

COPY --chown=doccano:doccano . /doccano
WORKDIR /doccano/backend

COPY --from=frontend-builder /frontend/dist /doccano/backend/client/dist

RUN python manage.py collectstatic --noinput \
   && chown -R doccano:doccano .

VOLUME /data

ENV DATABASE_URL="sqlite:////data/doccano.db"

ENV DEBUG="False"
ENV STANDALONE="True"
ENV SECRET_KEY="change-me-in-production"
ENV PORT="8000"
ENV WORKERS="2"
ENV CELERY_WORKERS="2"
ENV GOOGLE_TRACKING_ID=""
ENV DJANGO_SETTINGS_MODULE="config.settings.production"

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1 

USER doccano
EXPOSE ${PORT}

CMD ["/doccano/tools/run.sh"]
