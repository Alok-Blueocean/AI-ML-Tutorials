# Agentic Systems and Tool Calling

## What This Topic Is

So far in this course, an LLM has mostly been a text-in, text-out box: you send a prompt, it sends back words. **Agentic systems** turn the LLM into something that can *act* — look things up, run code, call an API, update a database, or hand off work to another AI — and then use the results to decide what to do next.

An "agent," in this practical sense, is just an LLM wired into a loop: it reads the current situation, decides on an action (often by calling a **tool**), observes the result, and repeats until the task is done. **Tool calling** (also called function calling) is the mechanism that makes this possible — it's how the model asks your code to do something on its behalf, in a structured way it can reliably produce and your program can reliably parse.

## Why It Matters

Plain LLMs are frozen in time and blind to your systems — they only know what was in their training data plus whatever you paste into the prompt. Real MLOps/LLMOps work usually needs the model to:

- Pull live data (a stock price, a customer record, today's date)
- Take real actions (send an email, create a ticket, query a database, deploy something)
- Do multi-step work that a single prompt can't reliably handle in one shot

Agentic design is what turns an LLM from "a smart autocomplete" into "a system that gets work done." It's also where a lot of production risk lives — an agent that can call tools can also call the *wrong* tool, loop forever, or take an unintended action — so understanding the mechanics is essential before you ship one.

## Main Concepts in Plain Terms

**Tool / function calling**
You describe available tools to the model (name, description, and input parameters — usually as a JSON schema). Instead of just replying with text, the model can reply with "call `get_weather` with `{city: 'Paris'}`." Your code executes that function, gets a real result, and feeds it back to the model as part of the conversation. The model never actually runs the code — it only decides *what* to call and *with what arguments*; your application is responsible for actually executing it safely.

**The agent loop**
A typical loop looks like: *Think → Act → Observe → Repeat*. The model reasons about what to do next, calls a tool, gets the observation back, and decides whether it's done or needs another step. This pattern is often nicknamed **ReAct** (Reason + Act). The loop needs a stopping condition (task complete, max steps reached, or a failure) so it doesn't run forever.

**Planning**
For anything beyond a couple of steps, agents benefit from an explicit plan: break the goal into subtasks before diving in, rather than deciding everything one step at a time. Some frameworks make this a separate "planner" step; simpler agents fold planning into the same reasoning loop.

**Memory**
LLMs are stateless between calls, so agents need memory bolted on:
- *Short-term / working memory* — the running conversation or scratchpad for the current task.
- *Long-term memory* — facts, past interactions, or documents stored externally (often in a vector database) and retrieved when relevant, similar to the RAG pattern covered elsewhere in this course.

**Multi-agent systems**
Instead of one agent doing everything, you can split responsibilities across several specialized agents (e.g., a "researcher," a "coder," and a "reviewer") that pass work between each other, coordinated by a manager agent or a fixed workflow. This mirrors how teams divide labor — each agent has a narrower job, simpler prompt, and clearer tools, which is often easier to test and debug than one giant do-everything agent.

**MCP (Model Context Protocol)**
MCP is an open standard for connecting LLM applications to external tools and data sources through a common protocol, instead of every app writing custom, one-off integration code for every tool. Think of it like a USB port for AI tools: an **MCP server** exposes a set of tools/resources (e.g., "search Jira," "read this database"), and any MCP-compatible client (an agent framework, an IDE assistant, a chat app) can plug into it without bespoke glue code. This matters operationally because it decouples "which tools exist" from "which agent framework you happen to be using."

**Orchestration frameworks (LangGraph, CrewAI, etc.)**
Hand-rolling an agent loop is fine for a demo, but production systems usually reach for a framework to manage the plumbing:
- **LangGraph** models an agent (or multi-agent system) as a graph of nodes and edges — good when you want explicit control over branching, retries, and state, closer to a state machine than a free-form loop.
- **CrewAI** frames things around "crews" of role-based agents (each with a role, goal, and tools) collaborating on a shared task — good for quickly expressing a multi-agent workflow in a role-based way.

Neither is "the" right answer; both sit on top of the same underlying ideas above (tools, loops, memory, planning).

## A Simple Example

Here's the shape of a tool definition and a call, using a generic function-calling style (this mirrors how most LLM APIs express it):

```python
# 1. Define a tool the model is allowed to call
tools = [
    {
        "name": "get_order_status",
        "description": "Look up the current status of a customer order by ID.",
        "parameters": {
            "type": "object",
            "properties": {
                "order_id": {"type": "string", "description": "The order ID, e.g. ORD-1234"}
            },
            "required": ["order_id"]
        }
    }
]

# 2. Model receives a user message + the tool list, and may respond with a tool call:
# model_response.tool_call = {"name": "get_order_status", "arguments": {"order_id": "ORD-1234"}}

# 3. Your code executes the real function
def get_order_status(order_id: str) -> str:
    return db.lookup(order_id)  # your actual business logic

result = get_order_status(**model_response.tool_call["arguments"])

# 4. Feed the result back to the model so it can produce a final natural-language answer
# "Your order ORD-1234 has shipped and should arrive Friday."
```

The key idea: the model only ever produces *structured intent* (name + arguments). Your application code stays in control of what actually executes — which is exactly where you'd add validation, permissions, and logging in a real system.

## Key Takeaways / Best Practices

- **Tool calling = structured intent, not execution.** The model proposes a call; your code decides whether and how to run it. Always validate arguments before executing.
- **Keep tools narrow and well-described.** Clear names, descriptions, and schemas reduce wrong or hallucinated calls — this is prompt engineering applied to tools.
- **Always bound the agent loop.** Set a max number of steps/iterations and clear success/failure conditions so an agent can't loop indefinitely or spiral into repeated bad actions.
- **Treat tool access as a security boundary.** Anything an agent can call, it might call incorrectly or be tricked into calling (e.g., via malicious content it reads) — apply least-privilege permissions, human approval for risky actions, and audit logs.
- **Separate short-term and long-term memory deliberately.** Don't dump everything into the prompt; retrieve only what's relevant for the current step.
- **Reach for multi-agent designs only when complexity justifies it.** A single well-scoped agent is easier to debug than a crew of five; split responsibilities when one agent's job genuinely becomes too broad.
- **Prefer standards where they exist.** MCP-style integration reduces custom glue code and makes tools reusable across agents and frameworks.
- **Frameworks (LangGraph, CrewAI) manage plumbing, not judgment.** They help with state, retries, and coordination — but you still need to design the tools, prompts, and guardrails yourself.
