# Part 70: JSON Advanced Features
## Steps 661-670: JSON Schema, JSONPath, JQ, and More

---

## Step 661: JSON Schema Fundamentals

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/schemas/pod.json",
  "title": "Kubernetes Pod",
  "type": "object",
  "required": ["apiVersion", "kind", "metadata", "spec"],
  "properties": {
    "apiVersion": {"type": "string", "const": "v1"},
    "kind": {"type": "string", "const": "Pod"},
    "metadata": {"$ref": "#/$defs/ObjectMeta"},
    "spec": {"$ref": "#/$defs/PodSpec"}
  },
  "$defs": {
    "ObjectMeta": {
      "type": "object",
      "required": ["name"],
      "properties": {
        "name": {
          "type": "string",
          "minLength": 1,
          "maxLength": 253,
          "pattern": "^[a-z0-9][a-z0-9.-]*[a-z0-9]$"
        },
        "namespace": {"type": "string", "default": "default"},
        "labels": {
          "type": "object",
          "additionalProperties": {"type": "string"}
        }
      }
    },
    "PodSpec": {
      "type": "object",
      "required": ["containers"],
      "properties": {
        "containers": {
          "type": "array",
          "minItems": 1,
          "items": {"$ref": "#/$defs/Container"}
        }
      }
    },
    "Container": {
      "type": "object",
      "required": ["name", "image"],
      "properties": {
        "name": {"type": "string"},
        "image": {"type": "string", "pattern": "^[^:]+:[^:]+$"},
        "ports": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["containerPort"],
            "properties": {
              "containerPort": {
                "type": "integer",
                "minimum": 1,
                "maximum": 65535
              }
            }
          }
        }
      }
    }
  }
}
```

---

## Step 662: JSON Schema - Advanced Validation

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Deployment Config",
  "type": "object",
  "properties": {
    "replicas": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100,
      "default": 1
    },
    "strategy": {
      "type": "string",
      "enum": ["RollingUpdate", "Recreate"],
      "default": "RollingUpdate"
    },
    "image": {
      "type": "string",
      "not": {"pattern": ":latest$"},
      "description": "Image must not use :latest tag"
    },
    "environment": {
      "type": "string",
      "enum": ["development", "staging", "production"]
    }
  },
  "allOf": [
    {
      "if": {
        "properties": {"environment": {"const": "production"}}
      },
      "then": {
        "properties": {
          "replicas": {"minimum": 2}
        },
        "required": ["resources"]
      }
    }
  ],
  "unevaluatedProperties": false
}
```

---

## Step 663: JSONPath for Kubernetes

```bash
# Basic field extraction
kubectl get pod mypod -o jsonpath='{.spec.containers[0].image}'
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'

# Multiple fields with formatting
kubectl get pods -o jsonpath=\
  '{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Filter with condition
kubectl get pods -o jsonpath=\
  '{.items[?(@.status.phase=="Running")].metadata.name}'

# Custom columns
kubectl get pods -o custom-columns=\
  'NAME:.metadata.name,IMAGE:.spec.containers[0].image,STATUS:.status.phase'

# Sort by field
kubectl get pods --sort-by='.metadata.creationTimestamp'
kubectl get pods --sort-by='.status.startTime'

# Go template (more powerful)
kubectl get pods -o go-template=\
  '{{range .items}}{{.metadata.name}} {{.status.phase}}{{"\n"}}{{end}}'

# Extract container resource limits
kubectl get pods -o jsonpath=\
  '{range .items[*]}{.metadata.name}{range .spec.containers[*]}{": "}{.name}{"="}{.resources.limits.memory}{"\n"}{end}{end}'
```

---

## Step 664: JQ - JSON Processor

