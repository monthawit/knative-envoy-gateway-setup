# knative-envoy-gateway-setup
knative-envoy-gateway-setup

# KNativeServerless-With-EnvoyGateway

## 1.Install Envoy Gateway 2 Gateway  เพื่อเอาไป Config ในข้อ 3.2 

- External Gateway ที่มี Port 443 + Secret SSL 
- Local-gateway  ที่มีเฉพาะ 80 

## 2. Install the Knative Serving component

### 2.1Install the required custom resources by running the command:
```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.20.1/serving-crds.yaml
```
### 2.2 Install the core components of Knative Serving by running the command:
```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.20.1/serving-core.yaml
```
## 3. Install the Knative Gateway API:

### 3.1

```bash
kubectl apply -f https://github.com/knative-extensions/net-gateway-api/releases/download/knative-v1.20.1/net-gateway-api.yaml
```

### 3.2 Create Config Gateway to use Envoy-Gateway 

ให้แยก External & Local ให้ชัดเจน ไม่เช่นนั้นตอนสร้างมันจะได้ status Unknow, Uninitialize เนื่องจาก ต้อง Sync Local ก่อน 

```bash
kubectl edit cm config-gateway -n knative-serving
```
Edit gateway Configmap
```bash
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  external-gateways: |
    - class: gateway-01
      gateway: envoy-gateway-system/gateway-01
      service: envoy-gateway-system/gateway-01
      supported-features:
      - HTTPRouteRequestTimeout
  local-gateways: |
    - class: local-gateway-01
      gateway: envoy-gateway-system/local-gateway-01
      service: envoy-gateway-system/local-gateway-01
      supported-features:
      - HTTPRouteRequestTimeout
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"external-gateways":"- class: gateway-01\n  gateway: envoy-gateway-system/eg-external\n  service: envoy-gateway-system/gateway-01\n  supported-features:\n  - HTTPRouteRequestTimeout\n","local-gateways":"- class: gateway-01\n  gateway: envoy-gateway-system/eg-internal\n  service: envoy-gateway-system/gateway-01\n  supported-features:\n  - HTTPRouteRequestTimeout\n"},"kind":"ConfigMap","metadata":{"annotations":{},"labels":{"app.kubernetes.io/component":"net-gateway-api","app.kubernetes.io/name":"knative-serving","serving.knative.dev/release":"devel"},"name":"config-gateway","namespace":"knative-serving"}}
  creationTimestamp: "2026-01-23T05:05:42Z"
  labels:
    app.kubernetes.io/component: net-gateway-api
    app.kubernetes.io/name: knative-serving
    serving.knative.dev/release: devel
  name: config-gateway
  namespace: knative-serving
  resourceVersion: "100309228"
  uid: fbd14183-f79c-4077-8436-25500fb07e78
```

### 3.3 Configure Knative Serving to use the Knative Gateway API ingress class:

```bash
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"gateway-api.ingress.networking.knative.dev"}}'
```

## 4. Setup DNS Server 

A Record  103.x.x.x  *.apps.olsxops.com 

In k8s Cluster 
```bash
kubectl patch configmap/config-domain \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"apps.olsxops.com":""}}'
```


## 5. Disable Namespace In URL , Enable HTTPS ( Optional )

```bash
kubectl edit cm config-network -n knative-serving 
```

Add ค่าพวกนี้ลงไป 

  default-external-scheme: https
  domain-template: '{{.Name}}.{{.Domain}}'
  http-protocol: Disabled




## 6. ทดสอบ Run 

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
    #  annotations:  
    #networking.knative.dev/disableNamespaceInURL: "true"    
spec:
  template:
    spec:
      containers:
        - image: ghcr.io/knative/helloworld-go:latest
          ports:
            - containerPort: 8080
          env:
            - name: TARGET
              value: "World"
```

Check Service 

```bash
kubectl get pod,svc,ksvc,httproute -n namespace 
```

# Install optional Serving extensions

Install the components needed to support HPA-class autoscaling by running the command:

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.20.1/serving-hpa.yaml
```

