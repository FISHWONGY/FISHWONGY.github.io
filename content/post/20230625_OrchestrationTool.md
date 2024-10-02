---
authors:
- Hugo Authors
date: "2023-06-25"
excerpt: A mini project testing out both tools
hero: /images/on_orchestrationTool/dagster_prefect.png
title: On Orchestration Tools
---

<div style="text-align: justify">

There is so many orchestrator these days for data team to choose, [airflow](https://airflow.apache.org/), [dagster](https://www.dagster.io/), [prefect](https://www.prefect.io/), [mage ai](https://www.mage.ai/) just to name a few ...

This weekend, I decided to build a mini project to test out both tool.

The project is similar to [before](https://fishwongy.github.io/post/20220610_gcf/), extract sports news from different news outlet and send them automatically to discord via webhook.


<p align="center">
<img alt = 'png' src='/images/on_orchestrationTool/dagster_prefect_workflow.png'/>
</p>

Let's jump right in.

<u><b>
    <p style="font-size:20pt ">
      Prerequisites
    </p>
</b></u>

- Python3.8+

<u><b>
    <p style="font-size:20pt ">
      Dagster
    </p>
</b></u>

```powershell
pip install dagster
```

To spin up a project

```powershell
pip install -e ".[dev]"

dagster dev
```

I have refactored the folder strcuture a bit using poetry instead of its original config.

Dagster Folder Structure
```md
.
├── README.md
├── app
├── dagster_discord_news
│   ├── __init__.py
│   ├── common.py
│   ├── helpers
│   ├── jobs.py
│   └── main.py
├── news_data
│   ├── bbcf1.txt
│   ├── bbcfootball.txt
│   └── officialf1.txt
├── poetry.lock
├── pyproject.toml
├── setup.cfg
├── setup.py
├── tests
│   └── test_assets.py
└── workspace.yaml

```

Dagster UI

<p align="center">
<img alt = 'png' src='/images/on_orchestrationTool/dagster_ui1.png'/>
</p>


<p align="center">
<img alt = 'png' src='/images/on_orchestrationTool/dagster_ui2.png'/>
</p>


_Photo updated on Jan 2024._

<u><b>
    <p style="font-size:20pt ">
      Prefect
    </p>
</b></u>


```powershell
pip install prefect
```

Prefect provide a free tier prefect cloud for user, which we can all register

Then what we need to do is login using the command line and start an agent for the job

```powershell
prefect cloud login

prefect agent start --pool "default-agent-pool"
```


Prefect Folder Structure
```md
.
├── app
│   ├── common.py
│   ├── errors.py
│   ├── helpers
│   │   ├── bbcnews.py
│   │   ├── discordapi.py
│   │   └── f1news.py
│   ├── main.py
│   └── subflows.py
├── publish_officialf1_news-deployment.yaml
└── publish_sports_news_to_dc-deployment.yaml

```

Prefect UI

<p align="center">
<img alt = 'png' src='/images/on_orchestrationTool/prefect_ui1.png'/>
</p>


<p align="center">
<img alt = 'png' src='/images/on_orchestrationTool/prefect_ui2.png'/>
</p>


And here is the news from BBC Sports!
<p align="center">
<img alt = 'png' src='/images/on_orchestrationTool/orchestrator_example.png'/>
</p>


<u><b>
    <p style="font-size:20pt ">
      Conclusion
    </p>
</b></u>


I found that [Prefect](https://www.prefect.io/) is a bit easier to set up for my project this time, with their free tier prefect cloud account, everything just comes together easily.
On the other hand, I like the [asset](https://docs.dagster.io/concepts/assets/software-defined-assets) concept from [dagster](https://dagster.io/), and I think their integration with dbt is better. 

Both Prefect and Dagster have the majority of their community on Slack, which is different from Airflow, where users typically ask on forums like Stack Overflow for debugging. This makes googling and debugging a bit tricky. 

However, both of their documentations are clear. So, for a mini-project this time, I can get it over the line pretty easily. But if it's an enterprise-level project and considering only using the community version to deploy on our infrastructure, it may be less straightforward compared to Airflow.


That's it for now. 

Thank you for reading and have a nice day!
