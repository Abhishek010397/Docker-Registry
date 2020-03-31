# Docker-Registry
Docker-Registry
On MacOS and Linux:

curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash

Install tiller
Create RBAC permissions for tiller
kubectl -n kube-system create sa tiller \
  && kubectl create clusterrolebinding tiller \
  --clusterrole cluster-admin \
  --serviceaccount=kube-system:tiller

Install the server-side Tiller component on your cluster
helm init --skip-refresh --upgrade --service-account tiller


Now wait for tiller to become ready:
kubectl rollout status -n kube-system deploy/tiller-deploy

output
deployment "tiller-deploy" successfully rolled out


Add an IngressController if not using k3s
If you're not using k3s, then install Nginx Ingress instead:

helm install stable/nginx-ingress --name nginxingress --set rbac.create=true,controller.hostNetwork=true,controller.daemonset.useHostPort=true,dnsPolicy=ClusterFirstWithHostNet,controller.kind=DaemonSet



Install cert-manager
You can now install cert-manager, the version used is v0.9.1.

# Install the CustomResourceDefinition resources separately
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/deploy/manifests/00-crds.yaml

# Create the namespace for cert-manager
kubectl create namespace cert-manager

# Label the cert-manager namespace to disable resource validation
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install \
  --name cert-manager \
  --namespace cert-manager \
  --version v0.9.1 \
  jetstack/cert-manager



Create a ClusterIssuer
The way that cert-manager issues certificates is through an Issuer. The Issuer can issue certificates for the namespace it is created in, but a ClusterIssuer can create certificates for any namespace, so that's the one we will use today.

Save issuer.yaml:

apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: default
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: user@example.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: traefik
Edit the file:
Edit the line: email: user@example.com.

If using Nginx instead of k3s and Traefik, then edit the following:

    solvers:
    - http01:
        ingress:
          class: nginx
Then run kubectl apply -f issuer.yaml.
kubectl describe clusterissuer/letsencrypt-prod


Configure DNS
For this tutorial a domain on-k3s.dev was purchased from Google Domains to show a full worked example.


Install the registry
At this stage we can install the registry, but we are going to install it without persistence. If you need persistence see the appendix for how to do this.

Save the following as install-registry.sh:

export SHA=$(head -c 16 /dev/urandom | shasum | cut -d " " -f 1)
export USER=admin

echo $USER > registry-creds.txt
echo $SHA >> registry-creds.txt

docker run --entrypoint htpasswd registry:2 -Bbn admin $SHA > ./htpasswd

helm install stable/docker-registry \
  --name private-registry \
  --namespace default \
  --set persistence.enabled=false \
  --set secrets.htpasswd=$(cat ./htpasswd)


You will need to have docker installed and ready for this step. If it's not started, then start it up now.

Then run the script:

chmod +x install-registry.sh
./install-registry.sh
