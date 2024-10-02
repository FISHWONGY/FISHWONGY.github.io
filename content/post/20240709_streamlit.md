---
authors:
- Hugo Authors
date: "2024-07-09"
excerpt: How to set up a Streamlit app for machine learning prediction in Cloud Run and GKE
hero: /images/on_streamlit_ml/cover-photo.png
title: On Streamlit application for ML in production
---

Let's take a look at what we have disuccsed so far:

[1 - Setting up MLFlow with CLoud Run with IAP](https://fishwongy.github.io/post/20240706_mlflow/)

[2 - Setting up Kubeflow with GKE and IAP](https://fishwongy.github.io/post/20240707_kubeflow/)

[3 - Setting up ML endpoint for prediction with FastAPI and Cloud Run, also with Vertex AI Endpoint Serving and KServe](https://fishwongy.github.io/post/20240708_mlep/)

Now, in this last article of the MLOps series, we will look in to Streamlit app for machine learning prediction in Cloud Run and GKE. 


<u><b>Architecture</b></u>

Here is an overview of the MLOPS workflow.

<p align="center">
<img alt = 'png' src='/images/on_mlep/mlops-combined-architecture.png'/>
</p>

And in today's post, we will focus on this part.

<p align="center">
<img alt = 'png' src='/images/on_streamlit_ml/streamlit-focus.png'/>
</p>


This article will cover the following topics:

1 - Deploying a Streamlit app to Cloud Run using a MLFlow endpoint deployed on Cloud Run with FastAPI

2 - Deploying a Streamlit app to Cloud Run using Vertex AI Endpoint Serving

3 - Deploying a Streamlit app to GKE using the KServe endpoint deployed on GKE 


<u><b>
    <p style="font-size:20pt ">
      What is Streamlit?
</b></u>

Streamlit is an open-source app framework. It allows you to create web applications for various use cases, including machine learning models, with minimal effort. It is designed to be easy to use and allows you to create interactive web applications with just a few lines of code.


<u><b>
    <p style="font-size:20pt ">
      Setting up the Streamlit app with MLFlow endpoint on Cloud Run
</b></u>


To follow along with this part of the article, you can find the code on the GitHub repository [here](https://github.com/FISHWONGY/mlops-flows/tree/main/03-mlflow-wine-streamlit)


Folder strucutre is as below:
```md
.
├── Dockerfile
├── README.md
├── app
│   ├── basic_auth.py
│   ├── gcp_secrets.py
│   ├── main.py
│   └── settings.py
├── deploy
│   └── production
│       └── service.yaml
├── poetry.lock
├── pyproject.toml
└── skaffold.yaml
```

The `settings.py` file is as below, note that I have obtained the CREDENTIALS by decoding the base64 encoded string of the username and password for the MLFlow server. Since when we build the FastAPI app in our previous post, we have added the logic to pass the credentials on the header of the requests.

```python
from pydantic_settings import BaseSettings
from gcp_secrets import GCPSecrets
import base64

secrets = GCPSecrets()


class AppConfig(BaseSettings):
    MLFLOW_TRACKING_USERNAME: str = secrets.get_secret("mlflow_server_username")
    MLFLOW_TRACKING_PASSWORD: str = secrets.get_secret("mlflow_server_password")
    MLFLOW_ENDPOINT: str = secrets.get_secret("mlflow_endpoint")
    CRED_STR: str = f"{MLFLOW_TRACKING_USERNAME}:{MLFLOW_TRACKING_PASSWORD}"
    CRED_BYTES: bytes = str.encode(CRED_STR)
    CREDENTIALS: str = base64.b64encode(CRED_BYTES).decode("utf-8")


def get_config() -> AppConfig:
    return AppConfig()


app_config = get_config()
```

Thus in `main.py` file, when requests are made, it looks something like this
```python
if st.button("Predict"):
    response = requests.post(
        f"{app_config.MLFLOW_ENDPOINT}/invocations",
        json=data,
        headers={
            "Content-Type": "application/json",
            "Authorization": f"Basic {app_config.CREDENTIALS}",
        },
    )
    response.raise_for_status()
    prediction = response.json()
```

The `service.yaml` file is as below:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: mlflow-wine-streamlit-app
  labels:
    cloud.googleapis.com/location: us-central1
  annotations:
    run.googleapis.com/ingress: all
    run.googleapis.com/ingress-status: all
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/vpc-access-egress: all-traffic
        run.googleapis.com/vpc-access-connector: projects/gcp-prj-id-123/locations/us-central1/connectors/custom-vpc-connector
    spec:
      serviceAccountName: svc@developer.gserviceaccount.com
      containers:
      - image: us-central1-docker.pkg.dev/gcp-prj-id-123/mlflow-gcp/mlflow-wine-streamlit-app
        ports:
        - name: http1
          containerPort: 80
        env:
        - name: GCP_PROJECT_ID
          value: gcp-prj-id-123
        - name: ENV
          value: prod
```

Dockerfile is as below
```Dockerfile
FROM python:3.11-slim

WORKDIR /usr/src/app

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    POETRY_VERSION=1.7.1 \
    USERNAME=nonroot

RUN adduser $USERNAME
USER $USERNAME

ENV HOME=/home/$USERNAME
ENV PATH="$HOME/.local/bin:$PATH"

RUN pip install pipx
RUN pipx install poetry==${POETRY_VERSION}

COPY ./poetry.lock pyproject.toml /usr/src/app/
RUN poetry install -nv --no-root

COPY app .

EXPOSE 80

CMD ["poetry", "run", "streamlit", "run", "main.py", "--server.port=80", "--server.address=0.0.0.0"]
```

And the `skaffold.yaml` file is as below:

```yaml
apiVersion: skaffold/v4beta2
kind: Config
metadata:
  name: mlflow-wine-streamlit-app
build:
  artifacts:
  - image: us-central1-docker.pkg.dev/gcp-prj-id-123/mlflow-gcp/mlflow-wine-streamlit-app
    docker:
      dockerfile: Dockerfile
    platforms:
      - "linux/amd64"
profiles:
- name: production
  manifests:
    rawYaml:
    - deploy/production/service.yaml
deploy:
  cloudrun:
    projectid: gcp-prj-id-123
    region: us-central1
```

To deploy the Streamlit app to Cloud Run, you can run the following command:

```bash
skaffold run -p production
```

<p align="center">
<img alt = 'png' src='/images/on_streamlit_ml/streamlit-login.png'/>
</p>

<u><b>
    <p style="font-size:20pt ">
      Setting up the Streamlit app with Vertex AI endpoint on Cloud Run
</b></u>

To follow along with this part, you can find the code on the GitHub repository [here](https://github.com/FISHWONGY/mlops-flows/tree/main/04-vertexai-ep-streamlit)


The set up is similar to the above, but instead of calling the MLFlow endpoint, we will call the Vertex AI endpoint. 

The `settings.py` file will look something like this:

```python
from pydantic_settings import BaseSettings

from gcp_secrets import GCPSecrets

secrets = GCPSecrets()


class AppConfig(BaseSettings):
    MLFLOW_TRACKING_USERNAME: str = secrets.get_secret("mlflow_server_username")
    MLFLOW_TRACKING_PASSWORD: str = secrets.get_secret("mlflow_server_password")

    GCP_CREDS: str = secrets.get_secret("svc-creds-json")

    PROJECT_ID: str = "gcp-prj-id-123"
    REGION: str = "us-central1"
    ENDPOINT_ID: str = "1234567"
    ENDPOINT_URL: str = f"https://{REGION}-aiplatform.googleapis.com/v1/projects/{PROJECT_ID}/locations/{REGION}/endpoints/{ENDPOINT_ID}:predict"


def get_config() -> AppConfig:
    return AppConfig()


app_config = get_config()
```


To obtain a requests header, we will need to do the following:

```python
SCOPES: list = ["https://www.googleapis.com/auth/cloud-platform"]
GCP_CREDS_DICT = extract_json_string(app_config.GCP_CREDS)
credentials = service_account.Credentials.from_service_account_info(
    GCP_CREDS_DICT, scopes=SCOPES
)
credentials.refresh(Request())
HEADERS = {
    "Authorization": f"Bearer {credentials.token}",
    "Content-Type": "application/json",
}
```

When making prediction, it will look something like this:

```python
if st.button("Predict"):
  response = requests.post(
      f"{app_config.ENDPOINT_URL}",
      headers=HEADERS,
      json=inference_input,
  )
  response.raise_for_status()
  prediction = response.json()
  pred_result = prediction["predictions"][0]
```

To deploy the Streamlit app to Cloud Run, you can run the following command:

```bash
skaffold run -p production
```

<p align="center">
<img alt = 'png' src='/images/on_streamlit_ml/streamlit-login.png'/>
</p>

<u><b>
    <p style="font-size:20pt ">
      Setting up the Streamlit app with KServe endpoint on GKE
</b></u>

To follow along with this part, you can find the code on the GitHub repository [here](https://github.com/FISHWONGY/mlops-flows/tree/main/05-kserve-k8s-streamlit)

Let's jump right in.

The folder structure is as below:

```md
.
├── Dockerfile
├── README.md
├── app
│   ├── basic_auth.py
│   ├── gcp_secrets.py
│   ├── main.py
│   └── settings.py
├── deploy
│   ├── common
│   │   ├── config.yaml
│   │   ├── deployment.yaml
│   │   ├── kustomization.yaml
│   │   ├── service.yaml
│   │   └── serviceaccount.yaml
│   └── production
│       └── kustomization.yaml
├── poetry.lock
├── pyproject.toml
└── skaffold.yaml
```

Before we start configuring the deployment artifacts, there is a few things we need to do

For deployment, in order to pull the image from the Google Container Registry, you need to create/ make sure your node pool has the scopes `cloud-platform` and `storage-full`
You can do this by running the following command:

```bash
gcloud container node-pools create default-node-pool-new \
    --cluster=kubeflow  \
    --zone=us-central1-c \
    --scopes=cloud-platform,storage-full \        
    --num-nodes=2

```

<p align="center">
<img alt = 'png' src='/images/on_streamlit_ml/create-nodepool.png'/>
</p>

Create a json key for accessing the GCP Artifacts Registry
```bash
gcloud iam service-accounts keys create key.json \                               
  --iam-account=svc@gcp-prj-id-123.iam.gserviceaccount.com
````

```bash
kubectl create secret docker-registry gar-json-key \                             
  --docker-server=us-central1-docker.pkg.dev \
  --docker-username=_json_key \
  --docker-password="$(cat key.json)" \
  --docker-email=user@email.com
```

Reference the secret in the `deployment.yaml` file as shown below:
```yaml
spec:
  serviceAccountName: default-editor
  imagePullSecrets:
  - name: gar-json-key
```

Also in `serviceaccount.yaml` file, 
```yaml
imagePullSecrets:
- name: gar-json-key
```

Ensure the service account has the role artifactregistry.reader
```bash
gcloud projects add-iam-policy-binding gcp-prj-id-123 \
    --member "serviceAccount:svc@gcp-prj-id-123.iam.gserviceaccount.com" \
    --role "roles/artifactregistry.reader"
```

```bash
gcloud artifacts repositories add-iam-policy-binding mlflow-gcp \                 
    --location=us-central1 \
    --member=serviceAccount:svc@gcp-prj-id-123.iam.gserviceaccount.com \
    --role="roles/artifactregistry.reader"
```

Dockerfile is as below:

```Dockerfile
FROM python:3.11-slim

WORKDIR /usr/src/app

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    POETRY_VERSION=1.7.1 \
    USERNAME=nonroot

RUN adduser $USERNAME
USER $USERNAME

ENV HOME=/home/$USERNAME
ENV PATH="$HOME/.local/bin:$PATH"

RUN pip install pipx
RUN pipx install poetry==${POETRY_VERSION}

COPY ./poetry.lock pyproject.toml /usr/src/app/
RUN poetry install -nv --no-root

COPY app .

EXPOSE 8501

CMD ["poetry", "run", "streamlit", "run", "main.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

The `skafoold.yaml` file is as below:

```yaml
apiVersion: skaffold/v2beta28
kind: Config
metadata:
  name: kserve-k8s-streamlit-app
build:
  artifacts:
  - image: us-central1-docker.pkg.dev/gcp-prj-id-123/mlflow-gcp/kserve-k8s-streamlit-app
    docker:
      dockerfile: Dockerfile
deploy:
  kubeContext: gke_gcp-prj-id-123_us-central1-c_kubeflow
  kustomize:
    paths:
    - deploy/common
profiles:
- name: production
  deploy:
    kustomize:
      paths:
      - deploy/production
```

The `deploy/common/config.yaml` file is as below:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kserve-k8s-streamlit-config
data:
  GCP_PROJECT_ID: "gcp-prj-id-123"
  ENV: "prod"
```

The `deploy/common/deployment.yaml` file is as below:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kserve-k8s-streamlit
  namespace: your-namespace
  labels:
    app: kserve-k8s-streamlit
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kserve-k8s-streamlit
  template:
    metadata:
      labels:
        app: kserve-k8s-streamlit
    spec:
      serviceAccountName: default-editor
      imagePullSecrets:
      - name: gar-json-key
      nodeSelector:
        cloud.google.com/gke-nodepool: default-node-pool-new
      containers:
      - name: kserve-k8s-streamlit
        image: us-central1-docker.pkg.dev/gcp-prj-id-123/mlflow-gcp/kserve-k8s-streamlit-app
        envFrom:
          - configMapRef:
              name: kserve-k8s-streamlit-config
        resources:
          limits:
            memory: 200Mi
            cpu: 50m
          requests:
            memory: 100Mi
            cpu: 50m
        ports:
        - containerPort: 8501
        volumeMounts:
        - name: kube-api-access
          mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          readOnly: true
      volumes:
      - name: kube-api-access
        projected:
          sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600
          - configMap:
              name: kube-root-ca.crt
              items:
              - key: ca.crt
                path: ca.crt
          - downwardAPI:
              items:
              - path: namespace
                fieldRef:
                  fieldPath: metadata.namespace
```

The `deploy/common/service.yaml` file is as below:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: streamlit-app-service
  namespace: your-namespace
spec:
  selector:
    app: kserve-k8s-streamlit
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8501
  type: LoadBalancer
  ```
  
The `deploy/common/serviceaccount.yaml` file is as below:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    iam.gke.io/gcp-service-account: svc@gcp-prj-id-123.iam.gserviceaccount.com
  name: default-editor
  namespace: your-namespace
imagePullSecrets:
- name: gar-json-key
```

The `deploy/production/kustomization.yaml` file is as below:

```yaml
namespace: your-namespace
resources:
  - deployment.yaml
  - config.yaml
  - serviceaccount.yaml
  - service.yaml
```

To deploy the Streamlit app to GKE, you can run the following command:

```bash
skaffold run -p production
```

<p align="center">
<img alt = 'png' src='/images/on_streamlit_ml/skaffold-deploy.png'/>
</p>

To get the external IP address of the service, you can run the following command:

```bash
kubectl get svc streamlit-app-service -n your-namespace
```

<p align="center">
<img alt = 'png' src='/images/on_streamlit_ml/external-ip.png'/>
</p>

This will give you the external IP address of the service, which you can use to access the Streamlit app by `http://<EXTERNAL-IP>:80`

<p align="center">
<img alt = 'png' src='/images/on_streamlit_ml/streamlit-login.png'/>
</p>

And that's it, now we have learnt how to deploy streamlit application using various endpoints and with tools such as Cloud Run and GKE.

Thank you for reading and have a nice day!
