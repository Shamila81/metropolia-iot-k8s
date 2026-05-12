# MQTTS Implementation Research & Kubernetes Deployment Guide

## Overview

This document describes the implementation of secure MQTT communication (MQTTS) for the Metropolia Garage infrastructure using Mosquitto, Kubernetes, TLS certificates, and Traefik TCP routing.

The goal is to migrate MQTT communication from unsecured TCP communication on port `1883` to encrypted TLS communication on port `8883`.

---

# Objective

The MQTT broker currently operates using plain-text communication:

```text
mqtt://broker:1883
```

This implementation introduces encrypted MQTT communication using TLS:

```text
mqtts://broker:8883
```

The secure implementation protects IoT telemetry, credentials, and device communication from network sniffing and interception.

---

# Architecture Overview

The implementation consists of:

- Mosquitto MQTT Broker
- TLS Certificates
- Kubernetes Secrets
- ConfigMaps
- Traefik TCP Ingress
- Dual Listener Support (`1883` + `8883`)

---

# TLS Certificate Strategy

## Recommended Approach: Existing Wildcard Certificate

The preferred approach is to reuse the existing wildcard TLS certificate:

```text
*.metrocloud.aiot-garage.net
```

Advantages:

- No self-signed certificate warnings
- Compatible with existing infrastructure
- Easier certificate renewal management
- Enterprise-style deployment model

---

# Required TLS Files

Mosquitto requires the following files:

| File | Purpose |
|---|---|
| `ca.crt` | Certificate Authority chain |
| `server.crt` | Public server certificate |
| `server.key` | Private key |

---

# Option A — Self-Signed Certificate Research

If no official certificate exists, OpenSSL can generate a local CA and broker certificate.

---

## Step 1 — Generate Certificate Authority

```bash
# Generate CA private key
openssl genrsa -out ca.key 2048

# Generate CA certificate
openssl req -new -x509 -days 3650 \
  -key ca.key \
  -out ca.crt \
  -subj "/CN=Metropolia-Lab-CA"
```

---

## Step 2 — Generate Server Key and CSR

```bash
# Generate server key
openssl genrsa -out server.key 2048

# Generate CSR
openssl req -new \
  -key server.key \
  -out server.csr \
  -subj "/CN=mqtt.metrocloud.aiot-garage.net"
```

---

## Step 3 — Sign the Server Certificate

```bash
openssl x509 -req \
  -in server.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out server.crt \
  -days 365
```

---

# Option B — Existing Domain Certificate (Recommended)

The instructor-recommended solution is to reuse the existing wildcard domain certificate already used within the infrastructure.

This is the production-grade solution.

---

# Kubernetes Implementation

## Phase 1 — Store Certificates as Kubernetes Secrets

The TLS files are stored securely using Kubernetes Secrets.

### Secret Creation via CLI

```bash
kubectl create secret generic mqtt-tls-certificates \
  --from-file=ca.crt \
  --from-file=server.crt \
  --from-file=server.key \
  -n mqtt
```

---

## Alternative Secret YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mosquitto-tls-certs
  namespace: mqtt

type: Opaque

stringData:
  ca.crt: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----

  server.crt: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----

  server.key: |
    -----BEGIN PRIVATE KEY-----
    ...
    -----END PRIVATE KEY-----
```

---

# Phase 2 — Mosquitto Configuration

The broker configuration is managed using a Kubernetes ConfigMap.

During migration, both listeners remain active:

| Port | Usage |
|---|---|
| `1883` | Legacy/testing/unsecured |
| `8883` | Secure TLS communication |

---

## Example mosquitto.conf

```conf
# Default MQTT Listener
listener 1883
protocol mqtt

# Secure MQTT Listener
listener 8883
protocol mqtt

