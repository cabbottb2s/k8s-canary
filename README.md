## Kubernetes Deployment Examples

This repository offers a simplistic [Go app](app/app.go) along with resources useful for deploying the app to [Kubernetes](https://kubernetes.io).

The Go application exposes 2 endpoints:

 * `/health`: Responds with HTTP 200, useful for talking to Kubernetes [Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/).
 * `/version`: Exposes the version number of the application, which is set as a [constant](app/app.go#L8).


## Instructions

### Setup (Option 1 - Minikube)
```
minikube start --memory 4096
eval $(minikube docker-env)
```
### Setup (Option 2 - GCE)
```
unset ${!DOCKER_*}
kubectl create namespace canary
kubectl config set-context $(kubectl config current-context) --namespace=canary
docker login
```
### Create Load Balancer
```
kubectl apply -f k8s/lb/app-lb.yml
kubectl get services

# Get service URL
export SVC_URL=$(minikube service app-lb --url)
#or
export SVC_URL=http://$(kubectl get service gateway-service-lb -o jsonpath="{.status.loadBalancer.ingress[0].ip}")

```

### Deploy 1.0 to prod
```
nano $PROJECT_ROOT/k8s/app-production.yml                                     # set version to 1.0
GOOS=linux GOARCH=amd64 go build -tags netgo -o app
export app_version=1.0
docker build --build-arg version=$app_version -t cabbottbridge2/app:$app_version .
docker push cabbottbridge2/app:$app_version                                          ## GCE only
kubectl apply -f k8s/lb/app-production.yml
kubectl rollout status deployment/kubeapp-production   ## informational
kubectl get events -w                                  ## informational

curl -v ${SVC_URL}/health
curl -v ${SVC_URL}/version
```
### Deploy 2.0 to prod-canary
```
GOOS=linux GOARCH=amd64 go build -tags netgo -o app
export app_version=2.0
docker build --build-arg version=$app_version -t cabbottbridge2/app:$app_version .
docker push cabbottbridge2/app:$app_version
kubectl apply -f k8s/lb/app-canary.yml
kubectl get pods
SVC_URL=$(minikube service app-lb --url)
for i in `seq 1 10`; do curl $SVC_URL/version; sleep 1; done
```
### Upgrade prod to 2.0
```
nano $PROJECT_ROOT/k8s/app-production.yml  # set version to 2.0
kubectl apply -f k8s/lb/app-production.yml
for i in `seq 1 10`; do curl $SVC_URL/version; sleep 1; done
```
### Decommission prod-canary
```
kubectl delete deployment/kubeapp-canary
kubectl get pods
```
