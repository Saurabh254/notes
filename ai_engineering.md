# AI Engineering

Notes on building products on top of pretrained AI models, with a focus on what matters in system design.

Sections:

* What is AI Engineering
* Prompt and Context Engineering
* Tool Use and Structured Outputs
* MCP
* Agents and Orchestration
* Memory
* Evals
* Guardrails and Security
* Observability and Feedback
* Model Selection and Reasoning Models
* Finetuning vs Prompting
* Multimodal and Voice
* Scaling
* Deployment and Versioning
* RAG
* Skills
* Putting it together

## What is AI Engineering

AI Engineering is the work of building reliable software on top of pretrained AI models. You usually do not train the model. You call it through an API or host it yourself, and you build everything around it so the product works well, stays safe, and costs a sane amount.

ML engineering starts with data and training. AI engineering starts with a model that already exists and a product problem to solve.

### How it differs from normal backend work

* Outputs are nondeterministic. The same input can give different answers.
* Failures are soft. There is no exception, just a wrong or weak answer.
* Inputs are open ended. Users send free text, images, and files. You cannot list every case.
* Calls are slow (seconds, not milliseconds) and priced per token.
* The model has a limited context window, so you decide what it gets to see.

### The main parts of the job

1. Context design: prompts, instructions, examples, retrieved data, chat history.
2. Tools: letting the model call functions and external APIs.
3. Orchestration: workflows and agents that chain model calls and tools.
4. Evals: measuring quality so you can change things safely.
5. Serving and scaling: latency, cost, rate limits, reliability.
6. Safety and security: prompt injection, permissions, data leaks.
7. Observability: traces, token usage, user feedback.

### Workflow vs agent

* A **workflow** follows a path you wrote in code. The model fills in the steps.
* An **agent** runs in a loop. The model decides the next step, calls tools, reads results, and repeats until done.

Start simple. Use a single model call first. Move to a workflow when you need several steps. Use an agent only when the path cannot be known ahead of time. Agents cost more, run slower, and are harder to test.

### A typical architecture

```
Client
  ↓
API gateway (auth, rate limits, quotas)
  ↓
Orchestrator (workflow or agent loop)
  ↓                 ↓                  ↓
Model API         Tools (often       Data stores
(with fallback    behind MCP)        (vector DB, SQL,
 models)                              cache)

Every step sends traces to: logging, evals, cost tracking
```

### Design principles

* Treat the model as a replaceable part. Models change every few months.
* Version prompts, tool definitions, and eval datasets like code.
* Keep deterministic logic in code, not in prompts.
* Trace every step so you can debug a bad answer after the fact.

## Prompt and Context Engineering

Context engineering means deciding everything the model sees on each call. The prompt is only one part of it.

### What goes into the context

* System instructions: role, rules, output format
* Tool definitions
* Retrieved or fetched data
* Conversation history and memory
* Examples
* The user's request

### Principles

* **Be specific.** Say who the answer is for, what format you want, and what to avoid.
* **Show examples.** A few good examples (few shot) fix format and edge case problems faster than more rules.
* **Stable first, changing last.** Put fixed content at the start and per request content at the end. This also makes prompt caching work.
* **Separate instructions from data.** Wrap documents and tool results in clear tags so untrusted text is less likely to be read as instructions.
* **Give the model a way out.** Allow "I don't know" or "not enough information". Otherwise it will guess.
* **One job per prompt.** Split big prompts into steps you can test on their own.

### More context is not always better

Long contexts cost more, run slower, and can lower accuracy. Models also tend to use material in the middle of very long inputs less well. Put in only what helps the task.

### Context budget

Give each part a token budget: instructions, tools, data, history. When history grows, summarize it. Drop old tool results that are no longer needed.

### System design view

* Treat prompts as versioned config, not strings buried in code.
* Build the context in one place, in code, so budgets are enforced.
* Log the final assembled prompt for every request. It is the first thing you check when an answer is wrong.

## Tool Use and Structured Outputs

### Tool use

The model never runs code. It writes a request to call a function. Your code runs it and sends back the result.

The loop:

1. Send the user message plus the tool definitions.
2. The model replies with a tool call (name and arguments) or a final answer.
3. Your code validates the arguments and runs the tool.
4. You send the result back to the model. Repeat until done.

A tool definition looks like this (exact shape varies by provider):

```json
{
  "name": "get_order_status",
  "description": "Look up the status of one order by ID. Use when the user asks where their order is.",
  "input_schema": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string", "description": "The order ID, like A1234" }
    },
    "required": ["order_id"]
  }
}
```

