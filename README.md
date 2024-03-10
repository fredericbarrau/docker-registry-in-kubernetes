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

## Push & pull

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