cafile /etc/mosquitto/certs/ca.crt
certfile /etc/mosquitto/certs/server.crt
keyfile /etc/mosquitto/certs/server.key
```

---

# Phase 3 — Kubernetes ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mosquitto-config
  namespace: mqtt

data:
  mosquitto.conf: |
    listener 1883
    protocol mqtt

    listener 8883
    protocol mqtt

    cafile /etc/mosquitto/certs/ca.crt
    certfile /etc/mosquitto/certs/server.crt
    keyfile /etc/mosquitto/certs/server.key
```

---

# Phase 4 — Deployment Mounts

The Mosquitto Deployment mounts:

- ConfigMap → Mosquitto configuration
- Secret → TLS certificates

Certificates are mounted to:

```text
/etc/mosquitto/certs
```

This avoids permission issues caused by the default Mosquitto entrypoint modifying `/mosquitto`.

---

## Example Deployment Snippet

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mosquitto
  namespace: mqtt

spec:
  replicas: 1

  selector:
    matchLabels:
      app: mosquitto

  template:
    metadata:
      labels:
        app: mosquitto

    spec:
      containers:
        - name: mosquitto
          image: eclipse-mosquitto:2

          ports:
            - containerPort: 1883
            - containerPort: 8883

          volumeMounts:
            - name: mosquitto-config
              mountPath: /mosquitto/config

            - name: mosquitto-certs
              mountPath: /etc/mosquitto/certs
              readOnly: true

      volumes:
        - name: mosquitto-config
          configMap:
            name: mosquitto-config

        - name: mosquitto-certs
          secret:
            secretName: mosquitto-tls-certs
```

---

# Phase 5 — Kubernetes Service

The Mosquitto Service exposes both MQTT ports internally.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mosquitto
  namespace: mqtt

spec:
  selector:
    app: mosquitto

  ports:
    - name: mqtt
      port: 1883
      targetPort: 1883

    - name: mqtts
      port: 8883
      targetPort: 8883
```

---

# Phase 6 — Traefik TCP Ingress

Because MQTT is a raw TCP protocol rather than HTTP, Traefik requires `IngressRouteTCP`.

This allows external encrypted MQTT traffic to enter the Kubernetes cluster securely.

---

## Example IngressRouteTCP

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRouteTCP
metadata:
  name: mqtts-ingress
  namespace: mqtt

spec:
  entryPoints:
    - mqtts

  routes:
    - match: HostSNI(`mqtt.metrocloud.aiot-garage.net`)
      services:
        - name: mosquitto
          port: 8883
```

---

# Security Benefits

## TLS Encryption

Encrypts all MQTT traffic between devices and the broker.

---

## Credential Protection

Protects usernames and passwords from interception.

---

## Enterprise Standardization

Aligns with modern cloud-native deployment and Kubernetes best practices.

---

## Simplified Certificate Management

Using existing wildcard certificates reduces administrative overhead.

---

# Expected Result

After deployment:

## Unsecured MQTT

```text
mqtt://broker:1883
```

Use Cases:

- Internal labs
- Legacy clients
- Student testing

---

## Secure MQTT

```text
mqtts://broker:8883
```

Use Cases:

- Production IoT communication
- Remote device telemetry
- Secure authentication

---

# Best Practices

| Best Practice | Reason |
|---|---|
| Use `8883` for production | Standard secure MQTT port |
| Keep `1883` temporarily | Backward compatibility |
| Use Kubernetes Secrets | Protect private keys |
| Mount certs read-only | Improve security |
| Reuse wildcard certificates | Simplify renewals |
| Use Traefik TCP routing | Proper MQTT ingress handling |

---

# Final Conclusion

Implementing MQTTS significantly improves the security posture of the Metropolia Garage infrastructure.

By combining:

- Mosquitto
- TLS encryption
- Kubernetes Secrets
- ConfigMaps
- Traefik TCP routing
- Existing wildcard certificates

the system achieves:

- Secure communication
- Scalable Kubernetes deployment
- Easier maintenance
- Cloud-native infrastructure compliance

This implementation provides a production-ready MQTT architecture suitable for modern IoT deployments.