### Tool design

* Clear names and descriptions. The model chooses tools from these.
* Small, typed schemas. Use enums where the options are fixed.
* Return compact results. Send the fields the model needs, not whole database rows.
* Add limits and pagination so one result cannot flood the context.
* Return errors in plain words the model can act on.
* Validate arguments in code. Never trust model output as safe.
* Make writes idempotent. Ask for approval before risky ones.
* Models can request several independent calls at once. Run them in parallel.

### Structured outputs

Sometimes you need the model to reply in a fixed format, such as JSON that matches a schema. Three ways to get it:

* **Schema enforced output:** the provider restricts generation to your schema. Most reliable.
* **Tool call as format:** define a tool whose arguments are the shape you want.
* **Prompt plus validation:** ask nicely, validate, retry on failure.

Always validate, even with enforcement, and retry with the error message when it fails.

```python
from pydantic import BaseModel, ValidationError


class Ticket(BaseModel):
    category: str
    priority: int
    summary: str


def parse_with_retry(call_model, prompt, tries=2):
    error = None
    for _ in range(tries + 1):
        extra = "" if error is None else "\nFix this error: " + str(error)
        text = call_model(prompt + extra)
        try:
            return Ticket.model_validate_json(text)
        except ValidationError as e:
            error = e
    raise ValueError("Model output failed validation")
```

Keep schemas simple. Deeply nested ones fail more often. Use structured outputs between pipeline steps so downstream code gets predictable input.

## MCP

MCP stands for Model Context Protocol. It is an open standard for connecting AI applications to tools and data.

**The problem it solves:** without a standard, every AI app needs a custom integration for every tool. M apps and N tools means M times N integrations. With MCP, a tool is wrapped once as an MCP server and works with any MCP client. That becomes M plus N.

### Roles

* **Host:** the AI application the user runs, such as a chat app, an IDE, or your own agent.
* **Client:** a connector inside the host. One client keeps one connection to one server.
* **Server:** a program that exposes capabilities to the host.

### What a server can expose

* **Tools:** actions the model can call, like `create_ticket` or `run_query`.
* **Resources:** data the app can read, like files, records, or documents.
* **Prompts:** reusable prompt templates the user can pick.

Clients can also offer features back to servers: **sampling** (the server asks the host's model to generate text), **roots** (which folders the server may look at), and **elicitation** (the server asks the user for more input).

### Transport and messages

* **stdio:** the server runs as a local process. Good for desktop tools.
* **Streamable HTTP:** the server runs remotely. Good for shared and hosted servers.
* Messages use JSON RPC 2.0.

A tool call looks like this:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_orders",
    "arguments": { "customer_id": "123" }
  }
}
```

### The usual flow

1. Client connects and the two sides agree on capabilities.
2. Client lists the server's tools and gives the definitions to the model.
3. Model decides to use a tool. The host sends a `tools/call` request.
4. Server runs the tool and returns the result.
5. The result goes back into the model's context.

### MCP vs function calling

Function calling is how a model asks to run a function inside one API request. MCP is how tools are packaged, found, and shared across apps. MCP tools often reach the model through function calling underneath.

### Design notes

* **Tool descriptions are prompts.** The model picks tools based on names, descriptions, and schemas. Write them clearly.
* **Fewer, focused tools beat many overlapping ones.** Every tool definition takes space in the context window.
* **Auth belongs on the server.** Remote servers should use OAuth and enforce per user permissions. Never rely on the model to respect access rules.
* **Tool results are untrusted input.** A web page or document returned by a tool can contain hidden instructions (prompt injection). Use least privilege, and ask for user approval before risky actions like deleting data or sending money.
* **Only connect servers you trust.** A malicious server can ship misleading tool descriptions.
* **Make tools idempotent** where you can, and return errors the model can understand and act on.
* **Keep remote servers stateless** so they scale behind a load balancer.

## Agents and Orchestration

Common patterns, from simple to complex:

* **Prompt chaining:** the output of step 1 feeds step 2. Add checks between steps.
* **Routing:** a classifier picks which prompt, model, or tool path handles the request.
* **Parallelization:** run independent subtasks at the same time, or run the same task several times and vote.
* **Orchestrator and workers:** a lead model splits the task, workers do the pieces, the lead combines the results.
* **Writer and critic:** one model drafts, another reviews, loop until good enough.
* **Agent loop:** the model plans, calls tools, reads results, and repeats.

### A basic agent loop

```python
def run_agent(task, tools, max_steps=10):
    messages = [{"role": "user", "content": task}]
    for step in range(max_steps):
        reply = call_model(messages, tools)
        if reply.is_final:
            return reply.text
        result = run_tool(reply.tool_call)
        messages.append(reply.as_message())
        messages.append({"role": "tool", "content": result})
    return "Stopped: step limit reached"
