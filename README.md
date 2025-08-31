## APM Datadog Tracing +Log+Profiling
### Step1 : Check the code
-- cd notes_app
--- vi app.py (check whether datadog tracing code is ingested)

```bash
from notes_app.notes_logic import NotesLogic
from flask import Flask, request


# tracing
from ddtrace import patch_all
patch_all()

# logging imports BEFORE calling _setup_logging()
import os, sys
import logging
from pythonjsonlogger import jsonlogger


def _setup_logging():
    root = logging.getLogger()
    root.setLevel(logging.INFO)

    h = logging.StreamHandler(sys.stdout)
    fmt = '%(asctime)s %(levelname)s %(name)s %(message)s %(dd.trace_id)s %(dd.span_id)s service=%(service)s env=%(env)s version=%(version)s'
    h.setFormatter(jsonlogger.JsonFormatter(fmt))

    class _SvcFilter(logging.Filter):
        def filter(self, record):
            record.service = os.getenv("DD_SERVICE", "notes-app")
            record.env = os.getenv("DD_ENV", "dev")
            record.version = os.getenv("DD_VERSION", "1.0.0")
            return True
    h.addFilter(_SvcFilter())

    root.handlers = [h]

    w = logging.getLogger("werkzeug")
    w.setLevel(logging.INFO)
    w.handlers = [h]
    w.propagate = False


# ✅ only call after imports are in place
_setup_logging()

```
### Step2 : check requirements.txt
```bash
flask==2.2.2
psycopg2-binary==2.9.6
requests==2.28.1
ddtrace
Werkzeug==2.2.2
python-json-logger
gunicorn
```

### Step 2.1 Check Dockerfile created
```bash
# Unless explicitly stated otherwise all files in this repository are dual-licensed
# under the Apache 2.0 or BSD3 Licenses.
#
# This product includes software developed at Datadog (https://www.datadoghq.com/)
# Copyright 2022 Datadog, Inc.
#FROM python:3
FROM python:3.11-slim
ENV DD_SERVICE="notes"
ENV DD_ENV="dev"
ENV DD_VERSION="0.1.0"
ENV DD_LOGS_INJECTION="true"
LABEL com.datadoghq.tags.service="notes"
LABEL com.datadoghq.tags.env="dev"
LABEL com.datadoghq.tags.version="0.1.0"
WORKDIR /home
COPY requirements.txt /home
COPY notes_app /home/notes_app

RUN pip install --no-cache-dir -r requirements.txt
# Run the application with Datadog
CMD ["ddtrace-run", "python", "-m", "notes_app.app"]
# Run with gunicorn in production
#CMD ["ddtrace-run", "gunicorn", "-b", "0.0.0.0:8080", "notes.app:app"]
```

### Step 3 : Create the Docker image usign Dockerfile_Datadog
```bash
docker build -t sumanth17121988/notes_app:1 -f Dockerfile_Datadog .
```
### Step 4 : Run on Kubernetes Deployment (Update the image  sumanth17121988/notes_app:1 in deployment.yaml) and validate the datadog parameter.
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notes
  labels:
    app: notes
    # Unified service tags
    tags.datadoghq.com/env: dev
    tags.datadoghq.com/service: notes
    tags.datadoghq.com/version: "0.1.0"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: notes
  template:
    metadata:
      labels:
        app: notes
        tags.datadoghq.com/env: dev
        tags.datadoghq.com/service: notes
        tags.datadoghq.com/version: "0.1.0"
        admission.datadoghq.com/enabled: "true"
      annotations:
        # Auto-instrument Python
        admission.datadoghq.com/python-lib.version: v2.3.1
        admission.datadoghq.com/config.mode: "hostip"
        # Log collection mapping
        ad.datadoghq.com/notes.logs: >-
          [{"source":"python","service":"notes","auto_multi_line_detection": true}]
    spec:
      containers:
        - name: notes
          image: sumanth17121988/notes:v2
          ports:
            - containerPort: 8080
          env:
            # Unified Service Tags (auto-populated from pod labels)
            - name: DD_ENV
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['tags.datadoghq.com/env']
            - name: DD_SERVICE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['tags.datadoghq.com/service']
            - name: DD_VERSION
              valueFrom:
                  fieldRef:
                    fieldPath: metadata.labels['tags.datadoghq.com/version']
            # Datadog Agent host (uses node's IP)
            - name: DD_AGENT_HOST
              valueFrom:
                fieldRef:
                  fieldPath: status.hostIP
            # Enable instrumentation features
            - name: DD_LOGS_INJECTION
              value: "true"
            - name: DD_PROFILING_ENABLED
              value: "true"
            - name: DD_RUNTIME_METRICS_ENABLED
              value: "true"
            - name: DD_TRACE_SAMPLE_RATE
              value: "1"
            - name: DD_TRACE_STARTUP_LOGS
              value: "true"
            - name: DD_DYNAMIC_INSTRUMENTATION_ENABLED
              value: "true"
            - name: DD_APPSEC_ENABLED
              value: "true"
            - name: DD_IAST_ENABLED
              value: "true"
            - name: DD_APPSEC_SCA_ENABLED
              value: "true"
            - name: DD_TAGS
              value: "app-name:news,team:devops"
            - name: DD_TRACE_AGENT_URL
              value: "http://$(DD_AGENT_HOST):8126"
