# 🔍 OpenTelemetry Kubernetes Setup Guide

> A comprehensive guide to setting up OpenTelemetry observability in Kubernetes with Flask applications

## 📋 Table of Contents

- [🤔 Frequently Asked Questions](#-frequently-asked-questions)
- [🔧 Component Overview](#-component-overview)
- [🐍 Flask Application Example](#-flask-application-example)
- [📦 Installation Steps](#-installation-steps)
- [⚙️ Configuration](#️-configuration)
- [🏗️ Architecture Overview](#️-architecture-overview)

---

## 🤔 Frequently Asked Questions

<details>
<summary><strong>❓ Question 1: If I deploy only the OpenTelemetry Operator (otel-operator) in my Kubernetes cluster, will I see traces in Jaeger from my Flask application?</strong></summary>

**✅ Answer:** No — the otel-operator only handles auto-instrumentation of your app. You still need to deploy the otel-collector, which receives the telemetry data from the app and forwards it to Jaeger. Without the collector, traces have nowhere to go.

</details>

<details>
<summary><strong>❓ Question 2: What is the main difference between the OpenTelemetry Operator and the OpenTelemetry Collector?</strong></summary>

**✅ Answer:**
* The **otel-operator** injects instrumentation (like the OpenTelemetry SDK) into your application pods automatically.
* The **otel-collector** receives, processes, and exports telemetry data (traces, metrics, logs) to observability backends like Jaeger.

</details>

### 🎯 Key Concepts

#### ✅ What otel-operator does:
* Injects the OpenTelemetry SDK into your app (e.g., Flask) automatically.
* Configures the SDK to export traces to a collector.

#### ❌ What it does not do:
* It does not collect, process, or export telemetry data to Jaeger, Prometheus, etc.
* It expects a collector to be present so the app has somewhere to send traces/metrics.

> **💡 Remember:** 
> - **otel-operator**: Automates instrumentation.
> - **otel-collector**: Handles telemetry data flow.

---

## 🔧 Component Overview

### 🤖 "Automates instrumentation" means:

You don't have to manually modify your app code to add tracing or metrics logic. The OpenTelemetry Operator does it for you automatically by:

1. **Injecting** the OpenTelemetry SDK into your app container (as a sidecar or shared volume).
2. **Setting** environment variables needed to start collecting telemetry (like exporter endpoints).
3. **Hooking** into libraries (e.g., Flask, Requests, SQLAlchemy) to create traces without you writing extra code.

---

## 🐍 Flask Application Example

### 📁 Flask Code Structure

```python
from flask import Flask, jsonify
import time
import random
import requests

app = Flask(__name__)

@app.route('/')
def home():
    # Simulate some processing time
    time.sleep(random.uniform(0.1, 0.5))
    return jsonify({
        "message": "Hello from Flask Observability Demo!",
        "status": "success"
    })

@app.route('/api/users')
def get_users():
    # Simulate database query delay
    time.sleep(random.uniform(0.2, 0.8))
    
    users = [
        {"id": 1, "name": "Alice", "email": "alice@example.com"},
        {"id": 2, "name": "Bob", "email": "bob@example.com"},
        {"id": 3, "name": "Charlie", "email": "charlie@example.com"}
    ]
    
    return jsonify({"users": users, "count": len(users)})

@app.route('/api/external')
def call_external():
    # Simulate external API call
    try:
        # This will create a span for the HTTP request
        response = requests.get('https://httpbin.org/delay/1', timeout=5)
        return jsonify({
            "external_status": response.status_code,
            "message": "External API call successful"
        })
    except Exception as e:
        return jsonify({
            "error": str(e),
            "message": "External API call failed"
        }), 500

@app.route('/health')
def health():
    return jsonify({"status": "healthy"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

### 📊 API Endpoints Overview

| Route | Purpose | Delay Simulation |
|-------|---------|------------------|
| `/` | Basic "Hello" message | 100ms–500ms |
| `/api/users` | Returns a list of fake users | 200ms–800ms |
| `/api/external` | Calls an external API (httpbin.org) | ~1 second |
| `/health` | Health check endpoint | None |

### 🕒 Where Delays Are Introduced

Each route simulates processing delay using `time.sleep()` with `random.uniform()` to make responses a bit slow and unpredictable, which is helpful when testing observability and performance monitoring tools like Jaeger.

| Route | Code | Delay Simulation |
|-------|------|------------------|
| `/` | `time.sleep(random.uniform(0.1, 0.5))` | 100ms–500ms |
| `/api/users` | `time.sleep(random.uniform(0.2, 0.8))` | 200ms–800ms |
| `/api/external` | Makes a real HTTP call to `httpbin.org/delay/1` | ~1 second |

### 📈 Span Creation Points

> **What is a "Span"?** A span is a unit of work in a trace — like a timer around a block of logic. OpenTelemetry creates spans to track how long certain operations take.

#### ✅ Automatic Span Creation

**`requests.get(...)` (in `/api/external`)**
```python
response = requests.get('https://httpbin.org/delay/1', timeout=5)
```
This line triggers a new span automatically (but only if OpenTelemetry auto-instrumentation is active). The span will represent the HTTP call to the external service.

#### 🛠️ Other Automatic Spans

The rest of your app (Flask routes, delays, etc.) can also have spans, but you don't need to write them manually. If you're using OpenTelemetry auto-instrumentation, it will:

* ✅ Automatically wrap Flask route handlers with spans.
* ✅ Automatically capture function entry/exit times.
* ✅ Auto-instrument common libraries (like requests, Flask, logging, etc.).

<details>
<summary><strong>🔍 Will the OpenTelemetry Operator automatically find that Flask function and add a span?</strong></summary>

**Yes — exactly!** If you're using the OpenTelemetry Operator with auto-instrumentation, it will:

✅ Automatically detect that your Flask app is using supported libraries like Flask, requests, etc.  
✅ Auto-inject the OpenTelemetry Python SDK into your container at runtime (via sidecar or init logic).  
✅ Automatically create spans for:
* Incoming HTTP requests (Flask route handlers)
* Outgoing HTTP calls (requests.get, etc.)
* Logging (if enabled)
* Other standard Python libraries (SQLAlchemy, Redis, etc.)

</details>

### 🐳 Dockerfile Configuration

```dockerfile
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

#### 🔥 What is Gunicorn?
* Gunicorn is a tool that runs your Flask app in a safe and fast way for production.
* It can run multiple workers (here, 2), so it can handle many requests at once (not just one at a time like flask run).

#### 🧠 What does `app:app` mean?
* First `app` → the filename: `app.py`
* Second `app` → the Flask app inside it: `app = Flask(__name__)`

---

## 📦 Installation Steps

### 1. 🔐 Install cert-manager

<details>
<summary><strong>❓ What is cert-manager?</strong></summary>

**🟢 Answer:** cert-manager is a tool in Kubernetes that automatically creates and manages SSL/TLS certificates. These certificates are like digital ID cards used to secure communication between apps, like HTTPS websites do.

You can think of it as a "certificate robot" that handles all the hard work of creating and renewing certificates for you.

</details>

<details>
<summary><strong>❓ Why do we need cert-manager with OpenTelemetry Operator?</strong></summary>

**🟢 Answer:** The OpenTelemetry Operator uses webhooks to check or edit resources in Kubernetes. These webhooks need to be secure, and for that, they need TLS certificates.

cert-manager gives the Operator those certificates automatically, so everything works safely and smoothly — without you doing it by hand.

Without cert-manager, the Operator's security features (webhooks) won't work correctly.

</details>

```bash
helm repo add jetstack https://charts.jetstack.io

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

### 2. 🔧 Install OpenTelemetry Operator

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts

helm search repo open-telemetry

helm fetch open-telemetry/opentelemetry-operator

helm upgrade --install opentelemetry-operator open-telemetry/opentelemetry-operator
```

### 3. 🔍 Install Jaeger

```bash
helm search repo jaegertracing

helm fetch jaegertracing/jaeger 

helm upgrade --install jaeger

# Alternative with custom values
svc=jaeger; helm upgrade -i $svc ./$svc -n default -f ./jaeger/values.yaml
```

---

## ⚙️ Configuration

### 📄 otel-collector.yaml

> **Important:** You are able to create this custom resource of kind `OpenTelemetryCollector` only because you have deployed the OpenTelemetry Operator.

#### 📦 What the Operator Does

When you install the OpenTelemetry Operator, it:

* Registers new custom resources like `OpenTelemetryCollector` (via CRDs).
* Watches for these custom resources.
* Automatically creates and manages the actual OpenTelemetry Collector Pods based on your configuration.

<details>
<summary><strong>❓ If I've already installed the OpenTelemetry Operator, do I still need to define and apply an OpenTelemetryCollector resource in Kubernetes?</strong></summary>

**✅ Answer:** Yes, even after installing the OpenTelemetry Operator, you still need to define and apply an `OpenTelemetryCollector` resource. The Operator doesn't automatically create a collector for you — instead, it watches for this custom resource (`kind: OpenTelemetryCollector`) and then creates and manages the actual collector deployment based on the configuration you provide.

</details>

### 📄 instrumentation.yaml

This configuration tells the OpenTelemetry Operator how to automatically inject instrumentation into your Python application at runtime, without you needing to change the app code manually.

**✅ In short:** This file tells the Operator: "Hey, for Python apps in this namespace, automatically add OpenTelemetry auto-instrumentation using this config and send traces to the collector."

### 📊 Component Deployment Matrix

| Component | Deploy Once? | Namespace Scoped? | Notes |
|-----------|--------------|-------------------|-------|
| OpenTelemetry Operator | ✅ Yes | ❌ No | Can monitor multiple namespaces. |
| Instrumentation CR | ❌ No | ✅ Yes | Must exist in each app's namespace. |

---

## 🏗️ Architecture Overview

<details>
<summary><strong>❓ What is the role of the OpenTelemetry Operator, Instrumentation, and Collector in setting up observability for a Python (Flask) application in Kubernetes?</strong></summary>

**✅ What You Got Right:**
* ✅ Instrumentation is a Custom Resource (CR) that tells the Operator how to auto-instrument your application.
* ✅ OpenTelemetry Collector is responsible for receiving, processing, and exporting telemetry data (e.g., to Jaeger).
* ✅ OpenTelemetry Operator is the "brain" that deploys and manages both Instrumentation and Collector if defined.

</details>

### 🔄 Correct Hierarchy & Flow

```
[OpenTelemetry Operator]
├── Watches your app deployments
├── Injects SDK/agent based on the Instrumentation CR
└── Can also deploy OpenTelemetryCollector CR

    │
    ▼

[Instrumentation CR] ← (per namespace)
├── Adds auto-instrumentation (e.g., sidecar/init container or env vars)
└── Injects SDK into your app automatically

    │
    ▼

[Your Application] ──── sends traces/logs/metrics ───→ [OpenTelemetry Collector]
                                                      └── exports to Jaeger/Prometheus
```

### 🎯 Key Relationships

1. **[OpenTelemetry Operator]** watches your app deployments
2. **[Instrumentation CR]** (per namespace) adds auto-instrumentation 
3. **[Your Application]** sends telemetry data to **[OpenTelemetry Collector]**
4. **[OpenTelemetry Collector]** exports to Jaeger/Prometheus

---

## 🚀 Getting Started

1. **Install Prerequisites**: cert-manager, OpenTelemetry Operator, Jaeger
2. **Deploy Collector**: Create and apply `otel-collector.yaml`
3. **Configure Instrumentation**: Create and apply `instrumentation.yaml` in your app's namespace
4. **Deploy Your App**: Your Flask application with proper annotations
5. **View Traces**: Access Jaeger UI to see your application traces

---

## 📚 Additional Resources

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Kubernetes Operator Guide](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [cert-manager Documentation](https://cert-manager.io/docs/)

---

> **💡 Pro Tip:** The beauty of OpenTelemetry auto-instrumentation is that you get comprehensive observability without modifying your application code. The operator handles all the complexity for you!