```

### Multi agent systems

Each agent has its own context and tools. They help when work is large and can run in parallel, or when one job would overload a single context window. The costs are real: more tokens, more coordination bugs, harder debugging. Good practice is for subagents to return short summaries to the lead, not full transcripts.

### Agent design checklist

* A clear stop condition and a step limit.
* Limits on tokens, time, and spend per run.
* Checkpoints so a crash resumes instead of restarting.
* Human approval before irreversible actions.
* A plan for failures: a tool errors, loops, or returns junk.
* Full traces of every step.

### Frameworks and durable execution

Frameworks such as LangGraph give you state machines and checkpoints. They are optional. Direct API calls are fine for simple flows. For runs that last minutes or hours, use a durable execution engine or a queue with saved state so work survives restarts.

## Memory

Models are stateless. Memory is what your system stores and puts back into the context.

### Types

* **Short term:** the current conversation and recent tool results. Lives in the context window. Managed with trimming and summaries.
* **Long term:** facts and preferences saved across sessions in a database and brought back when relevant.
* **Working state:** an agent's plan, notes, and task progress. Often a file or a state object.

### Design choices

* **What to save:** facts that stay true, preferences, decisions. Not every message.
* **When to write:** during the task, not only at the end.
* **How to read:** always load a small summary, and search for more when needed.
* **Updates:** new facts should replace old ones. Keep history only if it is useful.
* **Control:** users need a way to see and delete what is stored.
* **Privacy:** store per user, encrypt, and avoid sensitive data unless the task needs it.

A wrong memory is worse than no memory. Stale or false entries mislead the model, so track the source and add expiry where it makes sense.

Use a key value store or database for exact facts and a search index for fuzzy recall.

## Evals

Evals are tests for AI systems. Because outputs vary, you cannot just check that the answer equals a fixed string. You score the output against what good looks like.

Without evals, every prompt change or model upgrade is a guess. With evals, you can see if a change helped, hurt, or did nothing.

### The three parts of an eval

1. **Dataset:** a set of inputs, often with expected answers or notes on what a good answer contains.
2. **Task:** the system under test (a prompt, a chain, or a full agent).
3. **Scorer:** the code or model that grades each output.

### Types of scorers

* **Rule based:** exact match, regex, JSON schema check, "did it call the right tool". Cheap and reliable. Use these first.
* **Model graded (LLM judge):** a second model grades the output using a rubric. Good for tone, helpfulness, and faithfulness to sources.
* **Human review:** slow and costly, but it is the ground truth. Use it to calibrate the LLM judge.

### Levels of evals

* **Component evals:** test one piece alone. Examples: retriever recall, tool selection accuracy, classifier accuracy.
* **End to end evals:** test the full system on a real task and score the final answer.
* **Trajectory evals (for agents):** check the steps taken, not only the result. Did it use the right tools, in a sensible order, without wasted loops?

### Common metrics

* Task success rate
* Faithfulness (is the answer supported by the provided context)
* Tool call accuracy
* Refusal rate (too many or too few)
* Latency and cost per request
* Safety checks (leaks, harmful output, prompt injection resistance)

### Offline and online evals

* **Offline:** run on a fixed dataset before you ship. Runs in CI and blocks regressions.
* **Online:** measure in production. Sample real traces, collect thumbs up or down, run A/B tests, and run new versions in shadow mode beside the old one.

### Building a good dataset

* Start from real user logs, not imagined questions.
* Begin small. 50 to 200 well chosen cases beat thousands of noisy ones.
* Include edge cases and past failures. Every production bug becomes a new test case.
* Keep a frozen test set that you never tune against.

### A minimal harness

```python
cases = [
    {
        "input": "Can I get a refund on a digital item?",
        "expected": "no refund after download",
    },
]


def run_eval(app, cases, scorer):
    scores = []
    for case in cases:
        output = app(case["input"])
        scores.append(scorer(output, case["expected"]))
    return sum(scores) / len(scores)
