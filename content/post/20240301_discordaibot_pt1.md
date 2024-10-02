---
authors:
- Hugo Authors
date: "2024-03-01"
excerpt: Step by step guide to build a Discord AI Chat Bot Powered by GCP
hero: /images/on_discordAiBot/pt1_cover.png
title: On Discord AI Chat Bot - Part 1
---

In this blog post series, I will be discussing the steps of building a AI Chat Bot on Discord using Google Cloud Platform's (GCP) AI services, then deploy on the chat bot on GKE.

<p align="center">
<img alt = 'gif' src='/images/on_discordAiBot/discord-ai-bot-demo.gif'/>
</p>

[Part 1](https://fishwongy.github.io/post/20240301_discordaibot_pt1) focuses on the set up for the discord bot and GCP, [part 2](https://fishwongy.github.io/post/20240302_discordaibot_pt2) will discuss the integration of Vertex AI with the discord bot inclucding code implementation and [part 3](https://fishwongy.github.io/post/20240303_discordaibot_pt3) will focus on RAG implementation.

To follow through the code [here](https://github.com/FISHWONGY/discord-gcpai-bot).

<u><b>
    <p style="font-size:20pt ">
      Discord Bot Setup
</b></u>

1 - Go to Discord Developer Portal and create a new application.

2 - Add the Bot Permission

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/bot_permission.png'/>
</p>

3 - Generate the URL link to add the bot to the channel

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/bot_auth.png'/>
</p>

4 - Add needed intents to the Bot

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/bot_intents.png'/>
</p>

Below are optional if interested.

5 - Turn on developer mode on your Discord app, we will need to have developer mode turned on in order to get different ids.

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/bot_devmode.png'/>
</p>

6 - Setting description of Bot using slash commands

On Discord server > Server Settings > Integration > Find your BOT > Commands section > Right click to the Command ID

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/bot_command_id.png'/>
</p>


On Developer Portal > Choose your BOT > Paste the text as below in the Description

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/bot_description.png'/>
</p>

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/bot_description2.png'/>
</p>

7 - Github integration with Discord

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/gh_webhook.png'/>
</p>

a. In discord add an integration, and copy webhook URL
b. At the discord repo, settings > add repo
c. Paste in the webhook URL and append `/github` to the end. Select "Send me everything", set the type to `application/json`, and then Add Webhook

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/gh_webhook2.png'/>
</p>

<u><b>
    <p style="font-size:20pt ">
      GCP cli Setup
</b></u>

1	- Download cloud SDK
		https://cloud.google.com/sdk/docs/install

2	- Navigate to the google-cloud-sdk dir and run
```powershell
./install.sh
```

3	- Follow the steps in terminal inputting Y/N

4	- Restart your terminal

5	- On Local Terminal run
```powershell
gcloud auth login
```

6	- Run 
```powershell
gcloud config set project gcp-prj-id-123
```


7	- To install GKE related cli, run 
```powershell
gcloud config set project gke-gcloud-auth-plugin
```

8	- To verify above step, run 
```powershell
gke-gcloud-auth-plugin --version
```

9	- Run 
```powershell
gcloud container clusters get-credentials autopilot-cluster --region us-central1 --project gcp-prj-id-123
```

10 - Check various info of GKE
```powershell
k get namespace
```
```powershell
k -n namespace get pods
```


11	- If intersted in other componenets
```powershell
gcloud components list --show-versions
```

12	- Auth of aritfacts registry
```powershell
gcloud auth configure-docker us-central1-docker.pkg.dev
```

Check artifects
```powershell
gcloud container images list-tags us-central1-docker.pkg.dev/gcp-prj-id-123/repo/deployed-app
gcloud container images list-tags us-central1-docker.pkg.dev/gcp-prj-id-123/repo/deployed-app --sort-by=TIMESTAMP
```

```powershell
gcloud artifacts docker images list us-central1-docker.pkg.dev/gcp-prj-id-123/repo
```

<u><b>
    <p style="font-size:20pt ">
      GCP Project Setup
</b></u>

1 - Enable both Vertex API, Generative Language API and Cloud Logging API

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/enable_vertexai.png'/>
</p>

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/enable_gen.png'/>
</p>

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/enable_cloud.png'/>
</p>

2 - Add a repo in Artifact Registry that will be used for deployment

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/artifects_reg.png'/>
</p>

3 - Create a GKE cluster

```powershell
gcloud container clusters create autopilot-cluster \
--machine-type n1-standard-4 \
--num-nodes 1 \
--region "us-central1"
```


4 - Create a namespace on the cluster, here we will be using the namespace "automations"

```powershell
k create ns automations
```


5 - Create a service account for the application with sufficient role and bind it to the namespace


Create SVC account in k8s cluster
```powershell
# Create SVC account in k8s cluster
k create serviceaccount discordbot-sa \
    --namespace automations
```

Create GCP service account
```powershell
# Create gcp service account
gcloud iam service-accounts create discordbot \
    --project=gcp-proj-id-123
"projects/gcp-proj-id-123/serviceAccounts/discordbot@gcp-proj-id-123.iam.gserviceaccount.com"
```

Add roles to GCP SVC
```powershell
# Add role to gcp svc
gcloud projects add-iam-policy-binding gcp-proj-id-123 \
    --member "serviceAccount:discordbot@gcp-proj-id-123.iam.gserviceaccount.com" \
    --role "roles/iam.serviceAccountAdmin"

# Add all role to gcp svc
gcloud projects add-iam-policy-binding gcp-proj-id-123 \
    --member "serviceAccount:discordbot@gcp-proj-id-123.iam.gserviceaccount.com" \
    --role "roles/iam.serviceAccountAdmin" \
    --role "roles/container.admin" \
    --role "roles/secretmanager.admin" \
    --role "roles/secretmanager.secretAccessor" \
	--role "roles/aiplatform.user"
```

Binding
```powershell
# Binding
gcloud iam service-accounts add-iam-policy-binding discordbot@gcp-proj-id-123.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:gcp-proj-id-123.svc.id.goog[automations/discordbot-sa]"
```

Verify
```powershell
# Verify
gcloud iam service-accounts get-iam-policy discordbot@gcp-proj-id-123.iam.gserviceaccount.com
```


And that's it! We now have a Discord Bot and a GCP project configured. In the [next blog post](https://fishwongy.github.io/post/20240302_discordaibot_pt2), I will be discussing the implementation of the AI services and the deployment of the bot on GKE.

Thank you for reading and have a nice day!
