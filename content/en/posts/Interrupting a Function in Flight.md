+++
date = '2025-09-16T21:23:31+09:00'
draft = true
title = 'Interrupting a Function in Flight'
categories= ['Programming']
tags= ['', '']
+++

**How can we stop a function that is already running and keep it from proceeding?**

Picture an endpoint that accepts a user question, runs an LLM to determine intent, calls an embedding service to obtain vectors, hits both a full-text index and a vector database to fetch supporting snippets, sends the results to an external reranker, runs them through additional logic, and finally feeds everything back into an LLM to produce an answer.

That’s a typical RAG flow. A single endpoint hides a huge amount of work:

1. External API calls
2. Multiple database queries
3. Heavy computation

But what happens if the client cancels the request right after sending it?

> Cancellation includes:
>
> 1. The user clicks “Cancel question” in the UI.
> 2. The user closes the browser tab.

Historically my backend would “do nothing”—the request would keep running in the background. This time I wondered: could the server stop executing the rest of the pipeline as soon as the client cancels?

What if the language provided an execution unit that is isolated from the rest of the program, shares no memory, is cheap to spawn, and just as cheap to tear down?

That language is Elixir (or rather, Erlang).

Golang has goroutines. They seem to tick the same boxes. Do they?
