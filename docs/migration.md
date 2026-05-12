# Week 2: YAML Migration & Cluster Logic

## Overview

This document explains the migration process from Podman-based container deployments to Kubernetes-native YAML manifests within the Metropolia Garage infrastructure.

The migration focused on:

- Standardizing deployments
- Improving maintainability
- Implementing persistent storage
- Enabling internal Kubernetes networking
- Integrating Traefik ingress routing
- Preparing services for scalable cloud-native operation

---

# 1. Migration Strategy & Logic

## Podman to Kubernetes Transition

The original services were running as standalone Podman containers.  
The objective was to transform them into fully Kubernetes-managed deployments using YAML manifests.

This required rebuilding:

- Deployments
- Services
- Persistent Volumes
- ConfigMaps
- Secrets
- Ingress configurations

---

## Template Alignment

All manifests were rewritten to match the supervisor-provided Kubernetes templates.

### Goals

- Maintain consistent structure
- Improve readability
- Simplify future maintenance
- Standardize deployment logic across the cluster

---

## Namespace Isolation

Services were separated into dedicated namespaces:

| Namespace | Purpose |
|---|---|
| `mqtt` | Messaging services |
| `homeassistant` | Home Assistant deployment |
| `go2rtc` | Real-time camera streaming |

### Benefits

- Prevents resource conflicts
- Improves organization
- Simplifies troubleshooting
- Enables secure service grouping

---

## Storage Transition

Based on supervisor guidance, all Persistent Volume Claims (PVCs) were standardized to the:

```text
local-path
```

storage class.

### Reasoning

- Matches lab infrastructure standards
- Simplifies persistent storage allocation
- Provides node-local persistent volumes
- Reduces configuration complexity

---

## Ingress Implementation

Traefik ingress routing was added to expose services externally through the cluster edge network.

### Components Used

| Resource | Purpose |
|---|---|
| `IngressRoute` | HTTP/HTTPS routing |
| `IngressRouteTCP` | Raw TCP routing (MQTT) |

---

# 2. Analysis of the Four Key YAML Manifests

---

# I. Mosquitto (MQTT Broker)

Mosquitto acts as the central messaging broker for the IoT ecosystem.

---

## Namespace Placement

The service is deployed inside the:

```text
mqtt
```

namespace.

This groups all messaging-related services together.

---

## Dual Listener Configuration

Mosquitto was configured with two listeners:

| Port | Purpose |
|---|---|
| `1883` | Standard unsecured MQTT |
| `8883` | Secure MQTTS using TLS |

### Benefits

- Backward compatibility
- Secure production communication
- Easier migration for existing devices

---

## Persistent Storage

A Persistent Volume Claim (PVC) using:

```text
local-path
```

stores:

- MQTT database
- Retained messages
- Logs
- Persistent broker state

---

## IngressRouteTCP

MQTT uses raw TCP communication rather than HTTP.

Therefore:

```yaml
IngressRouteTCP
```

was required instead of a standard HTTP ingress.

---

# II. Zigbee2MQTT

Zigbee2MQTT connects physical Zigbee hardware to the MQTT ecosystem.

---

## Namespace Consolidation

The service was placed in the:

```text
mqtt
```

namespace alongside Mosquitto.

### Reasoning

- Simplifies internal communication
- Reduces cross-namespace complexity
- Improves maintainability

---

## Hardware Access

The deployment required direct access to the Zigbee USB coordinator.

### Kubernetes Requirements

```yaml
privileged: true
```

and:

```yaml
hostPath: /dev/ttyACM0
```

were added.

### Purpose

Allows the container to communicate with the physical USB Zigbee adapter attached to the host node.

---

## Internal Kubernetes Networking

Instead of using external IP addresses, Zigbee2MQTT connects internally using Kubernetes DNS.

### Example

```text
mosquitto.mqtt.svc.cluster.local
```

### Benefits

- Stable internal routing
- Easier service discovery
- Reduced dependency on external networking

---

# III. Home Assistant

Home Assistant provides the central automation and management interface for the lab infrastructure.

---

## Migration Goal

The service was transitioned from a standalone container deployment into a persistent Kubernetes-managed application.

---

## Persistent Volume Claim

A:

```text
5Gi
```

Persistent Volume Claim was allocated using:

```text
local-path
```

### Stores

- Configuration files
- Add-on data
- SQLite database
- Automation rules
- User settings

---

## Hardware Discovery Support

A hostPath mount was added for:

```text
/run/dbus
```

### Purpose

Allows Home Assistant to interact with and discover local hardware services on the host system.

---

# IV. Go2RTC

Go2RTC manages real-time camera and streaming services within the Kubernetes cluster.

---

## Real-Time Streaming Support

The deployment exposes multiple ports for different protocols.

| Port | Protocol |
|---|---|
| `1984` | API |
| `8554` | RTSP |
| `8555` | WebRTC |

---

## GPU Acceleration Planning

The manifest structure was designed to support future hardware acceleration.

### Important Note

H.265 encoding and decoding are significantly more efficient when processed by GPU hardware.

This allows future scalability for:

- High-resolution camera streams
- Reduced CPU usage
- Improved transcoding performance

---

## ConfigMap-Based Configuration

The:

```text
go2rtc.yaml
```

configuration file is injected using a ConfigMap.

### Benefits

- No container shell access required
- Easier updates
- Kubernetes-native configuration management
- Version-controlled infrastructure

---

# 3. Technical Challenges & Resolutions

---

# Immutable Storage Errors

## Problem

Errors occurred when attempting to modify:

```text
storageClassName
```

on existing Persistent Volume Claims.

Kubernetes treats several PVC fields as immutable after creation.

---

## Resolution

The existing PVCs were:

1. Deleted
2. Recreated using:

```text
local-path
```

as required by the supervisor guidelines.

---

# Podman to Kubernetes Translation Issues

## Problem

Podman-generated Kubernetes YAML files are not always production-ready.

Several manifests lacked:

- Proper storage definitions
- Correct networking logic
- Resource configuration
- Persistent volume mappings

---

## Resolution

Each YAML file was manually reviewed and corrected to match:

- Internal lab templates
- Kubernetes best practices
- Supervisor deployment standards

---

# Internal Service Communication

## Problem

Services needed reliable communication across namespaces.

---

## Resolution

Kubernetes internal DNS naming conventions were used.

### Example Format

```text
service.namespace.svc.cluster.local
```

### Example

```text
mosquitto.mqtt.svc.cluster.local
```

This ensured stable and predictable inter-service communication inside the cluster.

---

# Final Outcome

By the end of the migration:

- All major services were Kubernetes-native
- Persistent storage was standardized
- Internal DNS communication was operational
- Traefik ingress routing was implemented
- MQTT secure communication support was prepared
- Infrastructure became easier to maintain and scale

---

# Summary

The Week 2 migration established the foundational Kubernetes architecture for the Metropolia Garage IoT environment.

Key achievements included:

- Converting Podman deployments into Kubernetes manifests
- Implementing namespace isolation
- Standardizing storage using `local-path`
- Enabling internal cluster networking
- Deploying Traefik ingress routing
- Preparing secure MQTT communication

This migration significantly improved infrastructure organization, scalability, maintainability, and readiness for future cloud-native expansion.