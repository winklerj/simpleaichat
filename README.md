# simpleaichat

```py3
from simpleaichat import AIChat

ai = AIChat(system="Write a fancy GitHub README based on the user-provided project name.")
ai("simpleaichat")
```

simpleaichat is a Python package for easily interfacing with chat apps like ChatGPT and GPT-4 with robust features and minimal code complexity. This tool has many features optimized for working with ChatGPT as fast and as cheap as possible, but still much more capable of modern AI tricks than most implementations:

- Create and run chats with only a few lines of code!
- Optimized workflows which minimize the amount of tokens used, reducing costs and latency.
- Run multiple independent chats at once.
- Minimal codebase: no code dives to figure out what's going on under the hood needed!
- Chat streaming responses and the ability to use tools.
- Async support, including for streaming and tools.
- Ablity to create more complex yet clear workflows if needed, such as Agents.
- Coming soon: more chat model support (PaLM, Claude)!

Here's some fun, hackable examples on how simpleaichat works:

- Creating a coding assistant (without any unnecessary accompanying output)
- Allowing simpleaichat to make selections for inline ChatGPT usage guidelines.
- Async interface for conducting many chats in the time it takes to receive one AI message.

## Installation

simpleaichat can be installed [from PyPI](https://pypi.org/project/simpleaichat/):

```sh
pip3 install simpleaichat
```

## Quick, Fun Demo

You can demo chat-apps very quickly with simpleaichat! First, you will need to get an OpenAI API key, and then with one line of code:

```py3
from simpleaichat import AIChat

AIChat(api_key="sk-...")
```

And with that, you'll be thrust directly into a chat!

This AI chat will mimic the behavior of OpenAI's webapp, but on your local computer!

You can also pass the API key by storing it in an `.env` file with a `OPEN_AI_KEY` field in the working directory (recommended), or by setting the environment variable of `OPEN_AI_KEY` directly to the API key.

But what about creating your own custom conversations? That's where things get fun. Just input whatever person, place or thing, fictional or nonfictional, that you want to chat with!

```py3
AIChat("GLaDOS")  # assuming API key loaded via methods above
```

But that's not all! You can customize exactly how they behave too with additional commands!

```py3
AIChat("GLaDOS", "Speak in the style of a Seinfeld monologue")
```

```py3
AIChat("Ronald McDonald", "Speak using only emoji")
```

Need some socialization immediately? Once simpleaichat is installed, you can also start these chats directly from the command line!

```sh
simpleaichat
simpleaichat "GlaDOS"
simpleaichat "GLaDOS" "Speak in the style of a Seinfeld monologue"
```

## Building AI-based Apps

The trick with working with new chat-based apps that wasn't readily available with earlier iterations of GPT-3 is the addition of the system prompt: a different class of prompt that guides the AI behavior throughout the entire conversation. In fact, the chat demos above are actually using system prompt tricks behind the scenes, and you can read how they work in PROMPTS.md.

For developers, you can instantiate a programmatic instance of `AIChat` by explicitly specifying a system prompt, or by disabling the console app.

```py3
ai = AIChat(system="You are a helpful assistant")
ai = AIChat(console=False)  # same as above
```

You can then feed the new `ai` class with user input, and it will return and save the response from ChatGPT:

```py3
response = ai("What is the capital of California?")
print(response)
```

```
The capital of California is Sacramento.
```

Alternatively, you can stream responses by token with a generator if the text generation itself is too slow:

```py3
for chunk in ai.stream("What is the capital of California?", params={"max_tokens": 5}):
    response_td = chunk["response"]  # dict contains "delta" for the new token and "response"
    print(response_td)
```

```
The
The capital
The capital of
The capital of California
The capital of California is
```

Further calls to the `ai` object will continue the chat, automatically incorporating previous information from the conversation.

```py3
response = ai("When was it founded?")
print(response)
```

```
Sacramento was founded on February 27, 1850.
```

You can also save chat sessions (as CSV or JSON) and load them later. The API key is (obviously) not saved so you will have to provide that when loading.

```py3
ai.save_session()  # CSV, will only save messages
ai.save_session(format="json", minify=True)  # JSON

ai.load_session("my.csv")
ai.load_session("my.json")
```

### Functions

A large number of popular venture-capital-funded ChatGPT apps don't actually use the "chat" part of the model. Instead, they just use the system prompt/first user prompt as a form of natural language programming. You can emulate this behavior by passing a new system prompt when generating text, and not saving the resulting messages.

The `AIChat` class is a manager of chat _sessions_, which means you can have multiple independent chats or functions happening! The examples above use a default session, but you can create new ones by specifying a `id` when calling `ai`.

```py3
json = '{"title": "An array of integers.", "array": [-1, 0, 1]}'
functions = [
             "Format the user-provided JSON as YAML.",
             "Write a limerick based on the user-provided JSON.",
             "Translate the user-provided JSON from English to French."
            ]
params = {"temperature": 0.0, "max_tokens": 100}  # a temperature of 0.0 is deterministic

# We namespace the function by `id` so it doesn't affect other chats.
# Settings set during session creation will apply to all generations from the session,
# but you can change them per-generation, as is the case with the `system` prompt here.
ai = AIChat(id="function", params=params, save_messages=False)
for function in functions:
    output = ai(json, id="function", system=function)
    print(output)
```

```txt
title: "An array of integers."
array:
  - -1
  - 0
  - 1
```

```txt
An array of integers so neat,
With values that can't be beat,
From negative to positive one,
It's a range that's quite fun,
This JSON is really quite sweet!
```

```txt
{"titre": "Un tableau d'entiers.", "tableau": [-1, 0, 1]}
```

### Tools

One of the most recent aspects of interacting with ChatGPT is the ability for the model to use "tools." As defined from the ReAct paper, tools allow the model to decide when to use custom functions, which can extend beyond just the chat AI itself, for example retrieving recent information from the internet not present in the chat AI's training data. This workflow is analogous to ChatGPT Plugins.

Parsing the model output to invoke tools typically requires a number of shennanigans, but simpleaichat uses [a neat trick](https://github.com/minimaxir/simpleaichat/blob/main/PROMPTS.md#tools) to make it fast and reliable! Additionally, the specified tools return a `context` for ChatGPT to draw from for its final response, and can return a dictionary which you can also populate with arbitrary metadata for debugging and postprocessing.

You will need to specify functions with docstrings which provide hints for the AI to select them:

```py3
from simpleaichat.utils import wikipedia_search, wikipedia_lookup

def search(query):
    """Search the internet."""
    wiki_matches = wikipedia_search(query, n=3)
    return {"context": "\n---\n".join(wiki_matches), "titles": wiki_matches}

def lookup(query):
    """Lookup more information about a topic."""
    page = wikipedia_search_lookup(query, sentences=3)
    return {"context": page, "titles": query}

params = {"temperature": 0.0, "max_tokens": 100}
ai = AIChat(params=params, console=False)

ai("What is a good tourist attraction in California?", tools=[search, lookup])
```

```txt
{'context': 'Tourist attraction\n---\nBuena Park, California\n---\nLombard Street (San Francisco)',
 'titles': ['Tourist attraction',
  'Buena Park, California',
  'Lombard Street (San Francisco)'],
 'tool': 'search',
 'response': "There are many great tourist attractions in California, but two popular ones are Buena Park, which is home to several theme parks such as Knott's Berry Farm and Disneyland, and Lombard Street in San Francisco, which is known for its steep, winding road and beautiful views of the city."}
```

```py3
ai("Lombard Street?", tools=[search, lookup])
```

```
{'context': 'Lombard Street is an east–west street in San Francisco, California that is famous for a steep, one-block section with eight hairpin turns. Stretching from The Presidio east to The Embarcadero (with a gap on Telegraph Hill), most of the street\'s western segment is a major thoroughfare designated as part of U.S. Route 101. The famous one-block section, claimed to be "the crookedest street in the world", is located along the eastern segment in the Russian Hill neighborhood.',
 'titles': 'Lombard Street?',
 'tool': 'lookup',
 'response': 'Yes, Lombard Street is a famous tourist attraction in San Francisco, California. It is known for its steep, one-block section with eight hairpin turns, which is claimed to be "the crookedest street in the world". The street is located in the Russian Hill neighborhood and is a popular spot for tourists to take photos and enjoy the beautiful views of the city.'}
```

```py3
ai("Thanks for your help!", tools=[search, lookup])
```

```txt
{'response': "You're welcome! If you have any more questions, feel free to ask.",
 'tool': None}
```

## Miscellaneous Notes

- Like [gpt-2-simple](https://github.com/minimaxir/gpt-2-simple) before it, the primary motivation behind releasing simpleaichat is to both democratize access to ChatGPT even more without extolling complexity as a virtue, and also offer more transparency for non-engineers into how Chat AI-based apps work under the hood given the disproportionate amount of media misinformation about their capabilities. This is inspired by real-world experience from [my work with BuzzFeed](https://tech.buzzfeed.com/the-right-tools-for-the-job-c05de96e949e) in the domain, where after spending a long time working with the popular LangChain, a more-simple implementation was both much easier to maintain and resulted in much better generations. I began focusing development on simpleaichat after reading a [Hacker News thread](https://news.ycombinator.com/item?id=35820931) filled with many similar complaints, indicating value for an easier-to-use interface for modern AI tricks.
  - simpleaichat very intentionally avoids coupling features with common use cases where possible (e.g. Tools) in order to avoid software lock-in due to the difficulty implementing anything not explicitly mentioned in the project's documentation. The philosophy behind simpleaichat is to provide good demos, and let the user's creativity and business needs take priority instead of having to fit a round peg into a square hole like with LangChain.
  - simpleaichat makes it easier to interface with Chat AIs, but it does not attempt to solve common technical and ethical problems inherent to large language models trained on the internet, including prompt injection and unintended plagiarism. The user should exercise good judgment when implementing simpleaichat. Use cases of simpleaichat which go against OpenAI's usage policy (including jailbreaking) will not be endorsed.
  - simpleaichat does not use the "Agent" logical metaphor for its actions. If needed be, you can emulate the Agent workflow with a `while` loop without much additional code, plus with the additional benefit of much more flexibility such as debugging.
- The session manager implements some sensible security defaults, such as using UUIDs as session ids by default, storing authentication information in a way to minimize unintentional leakage, and type enforcement via Pydantic. Your end-user application should still be aware of potential security issues, however.
- Although OpenAI's documentation says that system prompts are less effective than a user prompt constructed in a similar manner, in my experience it still does perform better for maintaining rules/a persona.
- Outside of the explicit examples, none of this README uses AI-generated text. The introduction code example is just a joke, but it was too good of a real-world use case!

## Roadmap

- PaLM Chat (Bard) and Anthropic Claude support
- More fun/feature-filled CLI chat app based on Textual
- Simple example of using simpleaichat in a webapp
- Simple of example of using simpleaichat in a stateless manner (e.g. AWS Lambda functions)
- Ability to modify the base tool workflow

## Maintainer/Creator

Max Woolf ([@minimaxir](https://minimaxir.com))

_Max's open-source projects are supported by his [Patreon](https://www.patreon.com/minimaxir) and [GitHub Sponsors](https://github.com/sponsors/minimaxir). If you found this project helpful, any monetary contributions to the Patreon are appreciated and will be put to good creative use._

## License

MIT
