### Tutorial

We'll first install helm, then tiller, then Kubernetes users can add [Nginx](https://nginx.com/) in *Host* mode and k3s users can skip this because they will be using [Traefik](https://traefik.io). After that we'll add cert-manager and an Issuer to obtain certificates, followed by the registry. After everything is installed, we can then make use of our registry using the password created during the tutorial. You'll finish off by testing everything end-to-end, and if you get stuck, there are some helpful tips on how to troubleshoot.

Some components are installed in their own namespaces such as cert-manager, all others will be installed into the `default` namespace. You can control the namespace with `kubectl get --namespace/-n NAME` or `kubectl get --all-namespaces/-A`.

There will also be some ways to take the tutorial further in the appendix.

#### Install the helm CLI/client

Instructions for latest Helm install

* On MacOS and Linux:

      curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash

* Or via Homebrew on Mac:

      brew install kubernetes-helm

For Windows users, go to [helm.sh](https://helm.sh).

#### Install tiller

* Create RBAC permissions for tiller

```sh
kubectl -n kube-system create sa tiller \
  && kubectl create clusterrolebinding tiller \
  --clusterrole cluster-admin \
  --serviceaccount=kube-system:tiller
```

* Install the server-side Tiller component on your cluster

```sh
helm init --skip-refresh --upgrade --service-account tiller
```

> Note: this step installs a server component in your cluster. It can take anywhere between a few seconds to a few minutes to be installed properly. You should see tiller appear on: `kubectl get pods -n kube-system`.

* Now wait for tiller to become ready:

```sh
kubectl rollout status -n kube-system deploy/tiller-deploy

deployment "tiller-deploy" successfully rolled out
```

#### Your built-in IngressController with k3s

Given that neither Civo's k3s service nor k3sup offer a cloud LoadBalancer, we need to use an IngressController. Fortunately k3s comes with one called [Traefik](https://traefik.io/).

For k3s, don't install an IngressController, you already have one, skip ahead.

#### Add an IngressController if not using k3s

**If you're not using k3s**, then install Nginx Ingress instead:

```
helm install stable/nginx-ingress --name nginxingress --set rbac.create=true,controller.hostNetwork=true,controller.daemonset.useHostPort=true,dnsPolicy=ClusterFirstWithHostNet,controller.kind=DaemonSet
```

#### Install cert-manager

You can now install cert-manager, the version used is v0.9.1.

```sh
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

```

See also: [cert-manager v0.9.0 docs](https://docs.cert-manager.io/en/release-0.9/)

#### Create a ClusterIssuer

The way that cert-manager issues certificates is through an [Issuer](https://docs.cert-manager.io/en/release-0.9/tutorials/acme/http-validation.html). The `Issuer` can issue certificates for the namespace it is created in, but a `ClusterIssuer` can create certificates for any namespace, so that's the one we will use today.

Save `issuer.yaml`:

```yaml
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
```

* Edit the file:

Edit the line: `email: user@example.com`.

If using Nginx instead of k3s and Traefik, then edit the following:

```
    solvers:
    - http01:
        ingress:
          class: nginx
```

Then run `kubectl apply -f issuer.yaml`.

> Note you may receive an error, if you do then wait 1-2 minutes and try again whilst cert-manager registers itself

You can check the status of your issuer like this:

```
kubectl describe clusterissuer/letsencrypt-prod
```

Look for it to become `Ready`.

#### Configure DNS

For this tutorial a domain `on-k3s.dev` was purchased from Google Domains to show a full worked example.

![](/images/buy-dns.png)

Once you have purchased your domain, you need to point the DNS records at the hosts in the k3s cluster where Nginx is going to be listening on port 80 (HTTP) and port 443 (HTTPS/TLS).

![](/images/add-dns.png)

You can find your IP addresses with the Civo UI, or by typing in `civo instance ls` through the CLI.

#### Install the registry

At this stage we can install the registry, but we are going to install it without persistence. If you need persistence see the appendix for how to do this.

Save the following as `install-registry.sh`:

```sh
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
```

You will need to have `docker` installed and ready for this step. If it's not started, then start it up now.

Then run the script:

```
chmod +x install-registry.sh
./install-registry.sh
```

It will install the Docker registry from [the docker-registry](https://github.com/helm/charts/tree/master/stable/docker-registry) chart.

Later, when you want to use your registry you can find your username and password in the `registry-creds.txt` file.
