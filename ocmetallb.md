# OpenShift Routes - HTTP Only (Two VIPs)

## Architecture

```
http://erp.ow.com  -->  VIP 1 (MetalLB)  -->  HAProxy Router  -->  erp-svc  -->  erp pods
http://hr.ow.com   -->  VIP 2 (MetalLB)  -->  HAProxy Router  -->  hr-svc   -->  hr pods
```

---

## Step 1 - Install MetalLB

```bash
oc apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

Wait for MetalLB pods to be ready:

```bash
oc wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s
```

Create IPAddressPool with two VIPs:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: workshop-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.130.100/32
  - 192.168.130.101/32
```

```bash
oc apply -f ipaddresspool.yaml
```

Create L2Advertisement:

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: workshop-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - workshop-pool
```

```bash
oc apply -f l2advertisement.yaml
```

```
VIP 1 = 192.168.130.100  --> erp.ow.com
VIP 2 = 192.168.130.101  --> hr.ow.com
```

---

## Step 2 - Create Namespace

```bash
oc new-project workshop
```

---

## Step 3 - Deploy ERP

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: erp
  namespace: workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: erp
  template:
    metadata:
      labels:
        app: erp
    spec:
      containers:
      - name: erp
        image: nginxinc/nginx-unprivileged
        ports:
        - containerPort: 8080
```

```bash
oc apply -f erp-deployment.yaml
```

Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: erp-svc
  namespace: workshop
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.130.100
spec:
  selector:
    app: erp
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

```bash
oc apply -f erp-svc.yaml
```

---

## Step 4 - Deploy HR

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hr
  namespace: workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hr
  template:
    metadata:
      labels:
        app: hr
    spec:
      containers:
      - name: hr
        image: nginxinc/nginx-unprivileged
        ports:
        - containerPort: 8080
```

```bash
oc apply -f hr-deployment.yaml
```

Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hr-svc
  namespace: workshop
  annotations:
    metallb.universe.tf/loadBalancerIPs: 192.168.130.101
spec:
  selector:
    app: hr
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

```bash
oc apply -f hr-svc.yaml
```

---

## Step 5 - Create Route for ERP

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-erp
  namespace: workshop
spec:
  host: erp.ow.com
  to:
    kind: Service
    name: erp-svc
    weight: 100
  port:
    targetPort: 80
```

```bash
oc apply -f route-erp.yaml
```

---

## Step 6 - Create Route for HR

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: route-hr
  namespace: workshop
spec:
  host: hr.ow.com
  to:
    kind: Service
    name: hr-svc
    weight: 100
  port:
    targetPort: 80
```

```bash
oc apply -f route-hr.yaml
```

---

## Step 7 - Verify

```bash
oc get ipaddresspool -n metallb-system
oc get svc -n workshop
oc get routes -n workshop
oc get pods -n workshop
```

Expected svc output:

```
NAME      TYPE           CLUSTER-IP   EXTERNAL-IP       PORT(S)
erp-svc   LoadBalancer   10.x.x.x     192.168.130.100   80/TCP
hr-svc    LoadBalancer   10.x.x.x     192.168.130.101   80/TCP
```

Expected routes output:

```
NAME        HOST/PORT     SERVICES   PORT
route-erp   erp.ow.com    erp-svc    80
route-hr    hr.ow.com     hr-svc     80
```

---

## Step 8 - DNS

Add to /etc/hosts if DNS is not configured:

```bash
echo "192.168.130.100 erp.ow.com" | sudo tee -a /etc/hosts
echo "192.168.130.101 hr.ow.com"  | sudo tee -a /etc/hosts
```

---

## Step 9 - Test

```bash
curl http://erp.ow.com
curl http://hr.ow.com
```

---

## Troubleshooting

Service stuck in Pending (no IP assigned):

```bash
oc describe svc erp-svc -n workshop
oc get ipaddresspool -n metallb-system
oc get pods -n metallb-system
```

503 error - check pods and endpoints:

```bash
oc get pods -n workshop
oc get endpoints erp-svc -n workshop
oc get endpoints hr-svc -n workshop
```
