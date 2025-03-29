---
authors:
- Hugo Authors
date: "2024-10-01"
excerpt: How to deploy Apache Airflow on Google Cloud Platform
hero: /images/on_airflow_prod/cover-img.png
title: On Airflow in Production with GCP
---

A while ago, I wrote a [post](https://fishwongy.github.io/post/20240609_airflow_pt1/) about deploying Apache Airflow on using Kind and integrating it with Google Cloud Platform. Today, I want to dive deeper into deploying Apache Airflow as a production level application on Google Cloud Platform.

In this post, we will cover the following topics:

1 - Setting up GKE

2 - Creating node pool for scaling

3 - Deploying Airflow to GKE with Helm

4 - Setting up Ingress and DNS for Airflow server

5 - Setting up IAP and Cloud Armor for security

6 - Enable git-sync for DAGs

7 - Integrate with GCP service account that has sufficient permissions

8 - Scaling Airflow by adjusting CPU and memory requests

9 - Adding extra libraries to Airflow image

10 - Upgrading Airflow version


<u><b>
    <p style="font-size:20pt ">
      Setting Up GKE
    </p>
</b></u>

In my last [post](https://fishwongy.github.io/post/20240901_gcp_staticip/), I have talked about setting up a VPC and Cloud NAT for whitelisting IP addresses. To apply our knowledge, in this post, we will be setting up a GKE cluster that is within the VPC and has Cloud NAT enabled.

```powershell
gcloud container clusters create airflow-standard-cluster \
    --region us-central1 \
    --network my-gcp-vpc \
    --subnetwork my-gcp-vpc-subnet \
    --enable-ip-alias \
    --enable-private-nodes \
    --master-ipv4-cidr 172.16.0.0/28 \
    --no-enable-master-authorized-networks
```

The above command will take a few minutes to run.
It create a GKE cluster named `airflow-standard-cluster` in the `us-central1` region. The cluster will be within the `my-gcp-vpc` network and `my-gcp-vpc-subnet` subnet. The cluster will have private nodes and the master will not be authorized to any network.


<u><b>
    <p style="font-size:20pt ">
      Creating Node Pool for Scaling
    </p>
</b></u>

For scaling purposes, we will create a node pool that has autoscaling enabled. The node pool again, is within the VPC that has Cloud NAT enabled.

```powershell
gcloud container node-pools create airflow-external-pool-e2standard4 \
    --cluster=airflow-standard-cluster \
    --region=us-central1 \
    --num-nodes=1 \
    --enable-autoscaling \
    --machine-type=e2-standard-4 \
    --disk-size=100 \
    --disk-type=pd-balanced \
    --image-type=COS_CONTAINERD \
    --metadata disable-legacy-endpoints=true \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --service-account=svc@gcp-prj-123.iam.gserviceaccount.com \
    --enable-private-nodes \
    --node-locations=us-central1-a,us-central1-b,us-central1-c
```

The above command will create a node pool named `airflow-external-pool-e2standard4` in the `airflow-standard-cluster` cluster. The node pool will have 1 node initially and will autoscale based on the workload. The node pool will have a machine type of `e2-standard-4` and a disk size of 100GB. The node pool will be within the `us-central1-a`, `us-central1-b`, and `us-central1-c` zones.


<u><b>
    <p style="font-size:20pt ">
      Deploying Airflow to GKE with Helm
    </p>
</b></u>

To deploy Airflow to GKE, we will use Helm. First, we will need to create a namespace for Airflow.

```powershell
kubectl create ns airflow
```

Then we will deploy Airflow using Helm

```powershell
helm repo update

helm install airflow apache-airflow/airflow --namespace airflow --debug --timeout 100m0s

helm ls -n airflow
```

To obtain a values.yaml for configuring Airflow, we can run the following command:

```powershell
helm show values apache-airflow/airflow > values.yaml
```

To verify deployment, we can run the following command:

```powershell
kubectl get pods -n airflow
kubectl top nodes -n airflow
```

To test the Airflow webserver, we can port-forward the webserver pod to our local machine:

```powershell
kubectl port-forward svc/airflow-webserver 8080:8080 -n airflow
```
Then we can access the Airflow webserver at `http://localhost:8080`.


<u><b>
    <p style="font-size:20pt ">
      Setting Up DNS for Airflow Server
    </p>
</b></u>

First, we will set up some environment variables:

```powershell
PROJECT=gcp-prj-id-123
CLUSTER_NAME=gke_gcp-prj-id-123-central1_airflow-standard-cluster
IP_NAME=airflow-ingress-ip
ZONE_NAME=mydomain-com
DNS_NAME=mydomain.com
```

Create an external IP that we will later attach to our domain
```powershell
gcloud compute addresses create ${IP_NAME} \
--global \
--ip-version IPV4 \
--project ${PROJECT}
```

To get the IP address

```powershell
IP=$(gcloud compute addresses describe ${IP_NAME} --global --project=${PROJECT} | grep 'address:' | cut -d':' -f2)
```

To verify, go to `VPC network` on console > `External IP address`, an external IP should be attached with the name `airflow-ingress-ip`

Setup DNS zone and ‘A’ record - this doesn’t need to be done if the domain is created via GCP, GCP created one automatically

```powershell
gcloud dns managed-zones create ${ZONE_NAME} --dns-name=${DNS_NAME} \
    --description "Zone for airflow" \
    --project ${PROJECT}
```

Create an “A” record for domain

```powershell
gcloud dns record-sets transaction start --zone=${ZONE_NAME} --project ${PROJECT}

gcloud dns record-sets transaction add ${IP} \
   --name=${DNS_NAME} \
   --ttl=300 \
   --type=A \
   --zone=${ZONE_NAME} \
   --project ${PROJECT}

gcloud dns record-sets transaction execute --zone=${ZONE_NAME} --project ${PROJECT}
```

To verify this step,
Go to `Cloud DNS`  and check is the external IP attached to the domain properly


<u><b>
    <p style="font-size:20pt ">
      Setting Up Ingress for Airflow Server
    </p>
</b></u>

Wherever the command is be running, on the dir, we will need an ingress folder with the below 2 yaml files, `gcp-manage-certs.yaml` and `ing.yaml`

For the `gcp-manage-certs.yaml` file, it will look something like this
```yaml
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: gcp-managed-cert-airflow
spec:
  domains:
    - 'mydomain.com'
```

For the `ing.yaml` file, it will look something like this
```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: airflow-ing
  annotations:
    networking.gke.io/managed-certificates: gcp-managed-cert-airflow
    kubernetes.io/ingress.allow-http: 'false'
    kubernetes.io/ingress.global-static-ip-name: airflow-ingress-ip
spec:
  rules:
    - host: 'mydomain.com'
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: airflow-webserver
                port:
                  number: 8080

```

Then execute the below,
```powershell
OUTPUT_FILE=ingress/gcp-manage-certs.yaml
cp ${OUTPUT_FILE} ${OUTPUT_FILE}.tmp
sed -i "s#<<DOMAIN>>#${DNS_NAME}#g" ${OUTPUT_FILE}.tmp
kubectl apply -f ${OUTPUT_FILE}.tmp -n ${NS}

OUTPUT_FILE=ingress/ing.yaml
cp ${OUTPUT_FILE} ${OUTPUT_FILE}.tmp
sed -i "s#<<DOMAIN>>#${DNS_NAME}#g" ${OUTPUT_FILE}.tmp
kubectl apply -f ${OUTPUT_FILE}.tmp -n ${NS}
```

To verify our ingress, we can run the following command:

```powershell
kubectl get ing -n airflow
```

It may take a while for the Address to appear and SSL to work.

If everything goes well, in about 10 mins time, we can navigate to https://mydomain.com and we should be able to login 
<p align="center">
<img alt = 'png' src='/images/on_airflow_prod/airflow-login.png'/>
</p>


<u><b>
    <p style="font-size:20pt ">
      Enable IAP for Security
    </p>
</b></u>

1 - On GCP console `Security` > **`Identity-Aware Proxy`**

Find our backend service, which should be automatically created from the above step

Turn `airflow/airflow/webserver` on and add user under under **`IAP-secured Web App Use`**.
Then also on `GCP IAM page`, add the role `IAP-secured Web App User` as well 

<p align="center">
<img alt = 'png' src='/images/on_airflow_prod/airflow-iap.png'/>
</p>

2 - Setup OAuth consent screen

On GCP console `API & Services` > `OAuth consent screen` > `Edit App` > Under `Authorized domains` > Add our Airflow Domain
For example, `mydomain.com`

3 - Authorized redirect URIs

On GCP console `API & Services` > `Credentials` 

Under `OAuth 2.0 Client IDs`, we should be able to see our airfliw websever service there, click on edit on the right, for `Authorized redirect URIs`, add the following: https://mydomain.com/oauth-authorized/google


<u><b>
    <p style="font-size:20pt ">
      Set up Cloud Armor for Security
    </p>
</b></u>

On GCP console `Network Security` > **`Cloud Armor`** > `Create Policy` > `Backend security policy`
Then we can create the policy based on our needs, for example, we can block all traffic except from certain IP addresses that we know is from us.

<p align="center">
<img alt = 'png' src='/images/on_airflow_prod/security-policy.png'/>
</p>

To attach the security policy to our backend service, we can follow the below steps

<p align="center">
<img alt = 'png' src='/images/on_airflow_prod/attach-security-policy.png'/>
</p>


<u><b>
    <p style="font-size:20pt ">
      Enable git-sync for DAGs
    </p>
</b></u>

On values.yaml, find the part for git sync

The beginning looks something like this,
```yaml
# Git sync
dags:
  # Where dags volume will be mounted. Works for both persistence and gitSync.
  # If not specified, dags mount path will be set to $AIRFLOW_HOME/dags
```

scroll down a bit to gitSync and modify enabled to true and modify `credentialsSecret: git-credentials`

```yaml
# Git sync
dags:
  # Where dags volume will be mounted. Works for both persistence and gitSync.
  # If not specified, dags mount path will be set to $AIRFLOW_HOME/dags
  mountPath: ~
  persistence:
    # Annotations for dags PVC
    annotations: {}
    # Enable persistent volume for storing dags
    enabled: false
    # Volume size for dags
    size: 1Gi
    # If using a custom storageClass, pass name here
    storageClassName:
    # access mode of the persistent volume
    accessMode: ReadWriteOnce
    ## the name of an existing PVC to use
    existingClaim:
    ## optional subpath for dag volume mount
    subPath: ~
  gitSync:
    enabled: true
    repo: https://github.com/my-github/airflow-dags.git
    branch: main
    rev: HEAD
    depth: 1
    # the number of consecutive failures allowed before aborting
    maxFailures: 0
    # subpath within the repo where dags are located
    # should be "" if dags are at repo root
    subPath: ""
    credentialsSecret: git-credentials
    period: 5s
    wait: ~
    # add variables from secret into gitSync containers, such proxy-config
    envFrom: ~
    containerName: git-sync
    uid: 65533
```

Create git-credentials secret using kubectl, it is recommended to use a service account with read access to the repo

```powershell
kubectl create secret generic git-credentials \
    --from-literal=GIT_SYNC_USERNAME=gh_svc \
    --from-literal=GIT_SYNC_PASSWORD=##### \
    --from-literal=GITSYNC_USERNAME=gh_svc \
    --from-literal=GITSYNC_PASSWORD=#### \
    -n airflow
```

Apply our changes using Helm

```powershell
helm upgrade --install airflow apache-airflow/airflow -n airflow -f values.yaml --debug
```

<u><b>
    <p style="font-size:20pt ">
      Integrate with GCP service account that has sufficient permissions
    </p>
</b></u>

1 - Add GCP creds json as secret

```powershell
kubectl create secret generic gcp-sc-key --from-file=gcp-key.json=/your/path/to/creds.json -n airflow
```

2 - Modify these values on `values.yaml` - usually on the top of the file

```yaml
env:
  - name: GOOGLE_APPLICATION_CREDENTIALS
    value: "/opt/airflow/secrets/gcp-key.json"

# Volumes for all airflow containers
volumes:
  - name: gcp-service-account-volume
    secret:
      secretName: gcp-sc-key

# VolumeMounts for all airflow containers
volumeMounts:
  - name: gcp-service-account-volume
    mountPath: /opt/airflow/secrets
```

3 - Modify the rest of the services in the values.yaml 

```yaml
# Airflow scheduler settings
scheduler:
  extraVolumeMounts:
    - name: google-cloud-key
      mountPath: /opt/airflow/secrets
  extraVolumes:
    - name: google-cloud-key
      secret:
        secretName: gcp-sc-key

# Airflow webserver settings
webserver:
  extraVolumeMounts:
    - name: google-cloud-key
      mountPath: /opt/airflow/secrets
  extraVolumes:
    - name: google-cloud-key
      secret:
        secretName: gcp-sc-key

# Airflow triggerer settings
triggerer:
  extraVolumeMounts:
    - name: google-cloud-key
      mountPath: /opt/airflow/secrets
  extraVolumes:
    - name: google-cloud-key
      secret:
        secretName: gcp-sc-key
```

Apply our changes using Helm

```powershell
helm upgrade --install airflow apache-airflow/airflow -n airflow -f values.yaml --debug
```

<u><b>
    <p style="font-size:20pt ">
      Adjusting Airflow components for scaling
    </p>
</b></u>

Modify the `values.yaml` file

```yaml
# Airflow scheduler settings
workers:
  resources:
    limits:
      memory: 5Gi
      cpu: 3500m
    requests:
      memory: 3Gi
      cpu: 2500m

webserver:
  resources:
    requests:
      memory: 1Gi
      cpu: 500m
    limits:
      memory: 2Gi
      cpu: 1000m

triggerer:
  resources:
    requests:
      memory: 1Gi
      cpu: 500m
    limits:
      memory: 2Gi
      cpu: 1000m
```

Apply our changes using Helm

```powershell
helm upgrade --install airflow apache-airflow/airflow -n airflow -f values.yaml --debug
```

<u><b>
    <p style="font-size:20pt ">
      Adding extra libraries to Airflow image
    </p>
</b></u>

1 - Create a registry on GCP, for example, `airflow`

2 - Set up a `pyproject.toml`, `Dockerfile` and `skaffold.yaml`

For the `pyproject.toml` file, it will look something like this

```toml
[project]
name = "setup-airflow-gke"
version = "0.1.0"
description = "GCP Airflow Config"
readme = "README.md"
requires-python = ">=3.12, <3.13"
dependencies = [
    "airflow-provider-great-expectations==0.2.9",
    "apache-airflow-providers-google==10.22.0",
]
```

For the `Dockerfile`, it will look something like this
Please note the airflow version need to match the airflow version th at Helm deployed. We can check the airflow version using the `values.yaml`, the `airflowVersion:` field

```dockerfile
FROM apache/airflow:2.9.3

ENV UV_VERSION=0.4.0

RUN pip install pipx
RUN pipx install uv==${UV_VERSION}

COPY pyproject.toml .

RUN uv pip install -r pyproject.toml
```

For the `skaffold.yaml` file, it will look something like this

```yaml
apiVersion: skaffold/v4beta2
kind: Config
metadata:
  name: airflow-custom
build:
  tagPolicy:
    envTemplate:
      template: "{{.IMAGE_NAME}}:latest"
  artifacts:
  - image: us-central1-docker.pkg.dev/gcp-prj-id-123/airflow/airflow-custom
    docker:
      dockerfile: Dockerfile
    platforms:
      - "linux/amd64"
```

Modify image settings from `values.yaml` , usually on the top of the file, we will have to use the GCP registry and modify the image tag to latest

```yaml
# Default airflow repository -- overridden by all the specific images below
defaultAirflowRepository: us-central1-docker.pkg.dev/gcp-prj-id-123/airflow/airflow-custom

# Default airflow tag to deploy
defaultAirflowTag: latest
```

Also need to modify `pullPolicy: Always`, if not k8s will not pull the new image since they all got the latest tag

```yaml
images:
  airflow:
    repository: ~
    tag: ~
    # Specifying digest takes precedence over tag.
    digest: ~
    pullPolicy: Always
```

To deploy our changes,

First we will need to build and push the image to the GCP registry, for that we will use skaffold to handle the process.

```powershell
skaffold run --cache-artifacts=false
```

Apply our changes using Helm after successful build

```powershell
helm upgrade --install airflow apache-airflow/airflow -n airflow -f values.yaml --debug
```

Sometimes pods are not updated after new lib are added, to force the pods to restart, we can run the below command

```powershell
kubectl delete pod -l component=webserver -n airflow
kubectl delete pod -l component=scheduler -n airflow
kubectl delete pod -l component=worker -n airflow
kubectl delete pod -l component=triggerer -n airflow
```

To confirm the pod is using the newest image

```powershell
 kubectl describe pod airflow-triggerer-0  -n airflow
 ```

<u><b>
    <p style="font-size:20pt ">
      Upgrading Airflow version
    </p>
</b></u>

As in this post is being written, the latest version of Apache Airflow is 2.10.1, and the airflow version Helm using is 2.9.3. So we will upgrade our airflow instance from 2.9.3 to 2.10.1 

1 - Update Airflow version on Dockerfile

```dockerfile
FROM apache/airflow:2.10.1

ENV UV_VERSION=0.4.0

RUN pip install pipx
RUN pipx install uv==${UV_VERSION}

COPY pyproject.toml .

RUN uv pip install -r pyproject.toml
```

2 - Update Airflow version on `values.yaml`

```yaml
airflowHome: /opt/airflow

# Default airflow repository -- overridden by all the specific images below
defaultAirflowRepository: us-central1-docker.pkg.dev/gcp-prj-id-123/airflow/airflow-custom

# Default airflow tag to deploy
defaultAirflowTag: latest

# Default airflow digest. If specified, it takes precedence over tag
defaultAirflowDigest: ~

# Airflow version (Used to make some decisions based on Airflow Version being deployed)
airflowVersion: "2.10.1"
```

3 - Build and push the image to the GCP registry

```powershell
skaffold run --cache-artifacts=false
```

4 - Upgrade the Airflow instance with Helm

```powershell
helm upgrade --install airflow apache-airflow/airflow -n airflow -f values.yaml --debug
kubectl get pods -n airflow
```

5 - Optionally, we can delete the pods to force the new image to be used

```powershell
kubectl delete pods --all -n airflow
```

That’s it for now.

Thank you for reading and have a nice day!