---
apiVersion: v1
kind: Service
metadata:
  name: notes-service
  labels:
    app: notes
spec:
  type: LoadBalancer
  selector:
    app: notes
  ports:
    - protocol: TCP
      port: 8080        # external port
      targetPort: 8080 # app listens here
```


# apm-tutorial-python

The notes application and calendar application are both REST API's. The notes application has POST, GET, PUT and DELETE operations for creating, getting, updating and deleting notes. Additionally, the notes application POST /notes method has an additional parameters, add_date, that can be set to 'y' in order to make a call to the calendar application for a random date. This can be used to show distributed tracing across applications.

This is a sample Python application made to run in various deployment scenarios with two different services, a notes application and calendar application, in order to provide sample distributed tracing. The application is used in a tutorial showcasing how to enable APM tracing for an application. The different ways to deploy these applications are:
- locally on host machine (with Datadog Agent also running on host)
- within Docker containers (with Datadog Agent also in a container)
- within Docker containers (with Datadog Agent running on host)
- Google Kubernetes Engine (GKE)
- Amazon AWS Elastic Kubernetes Service (AWS EKS)

The sample application is a very simple pair of rest APIs, as seen below. All commands below are for host and/or Docker container deployment situations. For Kubernetes deployments, the URL will be that of the Kubernetes notes or calendar service. 

# REST APIs

## Calendar Application

### `GET /calendar`

Returns a random date in 2022.

#### Request

```sh
curl 'http://localhost:9090/calendar'
```

#### Response

```json
{"status":"success","date":"3/22/2022"}
```

## Notes Application

### `GET /notes/`

#### Request

```sh
curl 'http://localhost:8080/notes'
```

#### Response

```json
[{ "id": 0, "description": "Hello, this is a note." }]
```

### `POST /notes`

#### Request - without optional add_date parameter

```sh
curl -X POST 'http://localhost:8080/notes?desc=ImANote'
```

#### Response

```json
{ "id": 1, "description": "ImANote" }
```

#### Request - with optional add_date parameter

Makes a request to calendar service on port 9090.

```sh
curl -X POST 'http://localhost:8080/notes?desc=ImANoteWithDate&add_date=y'
```

#### Response

```json
{ "id": 2, "description": "HiImANoteWithDate. Message Date: 09/21/2022" }
```

### `GET /notes/:id`

#### Request

```sh
curl 'http://localhost:8080/notes/1'
```

#### Response

```json
{ "id": 1, "description": "ImANote" }
```

### `PUT /notes/:id`

#### Request

```sh
curl -X PUT 'http://localhost:8080/notes/1?desc=UpdatedNote'
```

#### Response

```json
{ "id": 1, "description": "UpdatedNote" }
```

### `DELETE /notes/:id`

#### Request

```sh
curl -X DELETE 'http://localhost:8080/notes/1'
```

#### Response

```json
{ "message": "note with id 4 deleted." }
```
