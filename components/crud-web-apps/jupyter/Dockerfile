# --- Build the backend kubeflow-wheel ---
FROM focker.ir/python:3.10-slim AS backend-kubeflow-wheel

WORKDIR /src

COPY ./common/backend/ .
RUN python3 setup.py bdist_wheel

# --- Build the frontend kubeflow library ---
FROM focker.ir/node:16.20.2-bullseye as frontend-kubeflow-lib

WORKDIR /src

ENV NG_CLI_ANALYTICS "ci"
COPY ./common/frontend/kubeflow-common-lib/package.json ./
COPY ./common/frontend/kubeflow-common-lib/package-lock.json ./
RUN npm ci

COPY ./common/frontend/kubeflow-common-lib/projects ./projects
COPY ./common/frontend/kubeflow-common-lib/angular.json .
COPY ./common/frontend/kubeflow-common-lib/tsconfig.json .
RUN npm run build

# --- Build the frontend ---
FROM focker.ir/node:16.20.2-bullseye as frontend

WORKDIR /src

COPY ./jupyter/frontend/package.json ./
COPY ./jupyter/frontend/package-lock.json ./
COPY ./jupyter/frontend/tsconfig.json ./
COPY ./jupyter/frontend/tsconfig.app.json ./
COPY ./jupyter/frontend/tsconfig.spec.json ./
COPY ./jupyter/frontend/angular.json ./
COPY ./jupyter/frontend/src ./src
COPY ./jupyter/frontend/i18n/ ./i18n/
# RUN ls /src/i18n/fa
ENV NG_CLI_ANALYTICS "ci"
RUN npm ci
COPY --from=frontend-kubeflow-lib /src/dist/kubeflow/ ./node_modules/kubeflow/

RUN npm run build -- --output-path=./dist/default --configuration=production

# Web App
FROM focker.ir/python:3.10-slim

WORKDIR /package
COPY --from=backend-kubeflow-wheel /src .
RUN pip3 install .

WORKDIR /src
COPY ./jupyter/backend/requirements.txt .
RUN pip3 install -r requirements.txt

COPY ./jupyter/backend/apps/ ./apps
COPY ./jupyter/backend/entrypoint.py .

COPY --from=frontend /src/dist/default/ /src/apps/default/static/
RUN cp -r /src/apps/default/static/fa/* /src/apps/default/static/

ENTRYPOINT ["/bin/bash","-c","gunicorn -w 3 --bind 0.0.0.0:5000 --access-logfile - entrypoint:app"]