## GitOps with Argo Project
Setting up a Kubernetes Native Continuous Deployment pipeline, following the 
[GitOps](https://www.gitops.tech/) principles, using the 
[ArgoProject Tools](https://argoproj.github.io/)

Reference/Tutorials (by [DevOpsToolkit](https://www.youtube.com/c/DevOpsToolkit)):
- [Argo Workflows Tutorial](https://www.youtube.com/watch?v=UMaivwrAyTA)
- [Argo CD Tutorial](https://www.youtube.com/watch?v=vpWQeoaiRM4)
- [Argo Rollouts Tutorial](https://www.youtube.com/watch?v=84Ky0aPbHvY)


## Setup

Create a cluster with [minikube](https://minikube.sigs.k8s.io/docs/) (as a vm):
```
minikube start --vm=true
```

Enable nginx ingress (only possible if cluster created as a vm):
```
minikube addons enable ingress
```

Verify that the NGINX Ingress controller is running (this can take up to a minute):
```
kubectl get pods -n ingress-nginx
```


Install argo cli (optional):
```
# Download the binary
curl -sLO https://github.com/argoproj/argo-workflows/releases/download/v3.1.9/argo-darwin-amd64.gz

# Unzip
gunzip argo-darwin-amd64.gz

# Make binary executable
chmod +x argo-darwin-amd64

# Move binary to path
mv ./argo-darwin-amd64 /usr/local/bin/argo

# Test installation
argo version
```

## Install Argo Workflows and Argo Events

Reference:
- https://argoproj.github.io/argo-events/installation/
- https://argoproj.github.io/argo-workflows/quick-start/

### Creating resources:

Create a namespace:
```
kubectl create namespace argo-events
```

Install argo-events:
```
kubectl apply -f argo/argo-events-namespace-install.yaml
```

or

```
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/namespace-install.yaml
```

Install argo-events event-bus:
```
kubectl apply -n argo-events -f argo/event-bus-native.yaml
```

or

```
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/eventbus/native.yaml
```

Install argo-workflows:
```
kubectl apply -n argo-events -f argo/argo-workflows-quick-start-postgres.yaml
```

or

```
kubectl apply -n argo-events -f https://raw.githubusercontent.com/argoproj/argo-workflows/stable/manifests/quick-start-postgres.yaml
```

Expose the service on localhost:
```
kubectl -n argo-events port-forward deployment/argo-server 2746:2746
```

Check https://localhost:2746 for ArgoWorkflows UI.

### Create a push-based gitops pipeline

Create dockerhub credentials:

```
export REGISTRY_SERVER=https://index.docker.io/v1/
export REGISTRY_USER=YOUR_DOCKER_HUB_USERNAME
export REGISTRY_PASS=YOUR_DOCKER_HUB_PASSWORD
export REGISTRY_EMAIL=YOUR_DOCKER_HUB_EMAIL
```

```
kubectl --namespace argo-events \
    create secret \
    docker-registry docker-registry-credentials \
    --docker-server=$REGISTRY_SERVER \
    --docker-username=$REGISTRY_USER \
    --docker-password=$REGISTRY_PASS \
    --docker-email=$REGISTRY_EMAIL
```

Replace the variables:
- `{BASE64_ENCODED_GITHUB_ACCESS_TOKEN_HERE}` 
- `{BASE64_ENCODED_GITHUB_USERNAME_HERE}` and
- `{BASE64_ENCODED_GITHUB_EMAIL_HERE}`

in `argo-workflow/github-access-secret.yaml` (including the `{ }`) with your base64 encoded github 
access token and other credentials (username and email). To do so, follow the instructions 
[here](https://docs.github.com/en/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token).

`base64` encode command:
```
echo -n <api-token-key> | base64
```

Check the Argo Events docs [github example](https://argoproj.github.io/argo-events/eventsources/setup/github/) for more
details.

Deploy the full pipeline:
```
kubectl -n argo-events apply -f argo-workflow/
```

Expose the event source port:
```
kubectl -n argo-events port-forward $(kubectl -n argo-events get pod -l eventsource-name=github-event-source -o name) 12000:12000
```

If you don't have `ngrok` installed, [install it and create an account](https://ngrok.com/download).

Expose your localhost to the world:
```
./ngrok http 12000
```

Add the ngrok endpoint as a repo webhook (repo Settings -> Webhooks -> Add webhook):
```
https://{ngrok_endpoint}.ngrok.io/argo-test
```

No you can push updates and check the pipeline execution. 


## Install Argo CD

[Reference docs.](https://argo-cd.readthedocs.io/en/stable/getting_started/)

Create a namespace:
```
kubectl create namespace argocd
```

Install argo-events:
```
kubectl apply -n argocd -f argo/argo-cd-core-install.yaml
```

or

```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/core-install.yaml
```

Change the argocd-server service type to LoadBalancer:
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Wait to see the deployments up and running:
```
watch kubectl get pods -n argocd
```

or:
```
watch kubectl get deployments -n argocd
```

Once the servers are all ready, run:
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

To login into ArgoCD UI username and passwords are needed.
To get the password for the `admin` username, run:
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Check https://localhost:8080 for ArgoCD UI and login the username: `admin` and the password.


### Modify the pipeline to be a pull-based gitops pipeline

Apply the ArgoCD project and the ArgoCD application for the staging environment:
```
kubectl apply -f argo-cd-project.yaml
```

and the application:
```
kubectl apply -f argo-cd-app.yaml
```

This should be the only time applying manually kubernetes manifests. Every new update
should be picked and applied by ArgoCD.

Hooray!!!


## Argo Rollouts

### Install Argo Rollouts

Create a namespace and apply the manifests:
```
kubectl create namespace argo-rollouts
```

```
kubectl apply -n argo-rollouts -f argo/argo-rollouts-install.yaml
```

Install the 
[argo rollouts plugin](https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation) for `kubectl`:
```
brew install argoproj/tap/kubectl-argo-rollouts
```

Expose the Argo Rollouts dashboard service:
```
kubectl argo rollouts dashboard --namespace prod
```

Check the dashboard:
http://localhost:3100


### Create a canary deployment
```
kubectl create ns prod
```

```
kubectl apply -f deployment-canary/
```

```
kubectl argo rollouts -n prod get rollout flask-api-canary-rollout
```

Get the cluster entry IP address:
```
minikube ip
```

Under `/etc/hosts` add a new entry with the format:
```
192.168.64.2    argo.canary
```

*Note replace the IP with the IP returned by the previous command!

Create a new argo rollouts revision by updating the currently deployed image:
```
NEW_IMAGE_VERSION=NEW_IMAGE_VERSION_HERE
kubectl argo rollouts -n prod set image flask-api-canary-rollout flask-api=$NEW_IMAGE_VERSION
```

Continue the rollout to 100% traffic to the new version (the rollout config is set to pause
the rollout when 50% of the traffic is redirected to the new version) :
```
kubectl argo rollouts -n prod promote flask-api-canary-rollout
```