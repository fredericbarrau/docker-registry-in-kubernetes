# Docker registry in Kubernetes

"Simple" example of how to run a docker registry in Kubernetes, for local developmennt, with TLS & basic auth.

This has been tested on MacOS with colima. Should work as-is on a minikube setup as well.

Beware that TLS can be a PITA to configure on the docker side though, and you might need the access to your docker **daemon** config to make it work.

**Auth:** docker-user:mYp4s50rD§

## Deploy

```bash
kubectl create ns registry
kubectl config set-context --current --namespace=registry
kubectl apply -f local-registry-deployment.yaml
```

## Config the docker daemon

Get the cert authority of your newly created docker registry:

```bash
# Get the cert authority of your docker registry (self-signed cert)
kubectl get secret docker-registry-tls -o jsonpath="{.data.tls\.crt}" | base64 --decode > ca.crt

# => Put this file on your docker daemon config folder (locally if you are on linux, in a dedicated VM if you are on Mac/Windows)

# SSH to your docker host (if a VM) first, and `sudo -i` then:
mkdir -p /etc/docker/certs.d/docker-registry/
# From your local machine:
scp ca.crt root@<host>:/etc/docker/certs.d/docker-registry/ca.crt
# Or copy & paste the content of the ca.crt file grabbed from k8s above into the ca.crt file in your VM, I won't judge you.
```

See [here](https://distribution.github.io/distribution/about/insecure/#use-self-signed-certificates) for more information on using docker with self-signed cert on a docker registry.

## Configure your /etc/hosts

```bash
# Add this in your /etc/hosts:
<IP of your kubernetes node> docker-registry
```

## (optional) Configure your docker host (if a VM)

```bash
# Add this in your /etc/hosts:
<IP of your kubernetes node> docker-registry
```

## Push & pull from your workstation

```bash
docker pull alpine:lastest
docker login docker-registry:30000
# User: docker-user
# Password: mYp4s50rD§
# Tag alpine image to your local registry
docker tag alpine:latest docker-registry:30000/alpine:latest
# Then push to check if auth & access is OK
docker push docker-registry:30000/alpine:latest
```

## Use your registry from within the kubernetes cluster

* Add a coreDNS entry for your registry in the kube-system namespace, to resolve the registry name to the service IP. Create a ConfigMap with the following content and apply it to your cluster:

```yaml
# File: local-registry-coredns.yaml
# CoreDNS custom config
apiVersion: v1
data:
  config-registry.override: |
    # Resolve the name of the registry to the service IP
    rewrite name docker-registry docker-registry.registry.svc.cluster.local
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
```

  ```bash
  kubectl apply -n kube-system -f local-registry-coredns.yaml
  ```

**For each namespace where you want to use the registry:**

* Copy the registry-secret with the registry's auth, and patch the namespace's default service account to authenticate to your registry:

  ```bash
  # Copy the registry-secret to your namespace from the default (created by the registry deployment)
  kubectl -n registry get secret registry-secret -o yaml | sed '/namespace:/d' | kubectl apply -n <NAMESPACE> -f -
  # Patch the default service account to use the registry secret:
  kubectl patch -n <NAMESPACE> serviceaccount default -p '{"imagePullSecrets": [{"name": "registry-secret"}]}'
  ```

* Then, in your podSpec, you can use the registry to fetch your images: `docker-registry:5000/alpine:latest`

**PodSpec example:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  namespace: <NAMESPACE>
spec:
  containers:
  - name: my-app
    # Reference the internal registry
    image: docker-registry:5000/my-app:latest
    ports:
    - containerPort: 80
```
