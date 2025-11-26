# Deploy nginx with Kubernetes and Cloudflare Tunnel

## INDEX

- [0. Requirements](https://github.com/pkthom/deploy-nginx-with-k8s-cloudflare#0-requirements)
- [1. Install ingress-nginx and nginx test app (with Helm)](https://github.com/pkthom/deploy-nginx-with-k8s-cloudflare#1-install-ingress-nginx-and-nginx-test-app-with-helm)
- [2. Create a Cloudflare Tunnel](https://github.com/pkthom/deploy-nginx-with-k8s-cloudflare#2-create-a-cloudflare-tunnel)
- [3. Create namespace and Secret for cloudflared](https://github.com/pkthom/deploy-nginx-with-k8s-cloudflare#3-create-namespace-and-secret-for-cloudflared)
- [4. Apply cloudflared-deploy.yaml](https://github.com/pkthom/deploy-nginx-with-k8s-cloudflare#4-apply-cloudflared-deployyaml)
- [5. Test](https://github.com/pkthom/deploy-nginx-with-k8s-cloudflare#5-test)

## 0. Requirements
You must already have:

- a Kubernetes cluster
- `kubectl` and `helm` installed
- a domain managed by **Cloudflare DNS**
- a **Cloudflare Tunnel** and a **Tunnel token**

## 1. Install ingress-nginx and nginx test app (with Helm)

Example (you can change this):

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=ClusterIP
```


Then create a simple nginx app with Helm:
```
kubectl create namespace test

helm create test-nginx
cd test-nginx
```

Edit values.yaml:
```
ingress:
  enabled: true
  className: "nginx"
  annotations: {}
  hosts:
    - host: your-domain.example.com   # <-- change to your domain
```

Install:
```
helm install test-nginx ./test-nginx -n test
kubectl get ingress -n test
```
You should see your-domain.example.com in the HOSTS column.


## 2. Create a Cloudflare Tunnel

In the Cloudflare dashboard:
1. Go to Zero Trust → Networks → Connectors
2. Create a new tunnel
3. Choose cloudflared as connector
4. Copy the Tunnel token
(from the docker run ... --token <TOKEN> example)

Add a Published application route:
- Hostname: your-domain.example.com
- Service type: HTTP
- Service URL:
http://ingress-nginx-controller.ingress-nginx.svc.cluster.local:80

This sends traffic for your domain to the ingress-nginx-controller Service in the cluster.

## 3. Create namespace and Secret for cloudflared

In your cluster:
```
kubectl create namespace cloudflare-tunnel

kubectl -n cloudflare-tunnel create secret generic cloudflared-token \
  --from-literal=TUNNEL_TOKEN='YOUR_TUNNEL_TOKEN_HERE'
```
Replace YOUR_TUNNEL_TOKEN_HERE with the real token from Cloudflare.

## 4. Apply cloudflared-deploy.yaml

Clone this repo and run:
```
kubectl apply -f cloudflared-deploy.yaml
kubectl -n cloudflare-tunnel get pods
```

You should see something like:
```
cloudflared-xxxxx   1/1   Running
```

Optionally check logs:
```
kubectl -n cloudflare-tunnel logs deploy/cloudflared | head
```

In the Cloudflare dashboard, the tunnel status should be HEALTHY.

## 5. Test

Open your browser:
```
https://your-domain.example.com
```

You should see the nginx test page running inside your Kubernetes cluster.

Traffic flow:
```
Browser → Cloudflare → Tunnel → cloudflared Pod → ingress-nginx → nginx app
```
