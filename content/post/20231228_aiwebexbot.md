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
      Features
</b></u>

- SQL Debugging and Optimization: The chatbot can understand SQL queries and help with debugging and optimization.
- Code Assistance: The chatbot can provide assistance with code-related issues.
- Jira Story Creation: The chatbot can create Jira stories based on user input.



<u>Usage</u>

To use the bot, send messages to it on Webex using the following command format:

For SQL debugging: 
```md 
!sqldebug <your SQL query> #error <your error message>
```

For SQL optimization:
```md 
!sqlopt <your SQL query> #opt <your optimization related message>
```

For SQL formatting: 
```md 
!sqlformat <your SQL query>
```

For programming assistance: 
```md 
!codeq -l <prog-lang> -q <your question or code>
```

To create a jira story: 
```md 
!jspost #{role} <story title>
```

Optional parameters: 
```md 
!jspost #{role} <story title> -u <user-id> -e <epic-id> -t <team-id> -sp <story-points> 
```

To get AI generated jira story content: 
```md 
!jsget #{role} <story title>
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

And that's it, here is the final working BOT example.

<p align="center">
<img alt = 'gif' src='/images/on_aiWebexBot/webex-bot-demo.gif'/>
</p>


[Code](https://github.com/FISHWONGY/webex-bot) for the Webx Chat BOT in this blog post.


<u><b>
    <p style="font-size:20pt ">
      Final Thoughts
</b></u>

As I am closing off my post now, I just can't help but think about how far we have come with Gen AI.

Let's take a look at the code I wrote in 2020 during lockdown to generate Shakespearean sonnet using TensorFlow.

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

Back then, I was already amazed at what TensorFlow could do. I could spend days playing with these AI sonnet generator for entertainment. I was so naive, haha.

Can't wait to see what lies ahead in 2024.


Thank you for reading and have a nice day!


