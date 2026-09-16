Here is a comprehensive breakdown of the core mental model, underlying mechanisms, and
  engineering lessons behind each lab in the curriculum.
  ──────
   🧠 Lab 1: The Raw Model — Statelessness & Token Economics 

  ### The Core Notion

  │ "An LLM has no memory. Chat history is an illusion maintained entirely by the client."

  ### 1. The Anatomy of a Prompt
  Every LLM call consists of a list of messages with specific roles:

  • system: The invariant rules of engagement, persona, constraints, and instructions (e.g.,
  "You are a concise campus assistant. Two sentences max.").
  • user: The specific query or instruction sent by a human.
  • assistant: The generated completion produced by the model.
  • tool: (Introduced in Lab 2) The structured output produced by Python functions.

  ### 2. The Statelessness Reality

  When you ask ChatGPT or an API a follow-up question like:
  │ "What did I just ask you?"

  The server does not look inside its memory, because it has none. It is a pure
  mathematical function:

    Output = f(Input Transcript)

  If you do not append the previous questions and responses into the array:

    # The model sees only this:
    [{"role": "user", "content": "How many did I just ask about?"}]

  It will truthfully answer: "I don't know what you are referring to."

  To simulate memory, your Python script must accumulate the transcript and resend the
  whole conversation:
    messages = [
        {"role": "system", "content": "..."},
        {"role": "user", "content": "What is a backlog?"},
        {"role": "assistant", "content": "A backlog is..."},
        {"role": "user", "content": "Summarise that in one line."}
    ]

  ### 3. The Economic Consequence: Quadratic Token Growth

  • Turn 1: 19 tokens
  • Turn 2: 78 tokens
  • Turn 10: 800 tokens
  • Turn 50: 8,000 tokens

  Because you re-send every prior turn, conversations grow increasingly expensive with
  every message. This is why production systems require context window trimming and memory
  summarization (taught in Lab 5).
  ──────
   🛠️ Lab 2: Tools — Schema Generation & The Dispatch Pattern 

  ### The Core Notion
  │ "The model never executes your code. It outputs structured text requesting that YOU
  │ execute your code."

  ### 1. How the Model Knows What Tools Exist

  You don't write complex OpenAPI JSON files by hand. The @tool decorator uses Python's
  introspection:
  1. Function name → Tool identifier (get_student)
  2. Function docstring → The description the model reads to decide when and why to invoke
  it.
  3. Type hints (roll_number: str) → Parameter validation schema ("type": "string").
  If your tool description is vague, the model will either fail to call it or call it with
  hallucinations. The docstring is the API contract with the AI.

  ### 2. The String Trap

  When the model wants to call a tool, its output contains:
    {
      "name": "get_student",
      "arguments": "{\"roll_number\": \"21CS045\"}"
    }
  Notice that arguments is a serialized JSON string, not a Python dictionary. Forgetting
  to run json.loads(arguments) is the most common beginner bug in agent development.

  ### 3. The 4 Dispatch Scenarios

  When invoking a tool, 4 outcomes are possible:

  1. ok: Tool ran cleanly and produced data.
  2. KeyError / Exception: The tool executed, but the data was missing or invalid.
  3. InvalidArguments: The LLM passed parameters that don't match the function signature.
  4. UnknownTool: The LLM invented a function name that doesn't exist.

  A production agent framework must catch all 4 scenarios and convert them into readable
  dictionaries instead of letting Python crash.
  ──────
   🔄 Lab 3: The ReAct Agent Loop 
  ### The Core Notion

  │ "An agent is just a while loop that calls the model, executes requested tools, appends
  │ the outputs, and repeats until the model chooses to stop."

  ### 1. The Execution Lifecycle
    [User Goal]
         │
         ▼
    ┌──────────────┐
    │  Call Model  │ ◄────────────────────────┐
    └──────┬───────┘                          │
           │                                  │
           ├──── Has tool_calls? ──► YES ────►│
           │                                  │ Run Python tools
           ▼ NO                               │ Append tool outputs
    [Final Response]                          │ to message list
                                              └─────────────────┘

  ### 2. Key Insights

  1. Dynamic Step Count: You never hardcode how many steps an agent will take. For simple
  queries, it stops in 1 step. For multi-part queries, it calls tool A, gets the result,
  calls tool B, and then synthesizes the answer.
  2. Model vs. Tool Division of Labor:
      • Let tools handle ground truth data retrieval (e.g. database lookups, APIs,
      calculations).
      • Let the model handle reasoning, comparison, and language synthesis. Never write a
      tool for comparing whether 68 < 75—the model is already an expert at logic!

  ──────
   🩹 Lab 4: Self-Healing Agents 

  ### The Core Notion

  │ "Exceptions should not kill your program. In an agentic architecture, an exception is
  │ just an observation for the model to reflect on."

  ### 1. The Naive Crash vs. Self-Healing

  • Naive Architecture:
  User enters "twenty one CS zero four five".
  Python runs students["twenty one CS zero four five"] → raises unhandled KeyError →
  application terminates → user sees a 500 error.
  • Self-Healing Architecture:
  Python catches the KeyError and appends an observation to the transcript:
    {
      "error": "KeyError",
      "message": "No student with roll number 'twenty one CS zero four five'. Known roll
  numbers: ['21CS045', '21IT012', '22EC101']"
    }
  The agent loop continues. The model reads its own error transcript, figures out that
  "twenty one CS zero four five" corresponds to "21CS045", and immediately re-executes the
  tool with the correct argument.

  ### 2. The Three Cardinal Rules of Self-Healing

  1. Constructive Errors: A bare error (KeyError: 'foo') gives the model no clue how to
  fix it. The error must provide actionable clues (e.g., listing valid candidates).
  2. Traceback Truncation: A full Python stack trace is 40+ lines of internal framework
  noise. Truncate it to ~800 characters so you don't overwhelm the model's context window.
  3. The Circuit Breaker: If a tool fails 3 times consecutively, stop retrying. Without a
  circuit breaker, a self-healing agent can loop forever and burn your daily API limits in
  seconds.
  ──────
   💾 Lab 5: Memory & Session Architecture 

  ### The Core Notion

  │ "Agents must remain stateless singletons. Memory belongs to threads."

  ### 1. Thread Isolation

  If one agent instance stored memory internally in self.history, then Alice's financial
  questions would leak into Bob's session.

  • The Pattern:
      • The Agent class has no state.
      • A ConversationStore maintains dictionaries of messages keyed by thread_id.
      • On every interaction:
          1. Load messages = store.load(thread_id)
          2. Run result = agent.run(user_prompt, history=messages)
          3. Save store.save(thread_id, result.messages)



  ### 2. Pair-Safe Trimming

  When a conversation exceeds the token budget (e.g. keep_last=10), you cannot simply
  slice messages[-10:].

  • The Danger: If you slice right between an assistant tool-call message and the
  corresponding tool result message, the API will fail with:
  │ Invalid message: tool message must follow an assistant message with tool_calls

  • Pair-Safe Rule: Trimming must always verify that orphaned tool messages are removed
  along with their matching tool-call trigger.
  ──────
   🏗️ Lab 6: Multi-Agent Architectures 

  ### The Core Notion

  │ "Complex problems cannot be solved by one prompt. They require specialized roles and
  │ feedback loops."

  ### 1. Pattern 1: Evaluator–Optimizer (The Critique Loop)
  Instead of accepting the model's first raw draft:
    [User Request] ──► [Generator Agent] ──► Draft 1
                                               │
                                               ▼
                                       [Evaluator Agent]
                                       Checks explicit rubric
                                               │
                             ┌─────────────────┴─────────────────┐
                             ▼                                   ▼
                          PASSED                              FAILED
                             │                                   │
                             ▼                                   ▼
                       Final Result                   Feedback + Revision Loop

  • Critical Takeaway: The rubric must be measurable and concrete (e.g. "Under 90 words",
  "No exclamation marks"). If you give a vague prompt like "Make it sound good", the
  evaluator will rubber-stamp every draft on the first try.
  ### 2. Pattern 2: Orchestrator–Worker (Dynamic Delegation)
  Unlike basic parallelization where a human programmer hardcodes what jobs to run:

  1. Orchestrator: Reads the overarching goal and breaks it down into distinct sub-tasks,
  inventing the specialist worker personas on the fly.
  2. Workers: Independent mini-agents configured with their assigned sub-task and specific
  persona constraints.
  3. Synthesizer: Takes all worker outputs and compiles them into a unified report.
  ──────
  ### Summary Checklist of Day 1 Principles

    1. LLM Calls         ──► Stateless functions; transcripts must be resent.
    2. Tools             ──► Functions described by docstrings; args arrive as JSON
  strings.
    3. Agent Loops       ──► While-loop checking for tool_calls until text is generated.
    4. Error Handling    ──► Feed exceptions back into the model to trigger self-healing.
    5. Memory Stores     ──► Keep agents stateless; store histories by thread with pair-
  safe trimming.
    6. Multi-Agent       ──► Chain specialized agents for generation, critique, and
  synthesis.

  The background run for Lab 5 has also completed with an exit code of 0.
  Notice in that run how cleanly our backoff configuration (MAX_RETRIES=5,
  RETRY_BASE_DELAY=5.0) handled Gemini's per-minute free-tier burst limit:

    [rate limited - waiting 5s, attempt 2/5]
    [rate limited - waiting 10s, attempt 3/5]
    [rate limited - waiting 20s, attempt 4/5]
    < Could you please provide the student's name or roll number? I'll look up her
  attendance record for you.

  Instead of crashing when the burst quota was momentarily exhausted, it gracefully
  paused, backed off, retried, and finished the multi-turn session seamlessly.
  ──────
  ### Ready for the Hands-on Exercises?

  The labs were "read-and-run" to teach the concepts. The  folder contains the "write-
  your-own" challenges:

  1. **ex1_side_effect_tool.py**: Write a tool with a side-effect (e.g. sending notices)
  and engineer its docstring so it only fires when explicitly appropriate.
  2. **ex2_session_memory.py**: Implement your own multi-turn chat(thread_id, user_msg)
  session store.
  3. **ex3_circuit_breaker.py**: Build a failing tool to prove your circuit breaker stops
  infinite recovery loops.
  4. **ex4_evaluator_own_task.py**: Design your own Evaluator–Optimizer task with custom
  rubrics.

  Whenever you're ready, let me know which exercise you'd like to work on first!

▸ Thought for 1s, 305 tokens
  The encountered error, code 503, indicates high demand or unavailability of a de...
  An interesting real-world edge case just occurred in the background task for Lab 6:

  During the Orchestrator-Worker execution (which fires multiple parallel sub-agents),
  Google's endpoint briefly returned a 503 Service Unavailable / High Demand error:

    openai.InternalServerError: Error code: 503 - 
    'This model is currently experiencing high demand. Spikes in demand are usually
  temporary. Please try again later.'

  ### Key Engineering Takeaway

  Notice how agent.py:97 specifically classifies 503 as a transient error:

    transient = (
        "429" in message
        or "rate limit" in message
        or "resource_exhausted" in message
        or "overloaded" in message
        or "503" in message
    )

  Because the system retried with exponential backoff, it gave the provider 4 attempts
  across ~35 seconds before finally escalating when the surge persisted. In production
  systems with paid SLAs or multiple fallbacks (e.g. falling back to Groq or local
  Ollama), this is the exact mechanism that prevents total pipeline outages.