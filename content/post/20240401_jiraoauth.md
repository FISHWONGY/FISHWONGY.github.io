---
authors:
- Hugo Authors
date: "2024-04-01"
excerpt: Setting up Jira OAuth Application with GCP Cloud Run
hero: /images/on_jiraoauth/jira-oauth-cover.png
title: On Jira OAuth with GCP Cloud Run 
---

<br />
<br />

A while back, I have talked about some [use case with Jira REST API](https://fishwongy.github.io/post/20231228_aiwebexbot) using api token.
To use the application in production, best practices will be either to obtain the the api token with a generic account or use OAuth 2.0 for SSO.
In this article,  we will discuss how to set up [Jira OAuth 2.0 (3LO) application](https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/) with GCP Cloud Run.

<u><b>
    <p style="font-size:20pt ">
      What is GCP Cloud Run
</b></u>

On Cloud Run, your code can either run continuously as a service or as a job. Both services and jobs run in the same environment and can use the same integrations with other services on Google Cloud.

- Cloud Run services. Used to run code that responds to web requests, or events.
- Cloud Run jobs. Used to run code that performs work (a job) and quits when the work is done.

In our case, we will be using Cloud Run service to host our Jira OAuth 2.0 app.

A Cloud Run service provides you with the infrastructure required to run a reliable HTTPS endpoint. Your responsibility is to make sure your code listens on a TCP port and handles HTTP requests.

Standard service features include:
- Unique HTTPS endpoint for every service
- Fast request-based auto scaling
- Built-in traffic management
- Private and public services

This is really helpful because every Cloud Run service is provided with an HTTPS endpoint on a unique subdomain of the *.run.app domain – and you can configure custom domains as well. Cloud Run manages TLS for you, and includes support for WebSockets, HTTP/2 (end-to-end), and gRPC (end-to-end).
And we will be using the HTTPS endpoint to set up our Jira OAuth 2.0 app.

For more information on GCP Cloud Run, you can check the [official documentation](https://cloud.google.com/run/docs/overview/what-is-cloud-run).

<u><b>
    <p style="font-size:20pt ">
      What is Jira OAuth 2.0 App
</b></u>

OAuth 2.0 (3LO) involves three parties:

- An Atlassian site (resource)
- A user (resource owner)
- An external application/service (client).

For example, a Jira or Confluence site (resource), an Atlassian user (resource owner), and client. Underlying the authorization interactions between these three parties is an authorization server.

To the user, the authorization process looks like this:

1. The app directs the user to an Atlassian screen that prompts them to grant access to their data on the Atlassian site. The screen displays the access being requested in the Atlassian product.
2. The user grants (or denies) access to their data on the Atlassian site, via the screen.
3. The user is directed back to the external service. If the user granted access, the external service can now access data (within the specified scopes) from the Atlassian site on the user's behalf.

For more information on Jira OAuth 2.0 App, you can check the [official documentation](https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/).

<u><b>
    <p style="font-size:20pt ">
      GCP Cloud Run App set up
</b></u>

Our application will look like this, use Fast API to build an application that we will deploy on CLoud Run, using the cloud run url for authentication with our Jira OAuth App that we can obtain our oauth token. And finally, using our oauth token, we can perform Jira operations with the REST API.

<p align="center">
<img alt = 'png' src='/images/on_jiraoauth/jiraoauth-flow.png'/>
</p>

Let's start with the code part. We will be using FastAPI to create a simple app that will be deployed on GCP Cloud Run. The below code is a simple health check endpoint, we will add the rest of the code after our first successful deployment and obtained the HTTPS endpoint.

app/main.py
```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import RedirectResponse, JSONResponse


app = FastAPI()

@app.get("/health-check")
def health_check():
    return {"status": "The application is running!"}
```

Then we will need to prepare our deployment files. We will be using Skaffold to deploy our app.

deploy/production/service.yaml
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: jira-sso-auth-app
spec:
  template:
    spec:
      serviceAccountName: svc@gcp-prj-123.iam.gserviceaccount.com
      containers:
      - image: us-central1-docker.pkg.dev/gcp-prj-123/repo/jira-sso-auth-app
        ports:
        - name: http1
          containerPort: 80
        env:
        - name: GCP_PROJECT_ID
          value: gcp-prj-123
        - name: ENV
          value: prod
```

Dockerfile
```dockerfile
FROM python:3.10-slim

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

COPY creds creds
COPY app .
CMD ["poetry", "run", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "80"]
```


pyproject.toml
```toml
[tool.poetry]
name = "jira-sso-auth"
version = "0.1.0"
description = ""
authors = [" "]
readme = "README.md"

[tool.poetry.dependencies]
python = "^3.9"
httpx = "^0.27.0"
fastapi = "^0.104.1"
uvicorn = "^0.24.0.post1"
starlette-session = "^0.4.3"
itsdangerous = "^2.1.2"


[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

skaffold.yaml
```yaml
apiVersion: skaffold/v4beta2
kind: Config
metadata:
  name: jira-sso-auth
build:
  artifacts:
  - image: us-central1-docker.pkg.dev/gcp-prj-123/repo/jira-sso-auth-app
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
    projectid: gcp-prj-123/repo
    region: us-central1
```

After preparing the deployment files, we can now deploy our app to GCP Cloud Run.

```bash
skaffold run -p production -v info
```

After the successful deployment, we will obtain the HTTPS endpoint of our app. It will be something like `https://jira-sso-auth-app-xxxxx.a.run.app`, and we will be using this HTTPS endpoint to set up our Jira OAuth 2.0 app.

Depending on use case, we may want to allow unauthenticated invocations.
<p align="center">
<img alt = 'png' src='/images/on_jiraoauth/cloudrun-auth.png'/>
</p>

<u><b>
    <p style="font-size:20pt ">
      Jira OAuth 2.0 App set up
</b></u>

1 - Log in to [jira developer console](https://developer.atlassian.com/console/myapps/) and create a new OAuth 2.0 app.

2 - After creating your app, go to `Permissions` section to add all necessary scopes the app will need. 

3 - Go to `Authorization` section to add the Callback URL. Using what we have obtained from the above steps, the Callback url should be something like `https://jira-sso-auth-app-xxxxx.a.run.app/oauth/callback`.

4 - After setting up the Callback URL, you will be provided with the `Client ID` and `Client Secret`, which can be found in the `Settings` section of your app.
We will be using these credentials in our app.


<u><b>
    <p style="font-size:20pt ">
      Adding OAuth 2.0 to our app
</b></u>

```python
from settings import CLIENT_ID, CLIENT_SECRET, JIRA_AUTH_URL, REDIRECT_URI

SCOPES = "read:jira-work read:jira-user read:me write:jira-work manage:jira-webhook manage:jira-data-provider offline_access"
AUTHORIZE_URL = "https://auth.atlassian.com/authorize"

app = FastAPI()


@app.get("/oauth/callback")
async def oauth_callback(request: Request, code: str, state: str):
    expected_state = request.session.get("oauth_state")
    if not expected_state or state != expected_state:
        raise HTTPException(status_code=400, detail="Invalid state parameter")

    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://auth.atlassian.com/oauth/token",
            json={
                "grant_type": "authorization_code",
                "client_id": CLIENT_ID,
                "client_secret": CLIENT_SECRET,
                "code": code,
                "redirect_uri": REDIRECT_URI,
            },
            headers={
                "Content-Type": "application/json",
            },
        )

        if response.status_code != 200:
            raise HTTPException(status_code=400, detail="Invalid authorization code")

        # Access token response
        token_data = response.json()
        access_token = token_data.get("access_token")
        refresh_token = token_data.get("refresh_token")
        expires_in = token_data.get("expires_in")
        # ... The rest of your logic to handle the token
    return "You are now logged in through OAuth!"


@app.get("/initiate-oauth")
def initiate_oauth(request: Request):
    state = str(uuid4())

    request.session["oauth_state"] = state

    params = {
        "audience": "api.atlassian.com",
        "client_id": CLIENT_ID,
        "scope": SCOPES,
        "redirect_uri": REDIRECT_URI,
        "state": state,
        "response_type": "code",
        "prompt": "consent",
    }
    url = f"{AUTHORIZE_URL}?" + "&".join(
        [f"{key}={value}" for key, value in params.items()]
    )
    return RedirectResponse(url=url, status_code=HTTP_302_FOUND)
    
```

Then we can redeploy our app to GCP Cloud Run.

```bash
skaffold run -p production -v info
```

After the successful deployment, we can now test our app by going to `https://jira-sso-auth-app-xxxxx.a.run.app/initiate-oauth` and you will be redirected to the Jira OAuth 2.0 app authorization page. After granting access, you will be redirected back to the app.

<br />
One thins to note is that, when using oauth token to perform Jira actions using the Jira REST API, instead of using the normal BASE_URL as `https://your-org.atlassian.net`, we will have to find our cloud id for the application and use it as the BASE_URL to make REST API requests with the oauth token.

1 - Make API request with personal token obtain via Jira Web UI

```python
import requests
import json
from requests.auth import HTTPBasicAuth

BASE_URL = "https://your-org.atlassian.net"
BASIC_AUTH = HTTPBasicAuth(JIRA_EMAIL, JIRA_TOKEN)
headers = {
    "Accept": "application/json",
    "Content-Type": "application/json",
}

# Make API request
endpoint = f"{BASE_URL}/rest/api/3/issue/{issue_key}"
response = requests.get(endpoint, auth=BASIC_AUTH, headers=headers)
```

2 - Make API request with oauth token
```python
import requests
import json

# Get the cloud id
OAUTH_ACCESS_TOKEN = "OAUTH-TOKEN"

JIRA_API_AUTH_URL = "https://api.atlassian.com/oauth/token/accessible-resources"
HEADERS = {
    'Authorization': f'Bearer {OAUTH_ACCESS_TOKEN}'
}

response = requests.get(JIRA_API_AUTH_URL, headers=HEADERS)

cloud_id = json.loads(response.content)[0]['id']

BASE_URL = f"https://api.atlassian.com/ex/jira/{cloud_id}"

# Make API request as usual
endpoint = f"{BASE_URL}/rest/api/3/issue/{issue_key}"
response = requests.get(endpoint, auth=BASIC_AUTH, headers=headers)
```

And that's it, now we have our application set up with Jira OAuth 2.0 using GCP Cloud Run, and we can use the token obtained to perform actions on Jira.

Thank you for reading and have a nice day!