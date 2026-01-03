# ACE Framework Explained

This guide walks through the ACE (Agentic Context Engineering) codebase so you can quickly understand how the system works and how to run it. The goal is to make the feedback-driven loop between the three agents—Generator, Reflector, and Curator—easy to visualize and reproduce.

## Big Picture

ACE keeps a **playbook** (structured notes and strategies) that grows over time. For each task sample, the system:

1. Generates an answer using the playbook.
2. Reflects on mistakes to produce tagged feedback.
3. Curates the playbook using the feedback so future generations improve.

### End-to-End Flow

```mermaid
flowchart TD
    A[Input sample (question + context + ground truth)] --> B[Generator
produces answer using playbook]
    B -->|answer, bullet_ids| C{Correct?}
    C -- Yes --> G[Log usage and accuracy]
    C -- No --> D[Reflector analyzes reasoning
+ tags helpful/harmful bullets]
    D --> E[Curator creates operations
(ADD/UPDATE/MERGE/DELETE)]
    E --> F[Apply operations to playbook
(update bullet IDs)]
    F --> B
    G -->|periodically| H[Evaluate on validation/test sets]
    H -->|save best| I[Best playbook snapshot]
```

## Core Components

### `ace/ace.py` – Orchestrator
- Initializes LLM clients and the three agents.
- Provides `run(mode=...)` for offline training, online training/testing, or evaluation-only workflows.
- Handles logging, saving intermediate playbooks, and tracking best validation accuracy.
- Uses helper methods to run tests (`_run_test`), train a single sample (`_train_single_sample`), and manage offline/online loops.

### Agents in `ace/core`
- **Generator (`generator.py`)**: Formats prompts with the current playbook and optional reflection, calls the LLM, and extracts referenced bullet IDs from responses.
- **Reflector (`reflector.py`)**: Reviews the generator’s reasoning, tags bullets as helpful/harmful/neutral, and returns structured feedback (JSON when available).
- **Curator (`curator.py`)**: Converts reflection feedback into structured operations (add/update/merge/delete) and applies them to the playbook while keeping token budgets and IDs consistent.
- **Bulletpoint Analyzer (optional)**: Deduplicates similar bullets when enabled.

### Playbook Utilities (`playbook_utils.py`)
- Extracts bullet JSON from free-form LLM text, applies curator operations, merges/refines bullet points, and keeps token counts and stats up to date.
- Provides helper functions like `extract_answer`, `count_tokens`, `get_playbook_stats`, and bullet usage logging.

### Data Pipeline Responsibilities
- A **data processor** (passed into `ACE.run`) defines `answer_is_correct` and `evaluate_accuracy`, so ACE remains task-agnostic.
- Evaluation functions (`evaluate_test_set`) call the Generator with the current playbook across datasets and compute accuracy.

## Using ACE

### 1) Install and Configure
- Install dependencies with `pip install -r requirements.txt`.
- Set up API keys in a `.env` file (see `README.md` for providers like SambaNova, Together, or OpenAI).

### 2) Initialize ACE
```python
from ace import ACE
from utils import initialize_clients

ace_system = ACE(
    api_provider="sambanova",            # or "together", "openai"
    generator_model="DeepSeek-V3.1",
    reflector_model="DeepSeek-V3.1",
    curator_model="DeepSeek-V3.1",
    max_tokens=4096,
    use_bulletpoint_analyzer=False
)
```

### 3) Prepare a Config
Common knobs (defaults shown where applicable):
- `num_epochs` (1) – epochs for offline training.
- `max_num_rounds` (3) – reflection rounds for incorrect answers.
- `curator_frequency` (1) – how often to run curator steps.
- `eval_steps` (100) – validation cadence during offline training.
- `online_eval_frequency` (15) – update cadence for online mode.
- `playbook_token_budget` (80000) – cap on playbook size.
- `json_mode` (False) – request structured JSON outputs.
- `no_ground_truth` (False) – disable ground-truth use in reflection.

### 4) Run Modes
- **Offline**: `ace_system.run(mode="offline", train_samples=..., val_samples=..., test_samples=..., data_processor=..., config=...)`
  - Trains on `train_samples`, validates periodically, and optionally tests before/after training.
- **Online**: `ace_system.run(mode="online", test_samples=..., data_processor=..., config=...)`
  - Streams through the test set once, updating the playbook as it goes.
- **Eval Only**: `ace_system.run(mode="eval_only", test_samples=..., data_processor=..., config=...)`
  - No training—just evaluate with the current playbook.

### 5) Outputs You Get
- `intermediate_playbooks/` and snapshots of best/final playbooks.
- `bullet_usage_log.jsonl` for which bullets were used and whether they helped.
- `train_results.json`, `test_results.json`, and detailed LLM logs.

## Tips for New Contributors
- Trace the loop starting in `ACE._train_single_sample` to see how generation, reflection, and curation connect.
- When debugging curator behavior, inspect the logged operation diffs and the applied playbook JSON.
- Keep the playbook token budget in mind when adding new prompts or operations—curation will prune/merge to stay within limits.

You now have the mental model to navigate the code and run ACE with your own datasets. Happy hacking!
