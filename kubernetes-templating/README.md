```
# install nginx-ingress
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && \
helm repo update && \
helm install ingress-nginx ingress-nginx/ingress-nginx

# install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.12.0
kubectl apply -f ./cert-manager/issuer.yaml

# install chartmuseum
kubectl create ns chartmuseum
helm repo add chartmuseum https://chartmuseum.github.io/charts
helm repo update chartmuseum
helm upgrade --install chartmuseum-release chartmuseum/chartmuseum  --wait  --namespace=chartmuseum   --version 3.9.3 -f chartmuseum/values.yaml
helm ls -n chartmuseum

# install harbor
kubectl create ns harbor
helm repo add harbor https://helm.goharbor.io
helm repo update harbor
helm upgrade --install harbor harbor/harbor --wait --namespace=harbor --version=1.12.0 -f ./harbor/values.yaml

# load images to harbor
helm package frontend
helm package hipster-shop
helm plugin install https://github.com/chartmuseum/helm-push
helm registry login https://harbor.158.160.99.232.sslip.io/ -u admin -p Harbor12345
helm push frontend-0.1.0.tgz oci://harbor.158.160.99.232.sslip.io//otus-kuber
helm push hipster-shop-0.1.0.tgz oci://harbor.158.160.99.232.sslip.io//otus-kuber
```
