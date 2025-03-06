---
title: AI Agents with Gemini and Google Colab
description: >-
  Creating a simple AI agent using Gemini and Google Colab.
date: 2025-03-06 08:40:00 +0100
categories: [Python]
tags: [python,ai-agents,google-colab]
tok: true
image:
  path: /assets/img/illustrations/ai-agents-google-colab.jpg
  alt: AI Agents
---

---
So far, I managed to keep myself away from the whole AI hype that is currently going on, because I thought most of it was really smoke and mirrors.

However, I was recently involved in a conversation with some colleagues and to my surprise I was utterly lost!

I am not a star engineer by any means, but I do like to keep up with the latest trends in technology, so naturally I tend to have a basic understanding of what is happening.

But that was not the case this time!

They were talking about M&Ms, llamas, alpacas, secret agents, hugging faces, rug pulls.. I'd swear I even heard references to Jean Claude Van Damme for a moment.

It became clear very quickly that I was left behind.. Hype or no hype, I knew I had to keep up with at least the basics.

So, long story short, I decided to start learning about AI agents, since that was the main focus in our chat.

## Large Language Models (LLMs)
Before we start with the agents, let's explain what an LLM is.

Given that I only just started with AI, I will avoid giving technical definitions out of fear they might be entirely wrong..

That being said, an LLM to me is basically an artificial brain. It's a pre-trained software that accepts text as input and attempts to auto complete it, based on the large amounts of data that it encountered during its training phase.

This gives the impression that it thinks and replies with an output. Don't quote me on that though..

## AI Agents
An AI agent is a program that has the ability to interact with the API of an LLM and essentially feed it questions and parse its answers.

The interesting thing though is that it is a wrapper around the LLM, which allows it to inject additional functionality that the LLM itself doesn't provide.

For example, given that an LLM is an isolated, pre-trained program, there is no way for it to know today's weather, unless someone else provides it.

And this is precisely what the agent is doing. Whenever a question about the weather arises, the agent can reach to the outside world, get the information and inject it back to the LLM's context.

Combining agents with LLMs makes the overall AI solution appear as if it can:
- Listen
- Think
- Use tools
- Answer

## Our first agent
In this example, we will create an AI agent that enhances an LLM with the ability to query live information about crypto currencies, such as prices and market caps.

