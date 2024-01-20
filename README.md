Ingress-Setup


az network vnet check-ip-address --name Docker-project-SG-vnet -g Docker-Project --ip-address 10.90.1.180


kubectl create namespace ingress-basi

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx


// Internal Nginx 

  helm install ingress-nginx ingress-nginx/ingress-nginx \
    --version 4.7.1 \
    --namespace ingress-basic \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.service.loadBalancerIP=10.90.1.180 \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-internal"=true \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
    --set controller.admissionWebhooks.patch.image.digest="" \
    --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux


---- internal-nginx ------


apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: internal-nginx
  namespace: ingress-basic
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /middleware
        pathType: Prefix
        backend:
          service:
            name: middleware-svc
            port:
              number: 8080  
      - path: /mobility
        pathType: Prefix
        backend:
          service:
            name: mobility
            port:
              number: 8081


// External Nginx 

helm install publicingress ingress-nginx/ingress-nginx \
  --version 4.1.3 \
  --set controller.replicaCount=2 \
  --namespace ingress-basic \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.service.externalTrafficPolicy=Cluster \
  --set controller.ingressClassResource.name="publicingress" \
  --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.ingressClass="publicingress" \
  --set controller.ingressClassResource.controllerValue="k8s.io/publicingress" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
  --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux

  ---External-nginx ------

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: external-nginx
  namespace: ingress-basic
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: publicingress
  rules:
  - http:
      paths:
      - path: /middleware
        pathType: Prefix
        backend:
          service:
            name: middleware-svc
            port:
              number: 8080  
      - path: /mobility
        pathType: Prefix
        backend:
          service:
            name: mobility
            port:
              number: 8081