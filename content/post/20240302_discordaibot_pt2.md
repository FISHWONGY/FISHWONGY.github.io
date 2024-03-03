---
authors:
- Hugo Authors
date: "2024-03-02"
excerpt: Step by step guide to build a Discord AI Chat Bot Powered by GCP
hero: /images/on_discordAiBot/pt2_cover.png
title: On Discord AI Chat Bot - Part 2
---

After the [last blog post](https://fishwongy.github.io/post/20240301_discordaibot_pt1), we should now have a Discord Bot and a GCP project configured. In this blog post, we will be discussing the code implementation of building a AI Discord Chat Bot on using Google Cloud Platform's(GCP) AI services, then deploy on the chat bot on GKE.

<p align="center">
<img alt = 'gif' src='/images/on_discordAiBot/discord-ai-bot-demo.gif'/>
</p>


So let's jump right in!

<u><b>
    <p style="font-size:20pt ">
      Project Setup
</b></u>

1 - Create `pyproject.toml` file


Note that we are using python 3.10 and poetry 1.3.0 here, as some of the pycord dependencies are not compatible with higher version of python and poetry.

```toml
[tool.poetry]
name = "discord-ai-bot"
version = "0.1.0"
description = ""
authors = [" "]
readme = "README.md"

[tool.poetry.dependencies]
python = "^3.10"

python-dateutil = "^2.8.2"
pydantic = "^1.10.4"
requests = "^2.31.0"
discord-webhook  = "1.3.0"
py-cord = "2.4.1"
pillow = "^10.2.0"
emoji = "^2.9.0"

google-cloud-aiplatform = "^1.38.0"
google-cloud-secret-manager = "^2.16.3"
google-cloud-storage = "^2.10.0"
google-cloud-logging = "^3.8.0"

langchain = "0.1.4"
langchain-google-vertexai = "^0.0.3"
langchain-google-genai = "^0.0.6"
langchain-openai= "^0.0.5"
protobuf = "^3.19.3"

python-dotenv = "^0.21.0"

[build-system]
requires = ["poetry-core=1.3.0"]
build-backend = "poetry.core.masonry.api"
```

2 - Install dependencies on your virtual environment

```powershell
poetry install
```

<u><b>
    <p style="font-size:20pt ">
      Bot Implementation
</b></u>

First, let's initiate a bot object

```python
class DiscordBot:
    def __init__(self) -> None:
        self.intents = discord.Intents.all()
        self.activity = discord.Activity(type=discord.ActivityType.competing, name="/help | !help")
        self.status = discord.Status.online
        self.DC_SER_ID = secrets.get_secret("dc-ser-id")
        self.TOKEN = secrets.get_secret("dc-bot-token")

    def bot_initiate(self):
        return commands.Bot(
            command_prefix="!", intents=self.intents, activity=self.activity
        )
```

To use embed message, we can do something like this:

```python
  @staticmethod
  def slash_help_embed() -> discord.Embed:
      embed = discord.Embed(
          title=emoji.emojize(":robot: Google Powered AI Bot! :robot:"),
          description="\n\n"
          "• **Quickstart** with Gemini `!g <your-input>`\n"
          "• For More info on `!` Bot Command do `!help`\n\n"
          "• For More info on **Slash Command** do  `/help <command>`\n"
          "  Example: `/help gemini`\n\n\n",
          color=discord.Colour.from_rgb(31, 102, 138),
      )
      embed.add_field(
          name="Slash Command Categories:",
          value="\n\n"
          "• `/gemini`: Chat with Google's Gemini\n"
          "• `/img`: Generate **AI image** with Vertex AI\n"
          "• `/lang`: Chat with Google's Bison LLM\n"
          "• `/py`: Chat with Google Gemini Powered Python Assistant\n"
          "• `/pycode`: Chat with Google Codey Powered Python Assistant\n"
          "• `/hist`: Retrieve and save chat history to GCS bucket\n\n"
          f"{emoji.emojize(':smiling_face_with_sunglasses:  Elevate your Discord experience with Gojo AI :smiling_face_with_sunglasses:')}\n\n",
      )
  
      embed.timestamp = datetime.utcnow()
      embed.set_footer(text="\u200b")
      embed.set_author(
          name="Gojo AI",
          icon_url="https://image.png",
      )
      embed.set_thumbnail(
          url="https://image.png"
      )
  
      return embed
```


Then putting it together, to send out an embed message with the slash command `/help` we can do something like this:

```python
pycordapi = DiscordBot()

bot = pycordapi.bot_initiate()

@bot.event
async def on_ready():
    logger.info(f"We have logged in as {bot.user}")


@bot.slash_command(
    name="help",
    guild_ids=[pycordapi.DC_SER_ID],
    description="To show all commands",
)
async def help(ctx):
    await ctx.defer()
    await ctx.respond(embed=pycordapi.slash_help_embed())
```

Optionally, we can also add an extra param for the `/help` command for further customization for each command's usage

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/help_command.png'/>
</p>

```python
@bot.slash_command(
    name="help",
    guild_ids=[pycordapi.TEST_SER_ID],
    description="To show all commands",
)
async def help(ctx, command: str = None):
    await ctx.defer()
    if command is None:
        await ctx.respond(embed=pycordapi.slash_help_embed())
    else:
        help_message = pycordapi.HELP_MSG_DEATILS.get(command)
        if help_message is not None:
            await ctx.respond(
                embed=pycordapi.get_embed(
                    f"{help_message}",
                    discord.Colour.from_rgb(31, 102, 138),
                )
            )
        else:
            await ctx.respond(
                embed=pycordapi.get_embed(
                    "Error\nNot an available slash command, do `/help` to check all available slash command",
                    discord.Colour.brand_red(),
                )
            )

```

If we don't want to use slash command, a normal command can be implemented like this:

```python
@bot.command(help="Testing command for bot development")
async def test(ctx, *, args):
    received_msg = "".join(args)

    logger.info(f"Received message: {received_msg} from user {ctx.author} ")

    await ctx.send(f"""Testing. Here is the received message:\n{received_msg}""")
```


<u><b>
    <p style="font-size:20pt ">
      AI Function Implementation
</b></u>

<p align="center">
<img alt = 'png' src='/images/on_discordAiBot/slash_command.png'/>
</p>

There are various models Google provided with API or python SDK, we will be using the following models:
[Gemini](https://deepmind.google/technologies/gemini/), [chat-bison v2](https://cloud.google.com/vertex-ai/docs/generative-ai/model-reference/text-chat), [codechat-bison](https://cloud.google.com/vertex-ai/docs/generative-ai/model-reference/code-chat) and [Vertex AI Imagen](https://cloud.google.com/vertex-ai/docs/generative-ai/image/overview)

Let's start with importing the relevant modules

```python
import vertexai
from vertexai.preview.generative_models import (
    GenerativeModel,
    ChatSession,
    Content,
    Part,
)
from vertexai.language_models import (
    ChatModel,
    CodeChatModel,
    InputOutputTextPair,
    ChatMessage,
)
```

To initiate the models, we can do something like this:

```python
class GCPAI:
    def __init__(self) -> None:
        vertexai.init(project=GCP_PROJECT, location="us-central1")
        self.gem_model = GenerativeModel("gemini-pro")
        self.chat_model = ChatModel.from_pretrained("chat-bison@002")
        self.codechat_model = CodeChatModel.from_pretrained("codechat-bison")
        self.img_gen_endpoint = f"https://us-central1-aiplatform.googleapis.com/v1/projects/{GCP_PROJECT}/locations/us-central1/publishers/google/models/imagegeneration:predict"
```


All of these models can take chat history as a parameter so that the bot or the conversation we are building can actually be tuned by the system prompt we feed to the model. So let's take a look on how can we implement chat history.

And as of I am writing this now, the GCP official documentation is not very clear on how to implement chat history, so I what I did is read on their source code and work my way backward.

Let's take Gemini as an example, we can do something like as below to initiate a Gemini chat session with chat history. In this example, we also want to provide the flexibility for user to either start a new chat session or to use the on-going chat session so that the chat bot has the context of the conversation.

```python
class GCPAI:
  def __init__(self) -> None:
    vertexai.init(project=GCP_PROJECT, location="us-central1")
    self.gem_model = GenerativeModel("gemini-pro")
    self.gem_history = [
            Content(
                role="user",
                parts=[Part.from_text(prompts.PYTHON_CONTEXT_PROMPT)],
            ),
            Content(
                role="model",
                parts=[Part.from_text(prompts.PYTHON_SYSTEM_PROMPT1)],
            ),
        ]
    self.gem_agent = self.gem_model.start_chat(history=self.gem_history)
    
    self.agents_config = {"gem": {
                "agent": self.gem_agent,
                "model": GenerativeModel("gemini-pro"),
                "clean_chat_params": {"history": []},
                "start_chat_params": {"history": self.gem_history},
            },
          }
  
  def get_response(
        self, prompt: str, response_type: str, use_existing_session: bool = True
    ) -> str:
        agent_info = self.agents_config[response_type]
        if use_existing_session:
            agent_to_use = agent_info["agent"]
        else:
            model = agent_info["model"]
            agent_to_use = model.start_chat(**agent_info.get("start_chat_params", {}))

        response = agent_to_use.send_message(prompt)
        return response.text

```

<u><b>
    <p style="font-size:20pt ">
      Deployment
</b></u>

Finally, let's talk about deploying our chat bot to GKE.

Our folder structure should look like this:

```markdown
.
├── Dockerfile
├── README.md
├── app
│   ├── commands.py
│   ├── helpers
│   │   ├── common_func.py
│   │   ├── gcp_ai.py
│   │   ├── gcp_secrets.py
│   │   ├── gcp_storage.py
│   │   ├── prompts.py
│   │   └── pycordapi.py
│   ├── main.py
│   └── utils
│       ├── matching_engine.py
│       └── matching_engine_utils.py
├── deploy
│   ├── common
│   │   ├── config.yaml
│   │   ├── deployment.yaml
│   │   ├── kustomization.yaml
│   │   └── serviceaccount.yaml
│   └── production
│       └── kustomization.yaml
├── poetry.lock
├── pyproject.toml
└── skaffold.yaml
```

We will be deploying using `skaffold`

So let's start by creating a `skaffold.yaml` file

```yaml
apiVersion: skaffold/v2beta28
kind: Config
metadata:
  name: discordbot
build:
  artifacts:
  - image: us-central1-docker.pkg.dev/gcp-proj-id-123/discord/discord-gcpai-bot-app
    docker:
      dockerfile: Dockerfile
deploy:
  kubeContext: gke_gcp-proj-id-123_us-central1_autopilot-cluster-2
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

Then under common/deployment.yaml, we can add the deployment configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: discord-gcpai-bot
  labels:
    app: discord-gcpai-bot
spec:
  replicas: 1
  selector:
    matchLabels:
      name: discord-gcpai-bot
  template:
    metadata:
     labels:
      name: discord-gcpai-bot
      app: discord-gcpai-bot
    spec:
      serviceAccountName: discordbot-sa
      restartPolicy: Always
      containers:
        - image: us-central1-docker.pkg.dev/gcp-proj-id-123/discord/discord-gcpai-bot-app
          name: discord-gcpai-bot
          envFrom:
            - configMapRef:
                name: discord-gcpai-bot-config
          resources:
            limits:
              memory: 512Mi
              cpu: 500m
            requests:
              memory: 512Mi
              cpu: 500m
```


For Dockerfile, we can do something like this:

```dockerfile
FROM python:3.10-slim

WORKDIR /usr/src/app

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    POETRY_VERSION=1.3.0

RUN apt-get update && apt-get install -y curl
RUN curl -sSL https://install.python-poetry.org | POETRY_VERSION=1.3.0 python3 -
ENV PATH="/root/.local/bin:$PATH"
COPY ./poetry.lock pyproject.toml /usr/src/app/
RUN poetry install -nv --no-root

COPY app .
CMD ["poetry", "run", "python3", "main.py"]
```

Then we can build and deploy using `skaffold`

```powershell
skaffold run -p production -v info
```

And that's it! We have successfully deployed our AI chat bot to GKE!
 In the [next blog post](https://fishwongy.github.io/post/20240303_discordaibot_pt3), I will be discussing the implementation of RAG with Google Vertex AI.

Thank you for reading and have a nice day!
