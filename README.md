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
## Setup CNBs with Tekton 
https://buildpacks.io/docs/tools/tekton/

Setup CNB and Gitclone tasks
```
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/master/task/buildpacks/0.3/buildpacks.yaml
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/master/task/git-clone/0.4/git-clone.yaml

```

Setup resources and pipeline
```
kubectl apply -f tools-setup/cnb/resources.yml 
kubectl apply -f tools-setup/cnb/pipeline.yml
```
### With Dockerhub username/password
```
kubectl create secret docker-registry docker-user-pass \
    --docker-username=msathetech \
    --docker-password=<DockerHubPassword> \
    --docker-server=https://index.docker.io/v1/ \
    --namespace default

kubectl apply -f tools-setup/cnb/sa.yaml
kubectl apply -f tools-setup/cnb/run.yaml
```

### With GKE workload identity 
```
kubectl apply -f tools-setup/cnb/sa-with-wi.yaml
kubectl annotate serviceaccount \
  tekton-sa \
  iam.gke.io/gcp-service-account=tekton-sa@$PROJECT_ID.iam.gserviceaccount.com

gcloud iam service-accounts add-iam-policy-binding \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[default/tekton-sa]" \
    tekton-sa@$PROJECT_ID.iam.gserviceaccount.com

kubectl apply -f tools-setup/cnb/run-with-wi.yaml
```


## Tekton with Artifact Registry 
https://g3doc.corp.google.com/company/teams/tekton/codelabs/clone-build-push.md?cl=head

## Tekton Chains with GCP 
https://g3doc.corp.google.com/cloud/build/cbes/g3doc/getting-started/chains-tutorial.md?cl=head
```
gcloud container clusters create ${CLUSTER_NAME} \
       --machine-type=${MACHINE_TYPE} \
       --num-nodes=1 \
       --enable-autoscaling --min-nodes "1" --max-nodes "5" \
       --region=${GCP_REGION} \
       --project=${PROJECT_ID} \
       --cluster-version=${GKE_VERSION} \
       --release-channel=${RELEASE_CHANNEL} \
       --enable-ip-alias \
       --image-type "COS_CONTAINERD" \
       --maintenance-window-start "2022-09-25T04:00:00Z" \
       --maintenance-window-end "2022-09-26T04:00:00Z" \
       --maintenance-window-recurrence "FREQ=WEEKLY;BYDAY=SU" \
       --workload-pool=${PROJECT_ID}.svc.id.goog \
       --scopes=cloud-platform

gcloud container clusters get-credentials ${CLUSTER_NAME} --region ${GCP_REGION} --project ${PROJECT_ID}

kubectl create serviceaccount $KSA_NAME \
  --namespace $NAMESPACE

gcloud iam service-accounts create $GSA_NAME \
  --project=$PROJECT_ID

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:$GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
--role "roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:$GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
--role "roles/cloudkms.cryptoOperator"

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:$GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
--role "roles/cloudkms.viewer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:$GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
--role "roles/containeranalysis.admin"

gcloud iam service-accounts add-iam-policy-binding $GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com \
--role roles/iam.workloadIdentityUser \
--member "serviceAccount:$PROJECT_ID.svc.id.goog[$NAMESPACE/$KSA_NAME]"

kubectl annotate serviceaccount $KSA_NAME \
  --namespace $NAMESPACE \
  iam.gke.io/gcp-service-account=$GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com

kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl get pods -n tekton-pipelines

kubectl apply --filename https://storage.googleapis.com/tekton-releases/chains/latest/release.yaml
kubectl get pods -n tekton-chains

kubectl patch configmap chains-config -n tekton-chains -p='{"data":{"artifacts.taskrun.format": "in-toto"}}'
kubectl patch configmap chains-config -n tekton-chains -p='{"data":{"artifacts.oci.format": "simplesigning"}}'
```
### Key Ring
```
gcloud kms keyrings create ${KEYRING} \
    --location ${GCP_REGION}

gcloud kms keys create ${SIGNING_KEY} \
    --keyring ${KEYRING} \
    --location ${GCP_REGION} \
    --purpose "asymmetric-signing" \
    --default-algorithm "rsa-sign-pkcs1-2048-sha256"

kubectl patch configmap chains-config -n tekton-chains -p='{"data":{"signers.kms.kmsref": "'"$KEY_REF"'"}}'
```

### Tekton Chain Controller SA 
```
gcloud iam service-accounts add-iam-policy-binding $GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:$PROJECT_ID.svc.id.goog[tekton-chains/tekton-chains-controller]"

kubectl annotate serviceaccount tekton-chains-controller \
    --namespace tekton-chains \
    iam.gke.io/gcp-service-account=$GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com
```

### Setup pipelines
kubectl apply -f https://raw.githubusercontent.com/tektoncd/chains/main/examples/kaniko/kaniko.yaml