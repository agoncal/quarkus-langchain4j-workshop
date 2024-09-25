# Step 01 - Introduction to Quarkus LangChain4J

To get started, make sure you use the `step-01` directory.

This step is the starting point for the workshop.
It's a simple Quarkus application that uses the Quarkus LangChain4J extension to interact with OpenAI's GPT-4o model.
It's a simple chatbot that we will extend in the subsequent steps.

## Running the application

==You can run it as follows by launching `./mvnw quarkus:dev`.==

??? note "mvnw permission issue"
    If you run into an error about the `mvnw` maven wrapper, you can give execution permission for the file by navigating to the project folder and executing `chmod +x mvnw`.

??? note "Could not expand value OPENAI_API_KEY"
    If you run into an error indicating `java.util.NoSuchElementException: SRCFG00011: Could not expand value OPENAI_API_KEY in property quarkus.langchain4j.openai.api-key`, make sure you have set the environment variable `OPENAI_API_KEY` with your OpenAI API key.

This will bring up the page at http://localhost:8080. 
Open it and click the red robot icon in the bottom right corner to start chatting with the chatbot.

![Miles of Smiles UI](images/ui-no-chatbot.png)

## Chatting with the chatbot

The chatbot is calling GPT-4o (from OpenAI) via the backend. 
You can test it out and observe that it has memory.
Example:

```
User: My name is Clement.
AI: Hi Clement, nice to meet you.
User: What is my name?
AI: Your name is Clement.
```

![An example of discussion with the chatbot](images/ui.png)

This is how memory is built up for LLMs.
==In the terminal, you can observe the calls that are made to OpenAI behind the scenes. Notice the roles 'user' (`UserMessage`) and 'assistant' (`AiMessage`).==

```bash
# The request -> Sending a message to the LLM
2024-09-16 08:33:24,600 INFO  [io.qua.lan.ope.OpenAiRestApi$OpenAiClientLogger] (vert.x-eventloop-thread-0) Request:
- method: POST
- url: https://api.openai.com/v1/chat/completions
- headers: [Accept: application/json], [Authorization: Be...ex], [Content-Type: application/json], [User-Agent: langchain4j-openai], [content-length: 378]
- body: {
  "model" : "gpt-4o",
  # The conversation so far, including the latest messages
  "messages" : [ {
    "role" : "user", # The role of the message (user or assistant)
    "content" : "My name is Clement."
  }, {
    "role" : "assistant", # Assistant means LLM
    "content" : "Hello, Clement! How can I assist you today?"
  }, {
    "role" : "user", # User means the user (you)
    "content" : "What is my name?"
  } ],
  "temperature" : 1.0,
  "top_p" : 1.0,
  "presence_penalty" : 0.0,
  "frequency_penalty" : 0.0
}

# The response from the LLM
2024-09-16 08:33:25,013 INFO  [io.qua.lan.ope.OpenAiRestApi$OpenAiClientLogger] (vert.x-eventloop-thread-0) Response:
- status code: 200
- headers: [Date: Mon, 16 Sep 2024 06:33:25 GMT], [Content-Type: application/json], [Transfer-Encoding: chunked], [Connection: keep-alive], [access-control-expose-headers: X-Request-ID], [openai-organization: user-vyycjqq0phctctikkw1zawlm], [openai-processing-ms: 213], [openai-version: 2020-10-01], [strict-transport-security: max-age=15552000; includeSubDomains; preload], [x-ratelimit-limit-requests: 500], [x-ratelimit-limit-tokens: 30000], [x-ratelimit-remaining-requests: 499], [x-ratelimit-remaining-tokens: 29958], [x-ratelimit-reset-requests: 120ms], [x-ratelimit-reset-tokens: 84ms], [x-request-id: req_2ea6d71590bc8d857260b25d9f414c0c], [CF-Cache-Status: DYNAMIC], [Set-Cookie: __...ne], [X-Content-Type-Options: nosniff], [Set-Cookie: _c...ne], [Server: cloudflare], [CF-RAY: 8c3ed3291afc27b2-LYS], [alt-svc: h3=":443"; ma=86400]
- body: {
  "id": "chatcmpl-A7zaWTn1uMzq7Stw50Ug2Pg9TkBpV",
  "object": "chat.completion",
  "created": 1726468404,
  "model": "gpt-4o-2024-05-13",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Your name is Clement. How can I help you today?",
        "refusal": null
      },
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 44,
    "completion_tokens": 12,
    "total_tokens": 56,
    "completion_tokens_details": {
      "reasoning_tokens": 0
    }
  },
  "system_fingerprint": "fp_25624ae3a5"
}
```

