# tock-mcp-server
MCP server for T.O.C.K. web-connector

## Building the server

```bash
make build-servers
```

## Running the server

```bash
make run-tock-mcp
```

## Configuration

The server expects a YAML configuration file with the following structure:

```yaml
tock:
  # Base URL of the Tock instance (no trailing slash)
  base_url: "https://<bot-api-host>"
  # Tock namespace
  namespace: "app"
  # Bot name
  bot: "howtonet"
  # Web-connector identifier
  connector: "web"
  # User ID sent to Tock with every request
  user_id: "mcp-user-001"
  # Optional extra HTTP headers forwarded to the Tock web-connector.
  # The value is the default used when the MCP caller does not provide one.
  # Leave empty ("") to make the header optional with no default.
  # extra_headers:
  #  X-Toki-Origin: "github-copilot"   # always sent; can be overridden by the caller
  #  X-Toki-Filter: ""                 # no default — provided by the caller if needed

server:
  # HTTP listen address of the MCP server
  addr: ":8083"
```

## Docker image (scratch)

The image is built with a `scratch` runtime and expects a configuration file available at `/etc/tock/config.yaml`.

Build locally:

```bash
docker build -t tock-mcp-server:local .
```

Run locally with a mounted config file:

```bash
docker run --rm -p 8083:8083 \
  -v "$(pwd)/config.yaml:/config/config.yaml:ro" \
  tock-mcp-server:local
```

## Deploying on Kubernetes (ConfigMap)


### with manifest
The container does not embed `config.yaml`, so each environment can mount its own ConfigMap.

Use the provided manifest:

```bash
kubectl apply -f k8s/tock-mcp-server.yaml
```

Example (extract):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tock-mcp-config
data:
  config.yaml: |
    tock:
      base_url: "https://demo.tock.ai"
      namespace: "sacquatella"
      bot: "howtonet"
      connector: "howtonetweb"
      user_id: "mcp-user-001"
    server:
      addr: ":8083"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tock-mcp-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tock-mcp-server
  template:
    metadata:
      labels:
        app: tock-mcp-server
    spec:
      containers:
        - name: tock-mcp-server
          image: ghcr.io/sacquatella/tock-mcp-server:<tag>
          args: ["-config", "/config/config.yaml", "-addr", ":8083"]
          volumeMounts:
            - name: config
              mountPath: /config
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: tock-mcp-config
```

### With Helm chart

A Helm chart is available in `k8s/chart/` for easier deployment and configuration management.

```bash
helm install tock-mcp ./k8s/chart -f values.yaml
```

```yaml
config:
  tock:
    base_url: "https://tock.my-company.com"
    namespace: "app"
    bot: "my-bot"
    connector: "web"
    user_id: "mcp-prod"

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: tock-mcp.my-company.com
      paths:
        - path: /mcp
          pathType: Prefix
  tls:
    - secretName: tock-mcp-tls
      hosts:
        - tock-mcp.my-company.com
```