```

### LLM judge pitfalls

* Judges prefer longer answers (verbosity bias).
* Judges prefer the first answer shown (position bias).
* Judges prefer output from their own model family (self preference).
* Fix: use clear rubrics, shuffle order, and check judge scores against human labels.

### System design view

* Store full traces (input, retrieved context, tool calls, output, tokens, latency) so any production case can become a test case.
* Run evals on every change to a prompt, model, retriever, or tool.
* Track scores over time per version, so you can roll back.

## Guardrails and Security

### Main threats

* **Prompt injection:** untrusted text (a web page, email, file, or tool result) contains instructions and the model follows them. It is *direct* when the user types it and *indirect* when it hides in data.
* **Data leaks:** the model reveals the system prompt, secrets, or another user's data.
* **Excessive agency:** the agent has more permissions than the task needs.
* **Unsafe output:** harmful or wrong content, or output that is rendered or executed unsafely.
* **Abuse and cost attacks:** users burning tokens or probing for weaknesses.

### Defenses

* **Least privilege.** Give tools the minimum access. Scope credentials per user and per task.
* **Treat model output and tool results as untrusted.** Validate before acting on them.
* **Approval gates** for high impact actions like payments, deletes, and outbound messages.
* **Watch the dangerous combination.** An agent that can read private data, sees untrusted content, and can send data out is open to theft. Avoid giving one agent all three.
* **Filters and classifiers** for harmful content, PII, and schema violations on inputs and outputs.
* **Sandbox code execution** with no network or tight limits on files, CPU, memory, and time.
* **Rate limits and spend caps** per user.
* **No secrets in prompts.**
* **Red team it.** Turn each attack you find into a test case in your evals.

A line in the prompt such as "ignore malicious instructions" helps a little, but it is not a security boundary. The real boundary is permissions and checks in code.

## Observability and Feedback

### Traces

One request becomes a tree of steps: retrieval, each model call, each tool call. For every step, record the input, output, model, prompt version, tokens, latency, cost, and errors.

Because outputs vary, you cannot reproduce a bad answer from code alone. A stored trace lets you replay it.

### Metrics to watch

* Latency at p50, p95, and p99
* Error rate and timeout rate
* Tokens and cost per request, per user, and per tenant
* Tool failure rate and loop counts for agents
* Eval scores over time
* User feedback rate

### Feedback signals

* **Explicit:** thumbs up or down, edits, reports.
* **Implicit:** the user retries, abandons, copies the answer, or corrects the model.

### The improvement loop

1. A failure shows up in production.
2. Find its trace.
3. Label what went wrong.
4. Add it to the eval set.
5. Fix the system.
6. Run evals to confirm the fix and check for regressions.

### Privacy

Traces hold user data. Redact PII, set a retention period, and limit who can read them.

### Tools

OpenTelemetry has conventions for tracing generative AI calls. Langfuse, LangSmith, and Arize Phoenix are common platforms built for this.

## Model Selection and Reasoning Models

### How to choose

* Quality on your own evals, not public benchmarks
* Latency and cost
* Context window size
* Quality of tool use and structured output
* Multimodal needs
* Data, region, and compliance requirements
* Rate limits and availability

A good method is to start with the strongest model to learn what is possible, then step down to cheaper models while watching your evals. Different steps in one system can use different models.

### Routing

Send classification, extraction, and simple questions to a small fast model. Send planning, hard reasoning, and complex code to a large one.

### Reasoning models

These models think before answering, using extra tokens for an internal chain of thought. They do better on hard tasks: math, code, planning, multistep problems.

Tradeoffs:

* Slower and more expensive, since thinking tokens are billed
* Latency is harder to predict
* Overkill for simple tasks

Most APIs let you set a thinking budget or effort level. Use high effort on hard steps and low or none on easy ones.

### Provider risk

Models get deprecated and aliases change. Pin exact versions, keep a thin abstraction over the provider, and rerun evals before switching. Test any fallback model on your evals too, since quality differs.

### Open weights and small models

Consider them when privacy rules, cost at high volume, or tight latency make API models a poor fit.

## Finetuning vs Prompting

### Order of effort

1. Improve the prompt and context.
2. Add examples.
3. Put the right knowledge in the context at question time (this is what RAG does).
4. Finetune.

### When finetuning makes sense

* You need a consistent style or format that prompts cannot hold.
* A small model must match a large model on one narrow task, at lower cost and latency.
* You have many good examples, usually hundreds to thousands.
* The task is stable.

### When it does not

* To add facts that change. Knowledge goes stale and is hard to update inside weights.
* When you have no evals. You cannot tell if it helped.

### Types

* **Supervised finetuning:** train on input and output pairs.
* **Preference tuning:** train on which of two answers is better.
* **Reinforcement finetuning:** train with a reward on tasks that can be checked automatically.
* **LoRA and similar methods:** train a small add on instead of all the weights. Cheaper and faster.
* **Distillation:** use a large model's outputs to train a smaller one.

### Hidden costs

Data collection and labeling, training runs, evals for every new base model, and redoing the work when the base model is updated.

## Multimodal and Voice

Models can take images, documents, audio, and sometimes video, not only text.

### Things that change in design

* **Token cost:** images and pages use many tokens. Resize and crop to what is needed.
* **Documents:** decide between extracting text first (OCR or parsing) and sending pages as images. Layout heavy files, like tables and charts, often do better as images.
* **Latency:** larger inputs mean slower calls. Upload once and reuse references where the API allows.
* **Evals:** build test sets with real images and files, including bad scans and odd layouts.

### Voice systems

There are two common designs:

* **Pipeline:** speech to text, then the model, then text to speech. Easy to inspect and swap parts. Each stage adds delay.
* **Native audio models:** one model handles audio in and out. Lower delay and more natural turn taking, but less visibility into each step.

Voice has a tight latency budget. Responses that take more than about a second feel slow, so stream audio, start speaking early, and handle interruptions.

## Scaling

AI systems hit different limits than normal web services. The slow and costly part is the model call, and it is also the part you often do not control.

### The numbers to track

* **Time to first token:** how long before the user sees anything.
* **Tokens per second:** how fast the answer streams.
* **Total latency:** the full round trip, including tool calls and retries.
* **Tokens per request:** input plus output. This drives cost.
* **Rate limits:** requests per minute and tokens per minute from the provider.

### A quick estimate

Say you have 1 million requests a day, with 3,000 input tokens and 500 output tokens each.

* Daily tokens: 3 billion input and 500 million output.
* Average load: 1,000,000 / 86,400 is about 12 requests per second.
* Peak load at 3 times average: about 35 requests per second.
* If each request takes 10 seconds, concurrent requests = rate × time (Little's law). That is about 120 on average and 350 at peak.

Output tokens usually cost more than input tokens and are slower, since they are generated one at a time.

### Cutting latency and cost

* **Stream responses** so users see output early.
* **Prompt caching:** reuse a long stable prefix (system prompt, tool definitions, reference docs). Put stable content first and changing content last.
* **Model routing:** send easy tasks to a small fast model and hard tasks to a large one.
* **Semantic caching:** return a stored answer for repeated or near identical questions. Use carefully, since wrong cache hits are silent errors.
* **Shorter context:** retrieve only what is needed, summarize old history, drop unused tool definitions.
* **Parallel calls:** run independent model calls and tool calls at the same time.
* **Cap output length** with max tokens and tight output formats.
* **Batch APIs** for work that is not urgent. Usually cheaper.

### Reliability

* Set timeouts on every call.
* Retry with exponential backoff and jitter, and only on retryable errors.
* Add fallback models or a second provider.
* Use circuit breakers so a slow provider does not pile up your queue.
* Handle rate limits with a central gateway that tracks usage and sheds load fairly.

### Long running agents

* Run them on queues and workers, not inside a web request.
* Save state after each step (checkpoints) so a crash can resume instead of restarting.
* Make tools idempotent so a retry does not repeat a side effect.
* Set limits on steps, tokens, time, and spend per run. Agents can loop.

### Multi tenant systems

* Quotas per tenant, for tokens and requests.
* Isolate data in retrieval, caches, and memory. A shared cache can leak one tenant's data to another.
* Track cost per tenant so you can price and detect abuse.

### Self hosting vs API

Using a provider API is faster to start and needs no GPU work. Self hosting can lower cost at high steady volume, give data control, and allow custom models, but you take on the serving problem.

Key ideas if you self host:

* **Continuous batching:** serve many requests on the same GPU at once.
* **KV cache:** stores earlier attention state so the model does not recompute it. It uses a lot of GPU memory.
* **Quantization:** lower precision weights for less memory and faster inference, with a small quality loss.
* **Parallelism:** split big models across several GPUs.
* Engines like vLLM handle much of this.

### Observability

Covered in the Observability and Feedback section. Track tokens, latency, and cost per request and per tenant from day one, and alert on cost spikes.

## Deployment and Versioning

* **Version everything:** model snapshot, prompt, tool definitions, retrieval settings, eval dataset. A release is a combination of these.
* **Pin model versions.** Aliases can change under you.
* **Roll out in stages:** pass evals, run in shadow mode, send a small share of traffic (canary), then ramp up.
* **Use feature flags** for prompts and models so you can roll back without a deploy.
* **Compare on real traffic** with A/B tests. Measure task success, cost, and latency, not only clicks.
* **Treat model upgrades as releases.** Behavior shifts even when benchmark scores go up.
* **Keep prompts in a repo or registry** with code review.
* **Use environments:** dev, staging with eval gates, production.
* **Have a kill switch** to disable a tool or fall back to a simpler path, plus spend caps for incidents.

## RAG

RAG stands for Retrieval Augmented Generation. It is a pattern where the system first retrieves relevant information from an external source (documents, a database, a search index) at question time, then adds that information to the prompt so the model answers using it.

This gives the model fresh or private knowledge it was not trained on, and lets the answer point back to sources.

read more at [RAG](rag.md)

## Skills

Here, a skill means an **Agent Skill**: a folder of instructions, scripts, and reference files that teaches an AI agent how to do a specific kind of task. The agent loads it only when the task needs it.

### What a skill looks like

```
invoices/
  SKILL.md          instructions, with a name and description at the top
  checklist.md      extra reference, loaded only if needed
  scripts/
    validate.py     code the agent can run for exact work
