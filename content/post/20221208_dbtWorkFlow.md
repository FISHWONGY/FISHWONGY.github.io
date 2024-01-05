---
authors:
- Hugo Authors
date: "2022-12-08"
excerpt: Integrating Github Actions with dbt
hero: /images/on_dbt_workflow/gh_actions.png
title: On DBT's CI/CD Workflow
---

<div style="text-align: justify">

[Last time](https://fishwongy.github.io/post/20221106_dbtsetup/) we talk about some of the basic dbt functionalities, in this blog post, I want to talk about how we can use github workflows to automate dbt.

For a lot of dbt-core users who are not using the dbt cloud version, integrating dbt with github workflows is an alternative to automate our models & tests run.

<u><b>
    <p style="font-size:20pt ">
      What is github workflow?
    </p>
</b></u>

According to the official [github website](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions), GitHub Actions is a continuous integration and continuous delivery (CI/CD) platform that allows you to automate your build, test, and deployment pipeline. You can create workflows that build and test every pull request to your repository, or deploy merged pull requests to production.


<u><b>
    <p style="font-size:20pt ">
      Set up dbt with github workflow
    </p>
</b></u>

1 - Set up workflow yml file

On you github dbt project repo, create a new folder `.github/workflows/dbt_job.yml`.
Alternatively, you can also directly click on Actions > New actions, to import the workflow yml.

Under `dbt_job.yml`, we will have to add below.

```yml
name: schedule_dbt_job

on:
  schedule:
     - cron: 0 7 * * *
env:
  DBT_PROFILES_DIR: ./
  
  DBT_SNOWFLAKE_ACCOUNT: ${{ secrets.DBT_SNOWFLAKE_ACCOUNT }}
  DBT_SNOWFLAKE_USERNAME: ${{ secrets.DBT_SNOWFLAKE_USERNAME }}
  DBT_SNOWFLAKE_PW: ${{ secrets.DBT_SNOWFLAKE_PW }}
  DBT_SNOWFLAKE_ROLE: ${{ secrets.DBT_SNOWFLAKE_ROLE }}
  DBT_SNOWFLAKE_DATABASE: ${{ secrets.DBT_SNOWFLAKE_DATABASE }}
  DBT_SNOWFLAKE_WAREHOUSE: ${{ secrets.DBT_SNOWFLAKE_WAREHOUSE }}
  DBT_SNOWFLAKE_SCHEMA: ${{ secrets.DBT_SNOWFLAKE_SCHEMA }}


jobs:
  schedule_dbt_job:
   runs-on: self-hosted
   steps:
     - name: Check out
       uses: actions/checkout@v3

     - name: Set up Python
       uses: actions/setup-python@v4
       with:
         python-version: "3.9.6"
         
  # pip install dbt
     - name: Install dependencies
       run: |
         pip install dbt-core==1.3.1
         pip install dbt-snowflake==1.3.0
         dbt deps

   # dbt related commands - 
     - name: Run dbt models
       run: dbt run --project-dir bronze_schema --models folder.subfolder.model_name

     - name: Test dbt models
       run: dbt test --project-dir gold_schema --select folder.subfolder.model_name
```


2 - Set up secrets/ env var for job

Navigate to Settings > Security > Actions, add all your secrets and/ or env variables for the job.

<p align="center">
<img alt = 'png' src='/images/on_dbt_workflow/gh_actions_var.png'/>
</p>


3 - Set up profiles.yml 
On the root of the dbt github repo, create a `profiles.yml`

It will look somthing like this

```yml
bronze_schema:
  outputs:
    dev:
    threads: 1
    type: snowflake
    account: "{{ env_var('DBT_SNOWFLAKE_ACCOUNT') }}"
    user: "{{ env_var('DBT_SNOWFLAKE_USERNAME') }}"
    role: "{{ env_var('DBT_SNOWFLAKE_ROLE') }}"
    password: "{{ env_var('DBT_SNOWFLAKE_PW') }}"
    database: "{{ env_var('DBT_SNOWFLAKE_DATABASE') }}"
    warehouse: "{{ env_var('DBT_SNOWFLAKE_WAREHOUSE') }}"
    schema: "{{ env_var('DBT_SNOWFLAKE_SCHEMA_BRONZE') }}"
    client_session_keep_alive: False
```


And that's it, you will have a CI/CD dbt workflow deployed with github actions.

---
Some final thoughts...

This is just one use case; there's so much more GitHub Actions can do, such as ensuring code quality and security on branches based on git push, automating deployment on merging branches, etc.

That's it for now.

Thank you for reading, and have a nice day!

