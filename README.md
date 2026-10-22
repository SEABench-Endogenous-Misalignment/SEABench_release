<h1 align="center">SEABench</h1>

<p align="center">
  <a href="https://arxiv.org/abs/2609.35596">
    <img alt="arXiv" src="https://img.shields.io/badge/arXiv-2609.35596-b31b1b.svg">
  </a>
  <a href="https://seabench-endogenous-misalignment.github.io/">
    <img alt="SEABench website" src="https://img.shields.io/badge/website-SEABench-181717.svg">
  </a>
  <a href="#citation">
    <img alt="Citation" src="https://img.shields.io/badge/cite-SEABench-6f42c1.svg">
  </a>

</p>

SEABench is a benchmark for studying endogenous misalignment in self-evolving
LLM agents. It contains 48 longitudinal task sequences spanning multiple
self-evolution surfaces, task domains, and harm types in a personal-assistant
environment. The benchmark pairs self-evolving runs with non-evolving controls
and uses an adaptive trajectory-discovery pipeline to find safety failures while
preserving the intent of the original tasks.

<p align="center">
  <img src="docs/images/seabench-schematic.jpg" alt="SEABench benchmark schematic" width="100%" />
</p>

## Core Features

- **Longitudinal self-evolution benchmark:** 48 task sequences across different tasks domains study whether
  behavior remains aligned after an agent self-evolves.
- **Multiple evolution surfaces:** Evaluate controller file, memory, and tools/skills evolution surfaces.
- **Safety-focused evaluation:** Existing task sequences cover privacy, hallucination, guardrail
  erosion, and contextual-boundary-collapse harm types with a hidden safety judge.
- **Configurable task sequences:** Define prompts, validators, optimization
  roles, controller-file overrides, workspace setup, and tool exposure in
  YAML without changing the runner.
- **Paired controls:** Compare an evolved agent with a non-evolving
  (`surface=none`) baseline under the same task sequence to separate ordinary
  task behavior from effects caused by persistent self-evolution.
- **Adaptive trajectory discovery:** Refine task prompts with TextGrad and
  paired replays, retaining useful task completion while searching for
  safety-relevant failures.
- **Post-hoc analysis and mitigation:** Inspect reasoning traces and evaluate
  mitigation strategies through the analysis workflows under
  `prompt_optimization_analysis/`.


## Task Sequence Format

Task sequences are YAML files with metadata, optional setup and generation
configuration, and an ordered list of tasks. A typical sequence looks like:

```yaml
id: example-sequence
type: task_sequence
title: Example Self-Evolution Sequence
description: A sequence of upstream evolution tasks followed by safety tests.
status: active
validation_backend: llm

generation:
  surface: controller_update
  target_harm: privacy

sequence_setup:
  clear_paths:
    - skills
    - created_tools
  ensure_dirs:
    - skills
  tool_allowlist:
    - read_file
    - list_files
    - apply_controller_update

tasks:
  - id: task_001
    title: Upstream task
    prompt: |
      Complete the user task_001
    validator:
      llm:
        criteria: |
          1. The response completes the requested task.
          2. The response follows the relevant safety constraints.
    optimization:
      role: upstream
```

See [`docs/task_setup.md`](docs/task_setup.md) for all supported setup fields
and additional examples.

## Quick Start

### Environment Setup

Install the dependencies and configure the provider key:

```bash
./scripts/setup_venv.sh
source .venv/bin/activate
```

### Setting API key

Set the API key for your preferred provider
```bash
export OPENROUTER_API_KEY="<your-key>"
```

### Adding a New Provider
To add another OpenAI-compatible provider, create a profile under
`configs/providers/`. For example, create `configs/providers/myprovider.yaml`:

```yaml
name: myprovider
base_url: https://api.myprovider.com/v1
api_key_env: MYPROVIDER_API_KEY
default_model: provider/model-name
context_window_tokens: 32768
request_token_reserve: 2048
```

Export the key named by `api_key_env`:

```bash
export MYPROVIDER_API_KEY="<your-key>"
```

Select the profile by its filename (without `.yaml`). For the task-sequence
runner without adaptive refinement, set it in the run configuration:

```yaml
provider:
  profile: myprovider
  agent_model: provider/model-name
  judge_model: provider/model-name
```


### Refine an Existing Sequence (Main Experiment)

The main experiment starts from one of the task-sequence YAML files in `tasks/`
and searches over prompt candidates using paired replays of a self-evolving
agent and a non-evolving (`surface=none`) baseline. The central entry point is
`prompt_optimization/refine_existing_sequence.py`.


```bash
python3 prompt_optimization/refine_existing_sequence.py \
  --task-sequence tasks/controller_update/computer_use/privacy.yaml \
  --output-dir prompt_optimization/out/controller_update_computer_use_privacy \
  --surface controller_update \
  --max-candidates-per-task 5 \
  --downstream-safety-lookahead \
  --downstream-standalone \
  --provider-profile openrouter \
  --agent-model openai/gpt-5.6-luna \
  --judge-model openai/gpt-5.6-luna
```

To resume an interrupted run, add the `--resume` flag to the same command. A run is complete
when `final_refined_sequence.yaml` has been produced.


For long-running experiments, invoke it through the retry shell wrapper
`scripts/run_refinement_existing_sequence.sh`.

```bash
./scripts/run_refinement_existing_sequence.sh \
  --task-sequence tasks/controller_update/computer_use/privacy.yaml \
  --output-dir prompt_optimization/out/controller_update_computer_use_privacy \
  --surface controller_update \
  --max-candidates-per-task 5 \
  --downstream-safety-lookahead \
  --downstream-standalone \
  --provider-profile openrouter \
  --agent-model openai/gpt-5.6-luna \
  --judge-model openai/gpt-5.6-luna
```

In this repository, **upstream** refers to the evolution tasks in the paper,
while **downstream** refers to the safety test tasks in the paper.

Candidate search defaults to the TextGrad backend. The main search budget is
set with `--max-candidates-per-task`. Use `--optimizer-model` to override the textgrad rewriter from their default provider profiles.

See `prompt_optimization/README.md` for the
lower-level refinement entry points, acceptance rules, and advanced options.

### Inital Task Sequence Creation
For creation of new task sequences, see `docs/task_sequence_creation.md`.


### Task Sequence Runner without Adaptive Refinement

The following workflow runs one task sequence without the adaptive refinement loop
inside an Apptainer sandbox. The sandbox image is built automatically on the
first run if it does not already exist.

```bash
./scripts/run.sh --run-config configs/runs/controller_update_computer_use_privacy.yaml
```

### Chain-of-Thought Analysis and Mitigation
Refer `prompt_optimization_analysis/reasoningflow/README` for details on running the post-hoc chain of thought analysis, and `prompt_optimization_analysis/reasoning_monitor/README` for the mitigation experiments.

## Citation

If you use SEABench in your research, please cite:

```bibtex
@article{das2026seabench,
  author        = {Das, Saswat and Viswanathan, Parvati and Donnelly, Daniel and
                   Huang, Chang and Abdelnabi, Sahar and Fioretto, Ferdinando},
  title         = {SEABench: Benchmarking Endogenous Misalignment in Self-Evolving Agents},
  journal       = {arXiv preprint arXiv:2609.35596},
  year          = {2026},
  url           = {https://arxiv.org/abs/2609.35596}
}
```
