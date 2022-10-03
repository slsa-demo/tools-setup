# tools-setup

## Setup Tekton pipeline
```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml 
kubectl get pods --namespace tekton-pipelines --watch
```
## Setup Tekton chains
```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/chains/latest/release.yaml
```
