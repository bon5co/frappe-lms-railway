# Railway-tuned Frappe LMS: the learning management system (courses, lessons, quizzes,
# assignments, batches, certificates, live classes) in one container, plus MariaDB as
# the second service.
#
# Base: ghcr.io/frappe/lms:v16.4.0 -- but that image does NOT contain the lms app.
# Measured 2026-11-24 by listing apps/ in the image: every tag from v2.52.1 through
# v2.69.1 and v16.0.0 through v16.4.0 holds bare `frappe` (apps.txt = "frappe", no
# apps/lms, no apps/payments), although each tag's build workflow is green. v2.52.0
# is the last tag that carries lms (frappe 15.104.0). So the stock image cannot run an
# LMS; this wrapper takes upstream's frappe 16.60.2 base and installs `payments` and
# `lms` into it from the upstream git tags, then builds their assets.
#
# What this changes against running upstream's compose file on Railway:
#
#  1. One container runs the whole bench (nginx + gunicorn + socketio + worker +
#     scheduler + redis) under honcho. Railway volumes cannot be shared between
#     services, and frappe's web, worker and scheduler all need the same `sites/`
#     directory, so upstream's six-container split is not available.
#  2. The site is created on first boot under a FIXED name and nginx passes that name
#     as X-Frappe-Site-Name, so it resolves whatever host the request arrives on.
#  3. Administrator gets an API key pair from the environment, so an agent can
#     authenticate with a token header and no browser login.
# upstream: frappe/lms
FROM ghcr.io/frappe/lms:v16.4.0@sha256:a8dff54c23458bdf8b0b5fa8acdcb07912203e52774aabe206c6ed02f6fa8bd5

USER root

# The only package upstream does not already carry.
RUN apt-get update -qq \
    && apt-get install -y -q --no-install-recommends redis-server \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

# nginx's stock pid file lives where an unprivileged user cannot write.
RUN sed -i 's|^pid .*|pid /tmp/nginx.pid;|' /etc/nginx/nginx.conf

# lms declares required_apps = ["frappe/payments"]; both are pinned to upstream tags.
USER frappe
RUN cd /home/frappe/frappe-bench \
    && bench get-app --branch v16.0.1 --skip-assets payments https://github.com/frappe/payments \
    && bench get-app --branch v16.4.0 --skip-assets lms https://github.com/frappe/lms \
    && bench set-config -g socketio_port 9000 \
    && bench build --app payments --app lms \
    && rm -rf apps/lms/.git apps/payments/.git \
    && rm -rf /home/frappe/.cache /home/frappe/frappe-bench/apps/lms/node_modules /home/frappe/frappe-bench/apps/lms/frontend/node_modules
USER root

# sites/ is covered by the volume on Railway, which hides the app list. Keep a copy.
RUN mkdir -p /home/frappe/bench-seed \
    && cp /home/frappe/frappe-bench/sites/apps.txt \
          /home/frappe/frappe-bench/sites/apps.json \
          /home/frappe/bench-seed/ \
    && chown -R frappe:root /home/frappe/bench-seed

COPY --chown=frappe:root Procfile /home/frappe/frappe-bench/Procfile.railway
COPY railway-nginx.sh /usr/local/bin/railway-nginx.sh
COPY railway-entrypoint.sh /usr/local/bin/railway-entrypoint.sh
RUN chmod +x /usr/local/bin/railway-nginx.sh /usr/local/bin/railway-entrypoint.sh

ENV SITE_NAME=learn \
    PORT=8080 \
    BACKEND=127.0.0.1:8000 \
    SOCKETIO=127.0.0.1:9000 \
    FRAPPE_SITE_NAME_HEADER=learn \
    DB_PORT=3306 \
    DB_ROOT_USER=root \
    REDIS_URL=redis://127.0.0.1:6379 \
    GUNICORN_WORKERS=2 \
    GUNICORN_THREADS=4 \
    FRAPPE_WORKER_QUEUES=short,default,long \
    ADMIN_USER=Administrator

EXPOSE 8080

ENTRYPOINT ["/usr/local/bin/railway-entrypoint.sh"]