```bash
# All pod names
kubectl get pods -o json | jq '.items[].metadata.name'

# Format as TSV table
kubectl get pods -o json | jq -r '.items[] | [.metadata.name, .status.phase] | @tsv'

# Filter running pods
kubectl get pods -o json | jq '.items[] | select(.status.phase=="Running") | .metadata.name'

# Count pods by status
kubectl get pods -o json | jq '[.items[].status.phase] | group_by(.) | map({(.[0]): length}) | add'

# All images in cluster
kubectl get pods -A -o json | jq -r '.items[].spec.containers[].image' | sort -u

# Find pods without resource limits
kubectl get pods -o json | \
  jq '.items[] | select(.spec.containers[].resources.limits == null) | .metadata.name'

# Cluster summary
kubectl get pods -o json | jq '{
  total: (.items | length),
  running: [.items[] | select(.status.phase=="Running")] | length,
  failed: [.items[] | select(.status.phase=="Failed")] | length,
  images: [.items[].spec.containers[].image] | unique
}'

# Find privileged containers (security)
kubectl get pods -A -o json | jq '.items[] |
  .metadata as $meta |
  .spec.containers[] |
  select(.securityContext.privileged == true) |
  {pod: $meta.name, ns: $meta.namespace, container: .name}'

# Pods running as root
kubectl get pods -o json | jq '.items[] | select(
  .spec.containers[].securityContext.runAsUser == 0 or
  .spec.securityContext.runAsNonRoot != true
) | .metadata.name'
```

---

## Step 665: JSON Patch (RFC 6902)

```json
// JSON Patch operations: add, remove, replace, move, copy, test

// Add label
[
  {
    "op": "add",
    "path": "/metadata/labels/environment",
    "value": "production"
  }
]

// Replace replicas
[
  {
    "op": "replace",
    "path": "/spec/replicas",
    "value": 3
  }
]

// Remove annotation
[
  {
    "op": "remove",
    "path": "/metadata/annotations/deprecated-annotation"
  }
]

// Add resource limits to first container
[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/resources",
    "value": {
      "limits": {"cpu": "500m", "memory": "256Mi"},
      "requests": {"cpu": "100m", "memory": "128Mi"}
    }
  }
]

// Test + replace (atomic: fails if test fails)
[
  {
    "op": "test",
    "path": "/spec/replicas",
    "value": 1
  },
  {
    "op": "replace",
    "path": "/spec/replicas",
    "value": 3
  }
]
```

```bash
# Apply JSON Patch with kubectl
kubectl patch deployment myapp --type=json \
  -p='[{"op":"replace","path":"/spec/replicas","value":3}]'
```

---

## Step 666: JSON Merge Patch (RFC 7396)

```bash
# JSON Merge Patch: null removes field, else merges

# Update replicas
kubectl patch deployment myapp --type=merge \
  -p='{"spec":{"replicas":3}}'

# Add annotation
kubectl patch deployment myapp --type=merge \
  -p='{"metadata":{"annotations":{"new-key":"new-value"}}}'

# Remove label (set to null)
kubectl patch deployment myapp --type=merge \
  -p='{"metadata":{"labels":{"old-label":null}}}'

# Strategic Merge Patch (K8s default for kubectl apply)
# Merges arrays by key (e.g. containers by name)
kubectl patch deployment myapp --type=strategic \
  -p='{
    "spec": {
      "template": {
        "spec": {
          "containers": [
            {
              "name": "app",
              "image": "myapp:v2.0",
              "resources": {
                "limits": {"cpu": "1000m", "memory": "512Mi"}
              }
            }
          ]
        }
      }
    }
  }'
```

---

## Step 667: JSON in Kubernetes ConfigMaps

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-json
data:
  config.json: |
    {
      "database": {
        "host": "postgres",
        "port": 5432,
        "name": "myapp",
        "pool": {
          "min": 2,
          "max": 10,
          "idleTimeout": 30000
        }
      },
      "cache": {
        "ttl": 3600,
        "maxSize": 1000
      },
      "features": {
        "newUI": true,
        "betaAPI": false
      }
    }
  
  logging.json: |
    {
      "level": "info",
      "format": "json",
      "outputs": ["stdout"],
      "fields": {
        "service": "myapp",
        "env": "production"
      }
    }

---
apiVersion: v1
kind: Pod
metadata:
  name: app-with-json-config
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: config
          mountPath: /etc/config
          readOnly: true
      env:
        - name: CONFIG_PATH
          value: /etc/config/config.json
  volumes:
    - name: config
      configMap:
        name: app-config-json
