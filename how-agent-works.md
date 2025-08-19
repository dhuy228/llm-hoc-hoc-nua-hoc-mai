# Attemp to understand the back-end flow when calling Agent

### `run` — Workflow Execution

Runs a workflow starting at the given agent.  
The agent executes in a loop until a **final output** is generated.

---

#### Loop Execution Flow
1. **Invoke agent** with the given input.  
2. **Check for final output**:  
   - If the agent produces something of type `agent.output_type`, the loop terminates.  
3. **Check for handoff**:  
   - If there’s a handoff, the loop continues with the new agent.  
4. **Else, run tool calls** (if any), then re-run the loop.  

---

#### Exceptions
The agent may raise an exception in two cases:  
1. **`MaxTurnsExceeded`** → Raised if `max_turns` is exceeded.  
   - A turn = one AI invocation (including any tool calls).  
2. **`GuardrailTripwireTriggered`** → Raised if a guardrail tripwire is triggered.  
   - Note: Only the **first agent’s input guardrails** are run.  

---

#### Arguments
- **`starting_agent`** → The starting agent to run.  
- **`input`** → The initial input to the agent.  
  - Can be a single string (user message) or a list of input items.  
- **`context`** → The context to run the agent with.  
- **`max_turns`** → Maximum number of turns to run the agent.  
- **`hooks`** → Object that receives callbacks on lifecycle events.  
- **`run_config`** → Global settings for the entire agent run.  
- **`previous_response_id`** → ID of the previous response (if using OpenAI models via the Responses API).  
  - Allows skipping re-passing input from the previous turn.  

---

#### Returns
A **run result** containing:  
- All inputs  
- Guardrail results  
- Output of the last agent  

> Since agents may perform handoffs, the **exact output type is not guaranteed**.  

---

Do you want me to keep this in **docstring style** (triple quotes, ready for Python code), or more like a **Markdown reference doc** (like above)?  

---

## 5. User
how does it know when a final output is generated?  

---

## 6. Assistant
Good question 👌  

It knows a **final output is generated** when the agent produces something that matches the agent’s declared **`output_type`**.  

Here’s the flow broken down:

- Each agent has an **`output_type`** (e.g. a structured schema, JSON, or some other expected type).  
- During the loop, after the agent is invoked, the system checks:  
  - **Does the agent’s response conform to `agent.output_type`?**  
    - ✅ If yes → This is considered the **final output**, and the loop **terminates**.  
    - ❌ If no → The loop continues (either with a handoff to another agent, or by executing tool calls).  

So, the **contract** between the agent and the workflow runner is:  
> *“When I emit something of my `output_type`, that’s my final answer.”*  

Would you like me to show you a **concrete Python example** where an agent produces its final output (vs a tool call or handoff), so you can see how that check happens in practice?  

---

## 7. User
but by the default, it returns str so how does it know when to end the loop?  

---

## 8. Assistant
Exactly — by default, most agents are typed as producing a `str` for their `output_type`.  

So how does the runner know *this* string is the “final output” instead of just an intermediate step like a tool call?  

Here’s what happens under the hood:  

1. **The runner inspects the agent’s response.**  
   - If the response is a plain string (and the agent’s `output_type` is `str`), that **is considered a final output**.  
   - The loop stops immediately.  

2. **If the response is not a string**, the runner checks:  
   - Is it a *handoff* (telling the system to pass control to another agent)?  
   - Is it a *tool call* request?  
   - If yes, the loop continues with those steps.  

So in the “default `str` output_type” case:  
- **Any normal string output means “I’m done”**.  
- The only way the loop continues is if the agent explicitly returns a handoff or tool call instead of a string.  

<img width="720" height="519" alt="image" src="https://github.com/user-attachments/assets/a6812748-4543-4d87-9ed5-e4785853ec64" />

