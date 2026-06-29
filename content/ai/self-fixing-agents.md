+++
title = "Self improving code using AI (agents)"
date = "2026-06-27"
aliases = ["ai"]
tags = ["ai", "dev"]
categories = ["ai", "software", "dev"]
author = "codecowboy.io"
showtoc = true
+++

## Intro

I've been messing around with multi-agent AI systems recently. I had a crazy idea - what if I could get one AI agent to write code, have another score it, and a third refine it based on that score? All automatically. All in a loop.

That's what I'm going to walk through here.

The things I wanted to explore were:
- Getting an AI agent to generate code from a prompt
- Getting a second AI agent to score that code and provide structured feedback
- Using that feedback to automatically refine the code in a loop
- Running the final accepted code as an actual subprocess

TLDR - if you just want the code it's here: [https://github.com/codecowboydotio/ai-self-propagate-experiment](https://github.com/codecowboydotio/ai-self-propagate-experiment)

## What is this?

I built a pipeline where Agent 1 generates a Python script, a scorer evaluates it, and a refiner improves it - round and round until the score is good enought. Once the code passes the threshold, Agent 1 writes it to a temp file and executes it as a child process.

There are a few configurable constants that control the loop:

```python
MAX_REFINEMENTS = 3
MIN_SCORE = 9.6
```

If the code scores 9.6 or above out of 10, it gets accepted. Otherwise we refine, up to three times. If it still hasn't hit the bar, the script exits with a non-zero code.

## Agent 1 - the generator

Agent 1 uses `claude-opus-4-8` with a tight system prompt that tells it to respond only with source code - no markdown, no commentary, no backticks.

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    system=(
        "You are a coding agent that responds only with source code. "
        "Do not include any commentary, markdown, or backticks. "
        "Respond only with valid, self-contained Python code."
    ),
    messages=[{"role": "user", "content": ORIGINAL_PROMPT}],
)
agent2_code = response.content[0].text
```

The task I gave it is simple - write a Python script that calls Claude and asks it "What is 2 + 2?". The point isn't the task, it's the pattern.

{{< notice info >}}
> **Info**
> The generated code becomes Agent 2 - not a model, but an actual Python script that gets executed as a subprocess later. Agent 1 literally creates Agent 2.
{{< /notice >}}

## Scoring the code

The scorer uses `claude-haiku-4-5-20251001` - faster and cheaper. It receives the original prompt, the code to evaluate, and the full history of previous attempts.

The history part is important. Without it, scores regress. The scorer forgets what it already rewarded and starts penalising things it previously accepted. I learned this the hard way - early runs would score something highly, then the next iteration would penalise the same thing again. Passing the full history fixes this.

The scorer returns a structured diff format:

```
Score: 8.5/10
Reason: The code is functional but missing error handling.
Diff:
- REMOVE: response = client.messages.create(...)
  ADD: try:\n    response = client.messages.create(...)\nexcept anthropic.APIError as e:\n    print(f"API error: {e}")\n    sys.exit(1) (+1.5)
```

Forcing exact `REMOVE/ADD` pairs rather than vague feedback makes the refiner's job much more deterministic. "Improve error handling" is useless. "Replace this exact line with this exact block" is not.

```python
def score_code(code: str, history: list[dict]) -> tuple[float, str]:
    history_text = ""
    for i, entry in enumerate(history, 1):
        history_text += (
            f"--- Attempt {i} ---\n"
            f"Code:\n{entry['code']}\n"
            f"Your previous score and feedback:\n{entry['feedback']}\n\n"
        )
    # ...
```

## Refining the code

When the score comes back below the threshold, the refiner kicks in - also `claude-opus-4-8`. It gets the full history plus the latest structured diff and applies the changes.

```python
def refine_code(history: list[dict]) -> str:
    refine_response = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        system="You are a coding agent that responds only with source code...",
        messages=[{
            "role": "user",
            "content": (
                f"Original prompt:\n{ORIGINAL_PROMPT}\n\n"
                f"History of previous attempts:\n{history_text}"
                f"Apply the structured diff from the latest feedback to produce an improved version."
            ),
        }],
    )
    return refine_response.content[0].text
```

Without the history injection, the refiner might fix one thing and accidentally break someting it doesn't know was already fixed in a previous pass.

## The loop

The refinement loop itself is pretty clean:

```python
refinement_history: list[dict] = []
refinements = 0

while True:
    score, scorer_text = score_code(agent2_code, refinement_history)

    if score >= MIN_SCORE:
        log(f"Score {score}/10 — accepted.")
        break

    if refinements >= MAX_REFINEMENTS:
        log(f"Score {score}/10 — maximum refinements reached. Stopping.")
        sys.exit(1)

    refinement_history.append({"code": agent2_code, "feedback": scorer_text})
    refinements += 1
    agent2_code = refine_code(refinement_history)
```

Each cycle: score → check threshold → refine → score again. The history list grows with every pass.

## Running Agent 2

Once the loop exits with an accepted score, Agent 1 writes the final code to a temp file and runs it:

```python
with tempfile.NamedTemporaryFile(
    mode="w", suffix=".py", delete=False, dir=os.path.dirname(__file__)
) as f:
    f.write(agent2_code)
    agent2_path = f.name

proc = subprocess.Popen(
    [sys.executable, agent2_path],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
)
stdout, stderr = proc.communicate(timeout=60)
```

The temp file gets cleaned up in a `finally` block no matter what happens. stdout and stderr are both captured - if Agent 2 blows up you'll see why.

{{< notice info >}}
> **Info**
> This is what makes it genuinely agentic. Agent 1 isn't just generating code for a human to run - it's generating, scoring, refining, and executing the result itself.
{{< /notice >}}

## Model choices

I used different models for different roles and I did this deliberately. The generator and refiner both use `claude-opus-4-8` because they need the reasoning capacity to either produce or correctly apply a structured diff. The scorer uses `claude-haiku-4-5-20251001` because scoring is cheaper work - fast and sufficient. You could swap haiku for sonnet if you want richer feedback, but I haven't found it necessary.

## Running it

Drop the script in a directory with a `.env` file containing your `ANTHROPIC_API_KEY`:

```bash
pip install anthropic python-dotenv
python agent1.py          # normal output
python agent1.py --debug  # full verbose output
```

The output looks something like this:

```
=== Agent 1 (PID 12345): generating Agent 2 ===
Score 9.8/10 — accepted.
=== Agent 1 (PID 12345): running Agent 2 ===
=== Agent 2 (PID 12346) ===
Agent 2 output: 2 + 2 equals 4.
```

## Summary

This is a simple but flexible pattern for self-improving code generation. The structured diff format, full history passing, and different model tiers for different roles are the things that make it actually work.

The next step is more than likely going to be introducing a truly distributed message bus similar to the article here [https://codecowboy.io/ai/autonomous-ai-agents/](https://codecowboy.io/ai/autonomous-ai-agents/). This way, each of the agents could refine others within the network and work together toward a shared goal.

I'm already thinking about implementing a shared goal deconstructor.