A very important aspect of the interaction with LLMs is their statelessness.
To build a conversation, you need to _resend_ the full list of messages exchanged so far.
That list includes both the user and the assistant messages.
This is how the memory is built up and how the LLM can provide contextually relevant responses.
We will see how to manage this in the subsequent steps.

## Anatomy of the application

Before going further, let's take a look at the code.

If you open the `pom.xml` file, you will see that the project is a Quarkus application with the `quarkus-langchain4j` extension.

```xml
<dependency>
    <groupId>io.quarkiverse.langchain4j</groupId>
    <artifactId>quarkus-langchain4j-openai</artifactId>
    <version>${quarkus-langchain4j.version}</version>
</dependency>
```

Quarkus LangChain4J OpenAI is a Quarkus extension that provides a simple way to interact with language models (LLMs) like GPT-4o from OpenAI.
It actually can interact with any model serving the OpenAI API (like vLLM or Podman AI Lab).
Quarkus Langchain4J abstracts the complexity of calling the model and provides a simple API to interact with it.

In our case, the application is a simple chatbot.
It uses a _WebSocket_, this is why you can also see the following dependency in the `pom.xml` file:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-websockets-next</artifactId>
</dependency>
```

If you now open the `src/main/java/dev/langchain4j/quarkus/workshop/CustomerSupportAgentWebSocket.java`  file, you can see how the web socket is implemented:

```java
package dev.langchain4j.quarkus.workshop;

import io.quarkus.websockets.next.OnOpen;
import io.quarkus.websockets.next.OnTextMessage;
import io.quarkus.websockets.next.WebSocket;

@WebSocket(path = "/customer-support-agent")
public class CustomerSupportAgentWebSocket {

    private final CustomerSupportAgent customerSupportAgent;

    public CustomerSupportAgentWebSocket(CustomerSupportAgent customerSupportAgent) {
        this.customerSupportAgent = customerSupportAgent;
    }

    @OnOpen
    public String onOpen() {
        return "Welcome to Miles of Smiles! How can I help you today?";
    }

    @OnTextMessage
    public String onTextMessage(String message) {
        return customerSupportAgent.chat(message);
    }
}
```

Basically, it:

1. Welcomes the user when the connection is opened
2. Calls the `chat` method of the `CustomerSupportAgent` class when a message is received and sends the result back to the user (via the web socket).

Let's now look at the cornerstone of the application, the `CustomerSupportAgent` interface.

```java
package dev.langchain4j.quarkus.workshop;

import io.quarkiverse.langchain4j.RegisterAiService;
import jakarta.enterprise.context.SessionScoped;

@SessionScoped
@RegisterAiService
public interface CustomerSupportAgent {

    String chat(String userMessage);
}
```

This interface is annotated with `@RegisterAiService` to indicate that it is an AI service.
An _AI service_ is an object managed by the Quarkus LangChain4J extension.
It models the interaction with the AI model.
As you can see it's an interface.
It's not a concrete class.
You do not implement this interface.
Quarkus LangChain4J will provide an implementation for you in the background.
Thus, your application only interacts with the methods defined in the interface.

There is a single method in the interface, `chat`.
It takes a user message as input (as it's the only parameter, we consider it to be the user message) and returns the response from the AI model.
How this is done is abstracted away by Quarkus LangChain4J.

!!! note "`SessionScoped`?"
    Attentive reader might have noticed the `@SessionScoped` annotation.
    This is a [CDI](https://jakarta.ee/specifications/cdi/) annotation which scopes the object to the session. In our case the session is the web socket.
    The session starts when the user connects to the web socket and ends when the user disconnects.
    This annotation indicates that the `CustomerSupportAgent` object is created when the session starts and destroyed when the session ends.
    It influences the _memory_ of our chatbot, as it remembers the conversation that happened so far in this session.

So far, so good! Let's move on to the [next step](./step-02.md).