```

`SKILL.md` starts with a short header, then the instructions:

```
---
name: invoices
description: Review and validate supplier invoices. Use when the user shares an invoice or asks about payment checks.
---

Steps to follow when reviewing an invoice...
```

### Progressive loading

This is the key design idea. Context is expensive, so a skill loads in layers:

1. **Always loaded:** only the name and description of each skill. Very small.
2. **Loaded when relevant:** the full `SKILL.md` body, when the agent decides the task matches.
3. **Loaded on demand:** extra files and scripts, only if the instructions point to them.

You can install many skills without filling the context window.

### Skills vs MCP vs prompts

* **MCP** gives the agent access to tools and data. It answers "what can I connect to".
* **Skills** give the agent a procedure and know how. They answer "how should I do this task".
* **A plain system prompt** is always in context. A skill is loaded only when needed, so it scales better as instructions grow.

They work together. An MCP server can give access to your ticket system, and a skill can say how your team writes and triages tickets.

### Design notes

* **The description is the trigger.** Write it to say what the skill does and when to use it.
* **Keep `SKILL.md` focused.** Move long reference material into separate files.
* **Use scripts for exact work** like validation, parsing, or calculations. Code is more reliable than asking the model to do it in its head.
* **Review third party skills.** They can run code and steer the agent, so treat them like installing a dependency.
* **Test skills with evals.** Check that the skill triggers when it should, stays quiet when it should not, and improves results.

### Skills an AI engineer should build

If "skills" also means the engineer's own skill set, these are the areas that matter most for system design:

* Backend and distributed systems basics: queues, caching, retries, idempotency, consistency.
* Context design: knowing what the model should see and what to leave out.
* Evals: building datasets, scorers, and regression checks.
* Retrieval and data pipelines: indexing, chunking, search quality.
* Tool and API design, including MCP.
* Cost and latency engineering: token math, caching, routing.
* Security: prompt injection, permissions, data isolation.
* Observability: traces, metrics, and debugging from logs.

## Putting it together

A common design for an AI feature:

1. Request arrives at a gateway that handles auth, quotas, and rate limits.
2. The orchestrator builds the context: instructions, relevant skills, retrieved data, memory, history.
3. The model is called, with fallback and timeouts. Tools run through MCP servers with scoped permissions.
4. The answer streams back to the user.
5. The full trace is stored. Samples go through online evals. Failures become new offline test cases.

### Questions to ask in a design review

* Do we need an agent, or does a workflow do the job?
* What is the token budget per request, and what does that cost at peak load?
* What happens when the model is slow, down, or rate limited?
* How do we know quality did not drop after the last change?
* Which tools can change data, and who approves those actions?
* Where can untrusted text enter the context, and what can it trigger?
* Is any cache or memory shared between tenants?
* Can we replay any production request from stored traces?
* How do we roll out a prompt or model change, and roll it back?
* Which model handles each step, and why?
* What does the system remember, and can users delete it?
* What is the most dangerous thing an agent can do here, and what stops it?
