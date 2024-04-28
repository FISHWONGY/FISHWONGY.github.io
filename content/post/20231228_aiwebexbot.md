---
authors:
- Hugo Authors
date: "2023-12-28"
excerpt: Read about how I built an AI Webex Chat BOT integrating with OpenAI's API
hero: /images/on_aiWebexBot/aibot_cover.png
title: On AI Webex Chat BOT
---

<br />
<br />


<div style="text-align: justify">

I like to build chatbots, because it is fun. 

In the past, I have built chatbots with application like [line](https://github.com/FISHWONGY/line-bot), and [discord](https://github.com/FISHWONGY/discord-bot) 

With Generative AI (Gen AI) being the hot topic this year, I have always wanted to build something I am interested in with it. During this Christmas break, I finally have the time to sit down and build it 🤓.

<u><b>
    <p style="font-size:20pt ">
      Overview
    </p>
</b></u>

<p align="center">
<img alt = 'gif' src='/images/on_aiWebexBot/webex-bot-demo.gif'/>
</p>


The Webex Chatbot uses OpenAI's GPT-3.5 model to provide automated responses. It was developed in Python and uses the Azure OpenAI API, Webex API and Jira API for chat operations and its core functionalities.


<u>Architecture</u>

<p align="center">
<img alt = 'png' src='/images/on_aiWebexBot/bot_workflow.png'/>
</p>


<u><b>
    <p style="font-size:20pt ">
      Building a Webex Bot
    </p>
</b></u>

First of all, lets set up our environment

pyproject.toml
```toml
[tool.poetry]
name = "webex-gpt-bot"
version = "0.1.0"
description = ""
authors = [""]
readme = "README.md"

[tool.poetry.dependencies]
python = "3.9"
webex-bot = "^0.4.1"
webexteamssdk = "^1.6.1"
websockets = "^10.2"
emoji = "^2.9.0"

openai = "^1.6.1"
langchain = "0.1.4"
langchain-openai= "^0.0.5"
requests = "^2.31.0"

[build-system]
requires = ["poetry-core>=1.4.1"]
build-backend = "poetry.core.masonry.api"
```

<u><b>
Building webex commands
</b></u>

Let's start with building a bot command that uses the OpenAI API to reformat our SQL code.


```python
from webex_bot.models.command import Command

class SqlFormatter(Command):
    def __init__(self):
        super().__init__(
            command_keyword="!sqlformat",
            help_message=f"Unrecognised command.\n"
            f"Please type !sqlformat followed by your snowflake query",
            card=None,
        )

    def pre_execute(self, message, attachment_actions, activity):
        return "<blockquote class=info>\n\n**Command received.**\n\n\n**Working on SQL Reformatting...**\n\n</blockquote>"

    def execute(self, message, attachment_actions, activity):
        q_str = message.strip().lower()

        if q_str == "":
            response_message = (
                "Invalid input format. \nPlease type !sqlformat followed by your snowflake query."
                "\nExample: \n\n```\n!sqlformat SELECT col1, col2, col3 FROM tbl\n\n```"
            )
        else:
            msg_hist = openaiapi.construct_formatter_prompt(query=q_str)
            response_message = openaiapi.chat_response(msg_hist)

        return response_message
```

Of course, we can also have a general programming assistant
```python
class CodeHelp(Command):
    def __init__(self):
        super().__init__(
            command_keyword="!codeq",
            help_message="Unrecognised command.\nPlease type !codeq followed by -l {lang} and -q {question}.",
            card=None,
        )

    def execute(self, message, attachment_actions, activity):
        try:
            lang = extract_value(r"-l\s*([\w\s]+)", message)
            question = extract_value(r"-q\s*(.+)", message)

            if not lang or not question:
                raise ValueError

            msg_hist = openaiapi.construct_code_help_prompt(lang, question)
            response_message = openaiapi.chat_response(msg_hist)
        except ValueError:
            response_message = (
                "Invalid input format. \nPlease type !codeq followed by -l {lang} with the programming language you are enquiring and -q {question} with your question."
                "\nExample: \n\n```\n!codeq -l python -q How can I print hello world?\n```"
                "\nOr"
                "\n```\n!codeq -q How can I print hello world? -l python\n```"
            )

        return response_message
```


Finally, as an optional, we can have a custom help command instead of the default ones from the webexbot library

```python
from webexteamssdk.models.cards import (
    TextBlock,
    Column,
    AdaptiveCard,
    ColumnSet,
)
from webexteamssdk.models.cards.actions import Submit, OpenUrl
from webex_bot.models.command import Command
from webex_bot.models.response import response_from_adaptive_card

class HelpBase(Command):
    def __init__(self):
        super().__init__(
            command_keyword="!customise",
            card=None,
        )

    def execute(self, message, attachment_actions, activity):
        message = message.strip().lower()
        if message == "sqlformatet":
            help_message = (
                f"{'-' * 80}\n"
                "**Command:** !sqlformat followed by &lt;SQL-query&gt;\n"
                "**Usage:** Reformat SQL for better readability"
                "\n\nExample: \n\n```\n!sqlformat SELECT col FROM tbl\n```"
                f"\n{'-' * 80}\n"
            )
        elif message == "codeq":
            help_message = (
                f"{'-' * 80}\n"
                "**Command:** !codeq followed by -l &lt;programming-language&gt; -q &lt;question&gt;\n"
                "**Usage:** Programming Assistant for programming languages of your choice"
                "\n\nExample: "
                "\n\n```\n!codeq -l python -q How can I print hello world?\n```"
                "\nOr"
                "\n```\n!codeq -q How can I print hello world? -l python\n```"
                f"\n{'-' * 80}\n"
        else:
            help_message = "Unrecognised command.\nPlease try again."

        return help_message


class HelpCommand(Command):
    def __init__(self):
        super().__init__(
            command_keyword="!help",
            help_message="!help - Shows help information",
            card=None,
        )

    def execute(self, message, attachment_actions, activity):
        actions = [
            Submit(
                title="!sqlformat - SQL Formatting\n!sqlformat <snowflake-query>",
                data={"command_keyword": "!customise sqlformat"},
            ),
            Submit(
                title="!codeq - Programming Assistant\n!codeq -l <lang> -q <question>",
                data={"command_keyword": "!customise codeq"},
            ),
        ]

        card = AdaptiveCard(
            body=[
                ColumnSet(
                    columns=[
                        Column(
                            items=[
                                TextBlock(
                                    text=emoji.emojize(
                                        ":computer_mouse: Click a command to get help"
                                    ),
                                    size="extraLarge",
                                    weight="Bolder",
                                )
                            ]
                        ),
                    ]
                ),
                TextBlock(
                    text=emoji.emojize(
                        ":speaking_head: Questions? Reach out to [Me](mailto:email@email.com)",
                    ),
                    separator=True,
                ),
            ],
            actions=actions,
        )
        return response_from_adaptive_card(card)
```


<u>Usage</u>

To use the bot, send messages to it on Webex using the following command format:

For SQL formatting: 
```md 
!sqlformat <your SQL query>
```

For programming assistance: 
```md 
!codeq -l <prog-lang> -q <your question or code>
```


<u>Design Explanations</u>


We can easily see that the commands for different functions are slightly different; while some commands use `-` for extra parameters, some use `#` for extra optional parameters.

To be honest, I'd prefer to use `-` for all optional params so that it is more similar to a proper CLI. However, when working with languages like SQL, the use of `-` is common inside the code.


```sql
-- For example when we are commenting
SELECT *
FROM tbl
```

I may be able to get away with it if I go down to the route of using explicit [regex](https://learn.microsoft.com/en-us/dotnet/standard/base-types/regular-expression-language-quick-reference) , however considering how often  ` -- something` is used in SQL syntax, I decided to just use `#` to resolve the issue once and for all this time.

---

<u><b>
    <p style="font-size:20pt ">
      Prompt Engineering
</b></u>


Prompting is important when working with Gen AI, in fact, Open AI has published a [guide](https://platform.openai.com/docs/guides/prompt-engineering/strategy-write-clear-instructions) for better prompt engineering.

For the project this time, I have tried the below tactics.

- Start with building a personality
Setting
```md
You are a <role>, you are good at <subject>, your job is to <action>
```

- Giving specific example on the expected answer or outcome
```md
For example, I will ask you a question: <question>.
You have to answer as the below strcuture, <provide-detailed-answer-structure>
```

- Positivity & Compliment


It could be frustrating when Gen AI is constantly not giving the output we are looking for; I get it. But it is better to sound positive for better Gen AI performance.

```md
Good job, we are getting there slowly. <part-of-the-gen-ai-response> is what I want, however ... <explicit-comment-on-desired-output>
```

Here is an example of the prompting function that I have for the bot. I tested it many times with different inputs to observe its response, and then I adjusted the user-assistant prompt based on the answer.

```python
def construct_prompt(query: str) -> list:
    system_prompt = f"{prompt.INITIAL_PROMPT}"
    q = f"""{query}"""
    user_prompt = f"""Good job, let's keep it going!\nRemember the pattern is <something>.\nHere is the next <question>\nPlease make sure you MUST answer in <output-format>'"""

    return [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": prompt.USER_Q1_PROMPT},
        {"role": "assistant", "content": prompt.ASSIST_ANS1_PROMPT},
        {"role": "user", "content": prompt.USER_Q2_PROMPT},
        {"role": "assistant", "content": prompt.ASSIST_ANS2_PROMPT},
        {"role": "user", "content": prompt.USER_Q3_PROMPT},
        {"role": "assistant", "content": prompt.ASSIST_ANS3_PROMPT},
        {"role": "user", "content": prompt.USER_Q4_PROMPT},
        {"role": "assistant", "content": prompt.ASSIST_ANS4_PROMPT},
        {"role": "user", "content": prompt.USER_Q5_PROMPT},
        {"role": "assistant", "content": prompt.ASSIST_ANS5_PROMPT},
        {"role": "user", "content": user_prompt},
    ]
```

And that's it, here is the final working BOT example. and [here](https://github.com/FISHWONGY/webex-gpt-bot) is the code

<p align="center">
<img alt = 'gif' src='/images/on_aiWebexBot/webex-bot-demo.gif'/>
</p>


<u><b>
    <p style="font-size:20pt ">
      Final Thoughts
</b></u>

As I am closing off my post now, I just can't help but think about how far we have come with Gen AI.

Let's take a look at the code back in 2020 during lockdown to generate Shakespearean sonnet using TensorFlow.

```python
# Generating sonnet

def build_model(vocab_size, embedding_dim, rnn_units, batch_size):
    model = tf.keras.Sequential([
        tf.keras.layers.Embedding(vocab_size, embedding_dim,
                                  batch_input_shape=[batch_size, None]),
        tf.keras.layers.GRU(rnn_units,
                            return_sequences=True,
                            stateful=True,
                            recurrent_initializer='glorot_uniform'),
        tf.keras.layers.Dense(vocab_size)
    ])
    return model
    
def generate_text(model, start_string):
  num_generate = 1000

  input_eval = [char2idx[s] for s in start_string]
  input_eval = tf.expand_dims(input_eval, 0)

  text_generated = []

  temperature = 1.0

  model.reset_states()
  for i in range(num_generate):
      predictions = model(input_eval)

      predictions = tf.squeeze(predictions, 0)
      predictions = predictions / temperature
      predicted_id = tf.random.categorical(predictions, num_samples=1)[-1,0].numpy()

      input_eval = tf.expand_dims([predicted_id], 0)

      text_generated.append(idx2char[predicted_id])

  return (start_string + ''.join(text_generated))
```

Some of the sonnet output example - 

```md
Is my will comb and cold and said upon!
Let my speech Lenry, coward like an oath:
Prenicious tyrant, hark you with it on:
And after hours with scarces seem'd both,
No, sir: yet you must deposed; scarcest what
Though not, let them curtsy was seat of bright
Not too keep for all one fair ladies, but
That he shall have one softer laugh at light,
Yet do too proud to most armoud, were shut;
Live, thy life that breaks Bianca rather:
Not too keep for all one fair ladies, but
Nay, I am Colit before and brother
Than to a mile before you fear? yes there?
That, my daughter my bosoms seek is there?
```

```md
I wis it to touch a se on thy fore--
Sir, this is not the wisdom of thy part.
Though not amazed the Lucentio's war,
Shords that will make me swear upon my heart,
Fear not the ladwers that contains use you
The honourable father of a day.
twenty-three armon's corse that e'er man do
The travel out of honour means to-day?
Why, then 'tis but omit me it had been
is full of something that afrailst with you,
Do this fair entranction to do't for sin.
Hail, their judges he should have 'em to
That hidon complots in the regal throne.
His golden rejembers sink in its own
```

I have also tried to play with generating Shakespeare's play using tf, and below is one of the result.

```md
LORD WISTBROKEO:
What if this request?

KATHARIIA:
He's gone.

Shepherd:
Pray now, I do that revenge!

HENRY BOLINGBROKE:
By this I only so.

PROSPERO:
Stay: I'll give formord of Gaulina!
A party wrong, speak lose the stores of mine.
Advanity, one Richard; then forget were r
trust than mercy: what will you ather
comes his as it is!
```

Back then, I was already amazed at what TensorFlow could do. I could spend days playing with these AI sonnet generator for entertainment.

Can't wait to see what lies ahead in 2024.


Thank you for reading and have a nice day!


