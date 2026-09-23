<p align="center">
  <img src="assets/logo.png" alt="AgentBug-Smith" width="720">
</p>

<p align="center">
  <a href="https://github.com/1123qqa/ICLR2027">Code</a>
  ·
  <a href="https://huggingface.co/buckets/IMICLRAUTHOR/live-harness-bench">Live-Harness Bench (200 instances)</a>
</p>

AgentBug-Smith reproduces harness bugs from open-source agent systems and releases them as **Live-Harness Bench**. The Hugging Face bucket stores the 200 executable instances. This repository stores the pipeline and every other experiment record.

## Pipeline

<p align="center">
  <img src="assets/e2e.png" alt="End-to-end pipeline" width="860">
</p>

Stage 1 retrieves agent repositories and keeps issues whose patches edit a harness component. On a labeled sample it reaches 95% repository accuracy and 92% issue accuracy.

<p align="center">
  <img src="assets/e2e-p2.png" alt="Harness bug identification" width="860">
</p>

Stages 2 and 3 build a container and a test that fails on the buggy commit and passes after the developer patch. On the shared pool of 225 issues, success is 45/225 (GPT-4.1-mini), 46/225 (Kimi-k2.5), and 63/225 (DeepSeek-v3.2). SWE-Factory reaches 21, 19, and 31 on the same pool. SWE-bench-Live reaches 6, 6, and 1.

<p align="center">
  <img src="assets/e2e-p3.png" alt="Environment and test generation" width="860">
</p>

The released benchmark has 200 instances, drawn from repositories that average 123k lines of code. Gold patches average 3.8 files and 202.9 changed lines. Component shares are 31.5% tool registry, 30.5% context and memory, 22.0% lifecycle and orchestration, 11.5% observability, and 4.5% governance.

<p align="center">
  <img src="assets/pie_issues_distribution.png" alt="Distribution over harness components" width="420">
</p>

<p align="center">
  <img src="assets/temporal_distribution.png" alt="Temporal distribution of issues" width="860">
</p>

mini-SWE-agent, OpenHands, and AutoCodeRover correctly resolve 9.00%, 8.50%, and 3.50% of the 200 instances. A skill trained on 121 issues from repositories held out of the test set raises mini-SWE-agent's correct resolution on the remaining 79 issues from 1.27% to 7.59%.

## Layout

| Path | What it is |
| --- | --- |
| `src/`, `exp/`, `prompt/`, `conf/` | Stage 2 and 3: Dockerfile loop, test generation, fail-to-pass check |
| `identification/` | Stage 1 code: awesome-list retrieval, GitHub Archive, issue crawl, issue filter |
| `rq1/f2p_by_models/` | Reproduction logs. See the note below before comparing models |
| `rq2/` | Identification labels and stability runs |
| `rq4/` | Skill split (121 / 79), `SKILL.md`, and rollout scores |
| `agent/patches/` | Patches from mini-SWE-agent, OpenHands, and AutoCodeRover |
| `agent/evaluation/` | Fail-to-pass logs for those patches |
| `data/issues/` | Issue JSON fed to the pipeline |
| `misc/identification/` | Raw repository and issue dumps from stage 1 |
| `misc/Agent-Issues.xlsx` | Working spreadsheet, including sheets that are not paper tables |
| `baselines/` | Launch scripts for SWE-Factory and SWE-bench-Live |
| `assets/` | Figures used above |

Two layout details matter when you read the logs:

- There is no `rq3/` directory. Environment-generation and test-generation numbers are columns of the same `rq1/` runs, not a second experiment.
- `rq1/f2p_by_models/gpt-4.1-mini/` contains only the 25, 50, 70, and 80-issue snapshots. The GPT run that enters the 225-issue comparison is `rq1/f2p_by_models/extended_gpt_f2p/`. Kimi and DeepSeek logs for that comparison are the sibling directories `kimi-k2.5/` and `deepseek-v3.2/`.
- `data/issues/issues_10`, `issues_25`, `issues_50`, `issues_70`, and `issues_80` are cumulative snapshots. `issues_<repo>/` folders are per-repository dumps. They are inputs, not the 200-instance benchmark.

## Setup

You need Python 3.10 or newer, Docker, a GitHub token, and an OpenAI-compatible endpoint. The experiments used [Forge](https://forge.tensorblock.co) as that endpoint.

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

Edit `.env` and replace the placeholder `forge-key` and `ghp_token`. `MODEL` selects the backbone. The default in code is `tensorblock/gpt-4.1-mini`.

### One issue

`exp/end-end.py` does not take a path on the command line. It runs the JSON assigned to `_ISSUE_JSON` near the top of that file. The checked-in default, `data/issues/issue_2805.json`, is not a file in this tree. Point it at a real file, for example `data/issues/issues_80/issue_177.json`, then:

```bash
python exp/end-end.py
```

The run writes `result/<issue>_<timestamp>/` with `env.dockerfile`, the generated test, `summary.json`, and the logs. Docker must be running. The script clones the upstream repository named in the issue JSON.

### Many issues

`exp/batch_end_end.py` reads a manifest and sets `_ISSUE_JSON` for each entry without editing `end-end.py`. Paths are relative to the repository root.

```json
{"issues": ["data/issues/issues_80/issue_177.json"]}
```

```bash
python exp/batch_end_end.py manifest.json --continue-on-error
```

### Score a patch

Download the benchmark from the Hugging Face bucket (the `benchmark/` prefix) or point `--artifacts-dir` at a local `result/` tree. Each child directory must be one instance.

Developer patch, taken from the issue JSON:

```bash
python exp/evaluate_f2p.py \
  --artifacts-dir /path/to/benchmark \
  --output-dir /tmp/f2p_eval \
  --patch-mode issue_json
```

Agent patch. The script looks for `result_<instance>/generated_patch.diff` under `--custom-patches-dir`, which matches `agent/patches/<agent>/`:

```bash
python exp/evaluate_f2p.py \
  --artifacts-dir /path/to/benchmark \
  --output-dir /tmp/f2p_eval \
  --patch-mode generated_diff \
  --custom-patches-dir agent/patches/mini-swe-agent-patches
```

### Stage 1

Identification is a separate code tree. Entry points:

| Step | Command | Notes |
| --- | --- | --- |
| Awesome-list repositories | `python identification/repo-hook/github/main.py` | Needs `GITHUB_TOKEN` and the Forge variables in `.env` |
| GitHub Archive stream | `python identification/repo-hook/github_archive/main.py` | See that directory's README |
| Issue crawl | `python identification/issue-hook/issue_crawler.py` | Closed issues linked to a merged pull request |
| Issue filter | `identification/pipeline/issue-filtering/` | Rule filter, then the harness judge |

Those READMEs still mention the paths of the machine where the crawl ran (`/home/cc`, `/home/fdse`). Use the commands above from this repository root instead of those absolute paths. `baselines/run_RQ1_swe-factory.sh` and `baselines/run_RQ1_sbl.sh` likewise embed that machine's conda prefix and checkout path. They record how the baselines were launched. They are not a portable installer.