We will use the [gemini-2.0-flash](https://deepmind.google/technologies/gemini/flash/) LLM, but I believe you can choose any Google model you want.

The best thing is that we can run our code in [Google Colab](https://colab.research.google.com/) as an interactive python notebook!

So go ahead and create a new file in your Google Colab account.

#### Install required libraries
The first library we need to install is the Python SDK for the Gemini API, contained in the [`google-generativeai`](https://pypi.org/project/google-generativeai/) package.

Additionally, we will need to install the [`requests`](https://pypi.org/project/requests/) library in order to be able to make HTTP requests to the [CoinMarketCap API](https://pro.coinmarketcap.com/api/v1?source=post_page---------------------------#operation/getV2CryptocurrencyQuotesLatest).

```bash
!pip install -q -U google-generativeai requests
```

If you are not using Google Colab, you will probably want to create a virtual environment to install these packages.

#### Load the secret API keys
Now, we need to load the API keys for the Gemini and the CoinMarketCap APIs.

You can generate your own keys for Gemini [here](https://aistudio.google.com/app/apikey) and for CoinMarketCap [here](https://pro.coinmarketcap.com/account).

Once you have your keys, add them in your secrets, on the left side panel of your Google Colab project. Make sure the names match the snippet below.

```python
from google.colab import userdata


GEMINI_API_KEY=userdata.get("GEMINI_API_KEY")
COINMARKETCAP_API_KEY=userdata.get("COINMARKETCAP_API_KEY")
```

If you don't want to register an account with CoinMarketCap just for testing it, you can simply mock the request later on.

#### Create a tool function
Tools are essentially functions that can be used by the agent in order to enhance the LLM's capabilities.

In this case, we can create a function that, given a list of crypto currency names, it makes a request to the CoinMarketCap API and returns a response.

```python
import json
from typing import Dict, List

from pydantic import BaseModel
import requests


COINMARKETCAP_API_URL = "https://pro-api.coinmarketcap.com/v2/cryptocurrency/quotes/latest"

class CryptoCurrency(BaseModel):
    name: str
    price: float
    total_supply: float
    market_cap: float


def get_currencies(currency_names: List[str]) -> List[CryptoCurrency]:
    """Given a list of crypto currency names, it makes a request to the
    CoinMarketCap API and returns a list of currency objects with their
    latest details, such as prices.

    Args:
        currency_names: A list of crypto currency names to retrieve.

    Returns:
        A list of CryptoCurrency objects with their latest details.
    """
    params = {"slug": ",".join(currency_names)}
    headers = {
        "Accepts": "application/json",
        "X-CMC_PRO_API_KEY": COINMARKETCAP_API_KEY,
    }

    crypto_currencies = []
    try:
        response = requests.get(COINMARKETCAP_API_URL, params=params, headers=headers)
        data = json.loads(response.text)

        for _, currency_data in data["data"].items():
            crypto_currencies.append(
                CryptoCurrency(
                    name=currency_data["name"],
                    price=currency_data["quote"]["USD"]["price"],
                    total_supply=currency_data["total_supply"],
                    market_cap=currency_data["quote"]["USD"]["market_cap"],
                )
            )

    except Exception as exc:
        print(exc)

    return crypto_currencies
```
You can read more about function calling [here](https://ai.google.dev/gemini-api/docs/function-calling), but note that the docstrings and the type annotations play a very significant role in the behavior of the agent.

#### Instantiate the agent
Now, we can instantiate the agent by choosing the required LLM model and providing the add-on tool that we created.

```python
from google import genai
from google.genai.types import FunctionDeclaration, GenerateContentConfig, Part, Tool


client = genai.Client(api_key=GEMINI_API_KEY)
chat = client.chats.create(
    model="gemini-2.0-flash",
    config=GenerateContentConfig(
        tools=[
            get_currencies,
        ]
    )
)
```

#### Interact with the agent
Finally, we can interact with the agent by sending messages.

Let's ask it to provide today's summary for Ethereum and Bitcoin.
```python
text = "Give me a summary of the latest stats for ethereum and bitcoin today"
response = chat.send_message(text)
print(response.text)
```

```
Here's a summary of the latest stats for Bitcoin and Ethereum:

Bitcoin:

  Price: 90877.53
  MarketCap: 1,802,366,934,215.69
  Total Supply: 19,832,921

Ethereum:

  Price: 2283.39
  MarketCap: 275,365,843,884.19
  Total Supply: 120,594,948.93
```

Now let's ask it whether it can convert these prices in USD.
```python
text = "Can you convert the prices in USD?"
response = chat.send_message(text)
print(response.text)
```

```
The prices I provided for Bitcoin and Ethereum are already in United States Dollars (USD). There is no conversion needed.
```

Clever enough, right?

## Where is the magic?
Overall, there is nothing fancy here. The code is rather boring and straight forward.

However, one might wonder, how was the program able to understand that it should:
- extract the `bitcoin` and `ethereum` strings from the text
- feed these to the `get_currencies` function
- use the HTTP response to generate a reply to the initial question

Well, the magic appears to happen in the docstrings and type annotations of the function.

The LLM is somehow contextualizing these information and tries to understand how they can be used.

As a result, when a question about cryptocurrencies arises, it attempts to find arguments that can be used to call the `get_currencies` function.

And that is true magic there, as it showcases the remarkable ability of LLMs to understand the relationships between words and phrases and create content.

## Conclusion
I am pretty sure I haven't even touched the surface yet, but at least I now have a slightly better understanding of LLMs and AI agents in general.

As I mentioned earlier, the easiest way to experiment is to run these snippets in Google Colab.

Also, don't forget to check out Google's [Gemini documentation](https://ai.google.dev/gemini-api/docs) as it has a lot of interesting treats in there!
