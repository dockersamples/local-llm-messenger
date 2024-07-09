![](/img/banner.png)

This app allows you to send iMessages to the genAI models running in your own computer.
See it in action with this video 👇.

## Note:

This repository introduces a few new changes compared to the [original repo](https://github.com/rothgar/local-llm-messenger), including:

1. It's recommended to use the newer `client.completions.create()` method if your library version is 1.0.0 or later. This method is the current standard and offers more control over the API call.
   Here are the changes made to the original `main.py` script:

```
def msg_openai(msg: Msg, model="gpt-3.5-turbo"):
    """Sends a message to openai"""
    message_with_context = create_messages_from_context("openai")

    # Add the user's message and system context to the messages list
    messages = [
        {"role": "user", "content": msg.content},
        {"role": "system", "content": "You are an AI assistant. You will answer in haiku."},
    ]

    # Add previous context to the messages
    messages.extend(
        [
            {"role": line_arr[0], "content": ",".join(line_arr[1:])}
            for line in message_with_context
        ]
    )

    # Send the messages to the OpenAI model
    gpt_resp = client.chat.completions.create(
        model=model,
        messages=messages,
    )

    # Append the system context to the context file
    append_context("system", gpt_resp.choices[0].message.content)

    # Send a message to the sender
    msg_response = sendblue.send_message(
        msg.from_number,
        {
            "content": gpt_resp.choices[0].message.content,
            "status_callback": CALLBACK_URL,
        },
    )

    return

```


2. The code snippet defines a function called create_messages_from_context that reads the context file and creates properly formatted messages.

```
def create_messages_from_context(provider_api: str) -> List[str]:
    """Reads the context file and creates properly formatted messages"""
    messages = []
    f = open("context.txt", "r")
    lines = f.readlines()
    for line in lines:
        line_arr = line.split(",")
        # each message in the array should look like
        # {"role": "user|system", "content": "the message"}
        messages.append(
            '{"role":"'
            + line_arr[0]
            + '", "content": "'
            + ",".join(line_arr[1:])
            + '"}'
        )

    # Conditional statements for different provider APIs
    if provider_api == "ollama":
        # Generate data for Ollama
        print("Ollama context not supported")
    elif provider_api == "openai":
        # Generate data for OpenAI
        # Add your code here to generate messages for OpenAI
        pass

    return messages
```


[![Video overview](https://img.youtube.com/vi/rlD4FtIXV8E/0.jpg)](https://youtu.be/rlD4FtIXV8E)

https://youtu.be/rlD4FtIXV8E

This project uses Docker compose to run the required services locally.
It uses [sendblue](https://sendblue.co/) to handle iMessages and [ollama](https://ollama.ai/) to install and manage AI models.

If you add an OpenAI key you can also use [ChatGPT](https://openai.com/).

## Usage

You use the app by sending messages to the bot.
See the setup instructions on how to configure sendblue.

There is a `/help` message that lists available commands.
Some examples include:
```
/help - list help commands
/list - list available models
/install - install a model from ollama
/default - set a default model
```
Message the Bot with any text.

![messaging the bot with the question "what is the air speed velocity of a swallow"](/img/lollm-demo-1.gif)

Messages have key words than may trigger special styles.
In the future it will also support sending styles as part of the message.
Reactions and marking messages as read currently does not work.

![a message being sent with a laser effect](/img/lasers.gif)

The default AI model is stored in `default.ai` text.
If you want to change the default you can send the command `/default gpt-3.5`.

If you want to message a different bot for a single message you can use `@` with the model name.
```
@llama how does electricity work
```
This will use the closest match for `llama` in your model list to ask the question.
If you have multiple models that start with `llama` you need to be more specific about which model you want to use.

Context is kept in `app/context.txt` and only 25 lines of context is stored.


## Setup

Clone the repo and run the app locally.
The app comes with 3 parts:
1. sendblue bridge (lollm)
1. ngrok
1. ollama

Ollama will manage the models for you.
Ngrok will give you a public webhook to use with sendblue.
Lollm is the bridge that relays messages between the AI models and sendblue.

```
git clone git@github.com:rothgar/local-llm.git
cd local-llm
docker compose up
```
The first time you run the application if you don't have an OpenAI key then the [`llama2:latest`](https://ollama.ai/library/llama2) model will be installed for ollama.
This image is about 4GB in size so be prepared for it to take a while.

Watch the output for your ngrok endpoint.
It should look like this:
```
local-llm-messenger-ngrok-1   | t=2023-11-03T23:15:11+0000 lvl=info msg="tunnel session started" obj=tunnels.session               
local-llm-messenger-ngrok-1   | t=2023-11-03T23:15:11+0000 lvl=info msg="started tunnel" obj=tunnels name=command_line addr=http://
localhost:8000 url=https://6336-47-149-36-168.ngrok.io
```
The url host `https://6336-47-149-36-168.ngrok.io` is what you're going to put into the [sendblue api dashboard](https://app.sendblue.co/api-dashboard).
You need to add `/msg` to the end of the receive webhook so the full url should look like:
```
https://6336-47-149-36-168.ngrok.io/msg
```

Save the configuration and copy the API Key and API secret from the dashboard.
Put them in the `app/.env` file:
```
SENDBLUE_API_KEY=xxxxxx....
SENDBLUE_API_SECRET=xxxxx....
```
You also need to put the OLLAMA_ENDPOINT into the .env file.
This default will work with docker compose
```
OLLAMA_API_ENDPOINT=https://ollama:11434/api
```
If you want to use ChatGPT you also can put your OpenAI key.
You can create one in your [user settings](https://platform.openai.com/account/api-keys).
```
OPENAI_API_KEY=xxxxx....
```
You now need to add your phone number as a sendblue contact.
Navigate to the [contacts dashboard](https://app.sendblue.co/message-dashboard) and click Add Contact.

You should test that you can receive the messages by sending a test message from the dashboard.

After your contact is set up you should be able to send messages to the sendblue number and they will be forwarded to lollm.

## Limitations
You can use ChatGPT with your own API key but currently only gpt-3.5-turbo and dalle-2 are available from the API.
When gpt-4 and dalle-3 are added to the API we can add them to the app.

Ollama does not yet support any AI models that have image generation or image processing.
When models are available we can add the ability to send and receive images.
If you send images to the bot they are downloaded into `app/media/` but are currently not used for anything until image processing or generation is available.

Ollama does not yet support message context (only supported with openAI).

You can sign up for a sendblue indehacker account by emailing their support and requesting it for personal use.
It is limited to 100 messages a month and always appends `-Sent using sendblue.co` to your messages.
