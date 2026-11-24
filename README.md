# frappe-lms-railway

Railway wrapper image for Frappe LMS (the Learning app on the Frappe framework, with its
Payments dependency). One container runs nginx, gunicorn, socket.io, the worker, the scheduler
and redis under honcho; MariaDB is the second Railway service. Site `learn` is created on first
boot and the Administrator API key pair comes from `FRAPPE_API_KEY` / `FRAPPE_API_SECRET`.

Image: `ghcr.io/bon5co/frappe-lms-railway`.

The base is `ghcr.io/frappe/lms:v16.4.0` pinned by digest, and the image **adds the `lms` and
`payments` apps itself** (`bench get-app` at their release tags, then `bench build`). The reason:
every upstream `ghcr.io/frappe/lms` tag from `v2.52.1` to `v2.69.1` and `v16.0.0` to `v16.4.0`
contains only the `frappe` app, with no `lms` and no `payments`, although the build workflow is
green. `v2.52.0` is the last tag that ships the app.