```

---

## Step 668: JWT Tokens in Kubernetes

```bash
# Kubernetes SA tokens are JWTs
# Get token from running pod
kubectl exec mypod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Decode JWT payload (base64url decode middle segment)
TOKEN=$(kubectl exec mypod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq .
```

```json
{
  "aud": ["https://kubernetes.default.svc"],
  "exp": 1735689600,
  "iat": 1704153600,
  "iss": "https://kubernetes.default.svc",
  "kubernetes.io": {
    "namespace": "default",
    "pod": {"name": "mypod", "uid": "abc123"},
    "serviceaccount": {"name": "my-sa", "uid": "def456"}
  },
  "sub": "system:serviceaccount:default:my-sa"
}
```

```yaml
# Projected token with custom audience and short expiry
apiVersion: v1
kind: Pod
metadata:
  name: projected-token-pod
spec:
  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: token
          mountPath: /var/run/secrets/tokens
  volumes:
    - name: token
      projected:
        sources:
          - serviceAccountToken:
              path: app-token
              expirationSeconds: 3600
              audience: my-app-audience
```

---

## Step 669: JSON Structured Logging

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: json-logging-app
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:1.0
          env:
            - name: LOG_FORMAT
              value: json
            - name: LOG_LEVEL
              value: info
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Parsers_File  parsers.conf
    
    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Refresh_Interval  10
    
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Merge_Log           On
    
    [OUTPUT]
        Name            loki
        Match           *
        Host            loki.monitoring.svc
        Port            3100
        Labels          job=fluentbit
        Line_Format     json
  
  parsers.conf: |
    [PARSER]
        Name        json
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
```

---

## Step 670: Workshop - JSON Tools Summary

```bash
# jq: query, filter, transform
kubectl get pods -o json | jq '.items[] | {name: .metadata.name, status: .status.phase}'

# yq: YAML/JSON processor
yq eval '.spec.replicas' deployment.yaml
yq eval '.spec.replicas = 3' -i deployment.yaml
yq eval -o=json deployment.yaml  # YAML to JSON

# JSONPath in kubectl
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# kubeconform (validate against K8s schemas)
kubeconform -strict -summary deployment.yaml
kubeconform -kubernetes-version 1.28.0 -strict manifests/

# conftest (OPA policy testing)
conftest test -p policy/ deployment.yaml

# kyverno CLI
kyverno apply policy.yaml --resource deployment.yaml

# Count pods by namespace
kubectl get pods -A -o json | jq '[.items[] | .metadata.namespace] | group_by(.) | map({(.[0]): length}) | add'

# Image inventory CSV
kubectl get pods -A -o json | jq -r '.items[] | [.metadata.namespace, .metadata.name, .spec.containers[].image] | @csv'
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: json-logging-quality
  namespace: monitoring
spec:
  groups:
    - name: logging
      rules:
        - alert: HighLogDropRate
          expr: |
            rate(fluentbit_output_dropped_records_total[5m]) > 10
          for: 5m
          annotations:
            summary: "FluentBit dropping records (possible JSON parse errors)"
          labels:
            severity: warning
```

---

## 📊 สรุป Part 70

| Tool | Use Case | Example |
|------|----------|---------|
| `jq` | JSON query/transform | `kubectl get pods -o json \| jq '.items[].metadata.name'` |
| `yq` | YAML/JSON processor | `yq eval '.spec.replicas = 3' -i deploy.yaml` |
| JSONPath | kubectl output | `-o jsonpath='{.items[*].metadata.name}'` |
| JSON Schema | Validation | `kubeconform -strict manifest.yaml` |
| JSON Patch | Atomic updates | `kubectl patch --type=json` |
| JWT | SA tokens | `cat /run/secrets/kubernetes.io/serviceaccount/token` |

---

## 🔗 ต่อไป
- [Part 71: Kubernetes Operators](./part-71-kubernetes-operators.md)

---
*Part 70 | Steps 661-670 | JSON Advanced Features | Educational Content*
