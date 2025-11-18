# nanochat-contracts Runbook

This document is written for a first-time operator who wants to reproduce the full contract-focused nanochat pipeline on rented GPUs. Follow the steps in order without skipping. Every command is meant to be run on the remote GPU machine unless explicitly noted as "local".

> **Nov 18, 2025 status**  
> We attempted to rerun the base → mid → SFT stages to repair the RL checkpoint’s `####` failure mode. The recovery sprint was halted after repeated checkpoint corruption during base retraining and escalating GPU costs. The instructions below remain valid, but expect to allocate fresh budget and time for a full pipeline restart (base + mid + SFT). Always archive checkpoints off the instance and validate them with `python - <<'PY' ... torch.load(... )` before starting the next stage.

## Quick command reference
- **SSH into node:** `ssh -i ~/.ssh/nanoCHAT.pem ubuntu@192.222.55.121`
- **Attach/detach training screen:** `screen -r run1000` → view logs → `Ctrl-A D`
- **List screens:** `screen -ls`
- **Reattach even if “attached”:** `screen -d -r run1000`
- **Start full run:** `screen -S run1000 && bash run1000.sh`
- **Check WANDB run:** open the URL printed in logs (or run `wandb status`)
- **Exit finished screen:** attach (`screen -r name`) then type `exit`
- **Stop instance after run:** terminate via Lambda dashboard once all screens are closed

## 0. Prerequisites (local workstation)

1. **GitHub + HuggingFace accounts**
   - GitHub: ensure you can `git clone` private repos from your laptop.
   - HuggingFace: log in at <https://huggingface.co/>, add a billing method (done), and create a personal access token (Settings ▸ Access Tokens ▸ New token, scope = `read`). Save this token locally; you will paste it later.
2. **Accept dataset terms**
   - Visit <https://huggingface.co/datasets/pile-of-law/pile-of-law> and click *Agree* under the license banner.
   - Visit <https://huggingface.co/datasets/theatticusproject/cuad-qa> and accept those terms as well.
3. **WANDB (optional but recommended)**
   - Create an account at <https://wandb.ai/> and copy your API key.
4. **Local SSH key**
   - Ensure you have an SSH key pair (`~/.ssh/id_rsa` or similar). Upload the public key to the GPU provider if asked.

Once the above is complete, move on to the remote steps.

## 1. Rent a cost-effective 8×H100 GPU server

1. **Provider recommendation**: Lambda Cloud (reliable pricing ~$23–24/hr at the time of writing).
2. **Create the server**
   - Log in to <https://cloud.lambdalabs.com>.
   - Navigate to *Create Instance* ▸ *Lambda Cloud GPUs* ▸ choose **8×H100 80GB**.
   - Select the latest Ubuntu 22.04 image, 1.6 TB NVMe.
   - Attach the SSH key you uploaded earlier and launch the instance.
3. **Record access details**
   - Note the public IP address (`INSTANCE_IP`).
   - Confirm the instance status is “running”.

## 2. Connect to the instance (local terminal)

```bash
ssh ubuntu@INSTANCE_IP
```
Replace `INSTANCE_IP` with the IP recorded above. You should now be on the remote server (prompt will show `ubuntu@something`).

## 3. Prepare the environment on the server

### 3.1 System packages

```bash
sudo apt update
sudo apt install -y build-essential git screen htop
```

### 3.2 Install uv and Rust toolchains

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
```

### 3.3 Clone the repo

```bash
cd $HOME
git clone https://github.com/samerGMTM22/My-nanochat.git my-nanochat
cd my-nanochat
```

### 3.4 Environment variables

```bash
export OMP_NUM_THREADS=1
export NANOCHAT_BASE_DIR="$HOME/.cache/nanochat"
mkdir -p "$NANOCHAT_BASE_DIR"
```

### 3.5 Python environment

```bash
command -v uv >/dev/null || source "$HOME/.local/bin/env"
[ -d ".venv" ] || uv venv
source .venv/bin/activate
uv sync --extra gpu
```

### 3.6 Build the Rust tokenizer

```bash
uv run maturin develop --release --manifest-path rustbpe/Cargo.toml
```

At this point, the repository dependencies are ready.

## 4. Authenticate CLI tools

### 4.1 HuggingFace CLI (for dataset downloads)

```bash
huggingface-cli login
```
Paste the `hf_...` token you created earlier. This enables the `datasets` library to pull streaming data.

### 4.2 Weights & Biases (optional)

```bash
export WANDB_API_KEY=<your_wandb_key>
export WANDB_RUN=nanochat-contracts-d32
wandb login $WANDB_API_KEY
```

If you skip WANDB, set `WANDB_RUN=dummy` when launching scripts.

## 5. Data preparation

All intermediate data is written under `~/.cache/nanochat/`. Run the following commands inside the repo root (`~/my-nanochat`).

### 5.1 Contract corpus (atticus_contracts)

```bash
python dev/prepare_contract_corpus.py --target-shards 120
```
- Streams the dataset and writes shards to `~/.cache/nanochat/contract_data/`.
- This can take multiple hours; consider running inside `screen` to avoid losing progress (`screen -S contracts`, run command, detach with `Ctrl-A D`).

### 5.2 FineWeb-Edu base shards and tokenizer

```bash
python -m nanochat.dataset -n 16        # immediate shards
python -m nanochat.dataset -n 800 &     # background download
python -m scripts.tok_train --max_chars=4000000000
python -m scripts.tok_eval
```
The background `-n 800` command keeps fetching shards while other steps run.

## 6. Full training pipeline

> **Tip:** Use `screen -S run1000` to capture the entire run. You can also redirect output with `tee`.

Launch:

```bash
bash run1000.sh
```

`run1000.sh` performs, in order:
1. Environment and tokenizer setup (already done; script re-validates).
2. Base pretraining (`torchrun ... scripts.base_train`).
3. Base loss/eval scripts.
4. Contract data verification (`python dev/prepare_contract_corpus.py`).
5. Contract midtraining via `scripts.contract_mid_train`.
6. SFT prep `python dev/prepare_cuad_sft.py` and `scripts.chat_sft`.
7. Intermediate evaluations (`scripts.chat_eval`).
8. Final report generation (`python -m nanochat.report generate`).

Monitor progress with `screen -r run1000` or by tailing `speedrun.log` if you redirected output.

## 7. Post-run tasks

1. Confirm checkpoints are saved under `~/.cache/nanochat/base_checkpoints`, `mid_checkpoints`, and `sft_checkpoints`.
2. Generate the report again if needed: `python -m nanochat.report generate`.
3. Optionally launch the web chat UI: `python -m scripts.chat_web` (tunnel the port with SSH if you want to access it locally).
4. When finished, stop the Lambda instance to halt billing (`lambda cloud` dashboard).

## 8. Troubleshooting checklist

- **Out-of-memory during training**: lower `--device_batch_size` in `run1000.sh` (e.g., set to 6) and restart from that phase.
- **HuggingFace download errors**: ensure `huggingface-cli login` succeeded and re-run the relevant prep script.
- **WANDB timeouts**: set `WANDB_RUN=dummy` to disable logging.
- **Disk usage**: check `df -h`; the cache plus checkpoints can exceed 1 TB. Remove unwanted shards if needed.
- **Contract midtraining hangs**: see `CONTRACT_MID_DEBUG.md` for the full postmortem. TL;DR—remove duplicate `next(train_loader)` calls and keep contract parquet shards chunky (≈20–30 files, 200–400 MB each) before running on 8×GPU.
- **CUAD SFT prep fails**: `python -m dev.prepare_cuad_sft` relies on HF `trust_remote_code`, which is deprecated. Download the JSON files directly (`huggingface-cli download theatticusproject/cuad-qa cuad_data.json cuad_test.json --repo-type dataset --local-dir ~/.cache/nanochat/cuad-qa-local`), convert them to `cuad_sft_conversations.jsonl` manually, and document progress in `PERSONALIZATION_SFT_PLAN.md`.
- **Dataset cache corruption (ARC/GSM8K)**: If `load_dataset('allenai/ai2_arc', ...)` or GSM8K throws `TypeError: must be called with a dataclass type or instance` even after pinning `datasets==2.21.0`, the cached metadata is stale. Fix = delete **all** cache folders for the dataset:
  ```bash
  rm -rf ~/.cache/huggingface/hub/datasets--allenai--ai2_arc
  rm -rf ~/.cache/huggingface/hub/.locks/datasets--allenai--ai2_arc
  rm -rf ~/.cache/huggingface/datasets/allenai___ai2_arc
  # repeat for openai/gsm8k if needed
  ```
  Then rerun `load_dataset` to rebuild fresh metadata. See “Dataset cache recovery postmortem” below for the full analysis/log.

## 9. Next steps (after run completes)

Once you confirm training finished, come back here so we can:
1. Gather all checkpoint artifacts.
2. Upload the final model to HuggingFace (model card, `git lfs` repo, etc.).
3. Demonstrate inference on HuggingFace (Hosted Inference API or Spaces).

Let me know when your training run is done and we’ll continue with that publishing workflow.

## Command cheat sheet
- **SSH**: `ssh -i ~/.ssh/nanoCHAT.pem ubuntu@INSTANCE_IP`
- **Activate env**: `cd ~/my-nanochat && source .venv/bin/activate`
- **Screen basics**: `screen -S name`, detach `Ctrl-A D`, list `screen -ls`, reattach `screen -r name`, force `screen -d -r name`
- **Contract prep**: `python -m dev.prepare_contract_corpus --target-shards 120`
- **FineWeb download**: `python -m nanochat.dataset -n 16 && python -m nanochat.dataset -n 800 &`
- **Tokenizer**: `python -m scripts.tok_train --max_chars=4000000000` then `python -m scripts.tok_eval`
- **Full pipeline**: `screen -S run1000 && bash run1000.sh`
- **Monitor training**: `screen -r run1000` (detach after viewing)

## Error-handling scenarios
- **SSH timeout**: Wait for Lambda instance to show “Running” before `ssh`; ensure `chmod 600 ~/.ssh/nanoCHAT.pem`.
- **`ModuleNotFoundError: nanochat`**: Run scripts via `python -m dev.prepare_contract_corpus …` from repo root with venv active.
- **HuggingFace dataset script errors**: Downgrade `datasets` inside `.venv`:
  ```bash
  pip uninstall -y datasets
  pip install "datasets==2.21.0"
  ```
- **Screen already attached**: `screen -d -r name` to reattach, then detach with `Ctrl-A D`.
- **WANDB authentication**: `export WANDB_API_KEY=... && wandb login $WANDB_API_KEY` before `run1000.sh`.
- **Contract shard count < target**: rerun `python -m dev.prepare_contract_corpus --target-shards 120`; script resumes where it left off.

## Dataset cache recovery postmortem (Nov 16 2025)
During RL evaluation, `scripts.chat_eval` crashed when ARC (allenai/ai2_arc) attempted to load via HuggingFace `datasets`. Despite running `datasets==2.21.0`, the loader raised:

```
TypeError: must be called with a dataclass type or instance
  File ".../datasets/features/features.py", line 1467, in generate_from_dict
    field_names = {f.name for f in fields(class_type)}
```

### Root cause
Earlier runs on the same Lambda VM pulled ARC using a different `datasets` version. Its metadata (`dataset_info.json`, `state.json`) serialized dataclass structures incompatible with 2.21.0. The HuggingFace cache spans **three** locations:
1. `~/.cache/huggingface/hub/datasets--<org>--<name>` (new Hub cache)
2. `~/.cache/huggingface/hub/.locks/datasets--<org>--<name>` (lock files)
3. `~/.cache/huggingface/datasets/<org>___<name>` (legacy triple-underscore directory)

Only deleting `~/.cache/huggingface/datasets/allenai___ai2_arc` left the corrupted metadata in the Hub cache, so the error persisted.

### Resolution steps
1. Confirmed `datasets` version inside venv (`python -c "import datasets; print(datasets.__version__)"` → 2.21.0).
2. Reproduced error via `python -c "from datasets import load_dataset; load_dataset('allenai/ai2_arc','ARC-Easy',split='test')"`.
3. Enumerated all cache folders containing “arc”:  
   `find ~/.cache/huggingface -type d -name "*arc*"`
4. Deleted every relevant path:
   ```bash
   rm -rf ~/.cache/huggingface/hub/datasets--allenai--ai2_arc
   rm -rf ~/.cache/huggingface/hub/.locks/datasets--allenai--ai2_arc
   rm -rf ~/.cache/huggingface/datasets/allenai___ai2_arc
   ```
5. Repeated for GSM8K (`openai--gsm8k`).
6. Reran the loader; ARC downloaded fresh metadata and succeeded (`✓ ARC-Easy loaded: 2376 examples`).

### Takeaway
When switching `datasets` versions or copying VM images, always clear **all** HuggingFace cache directories for problematic datasets. Otherwise, incompatible metadata will keep crashing even after reinstalling packages.

### RL evaluation cache incident (Nov 16 2025)
While launching `scripts.chat_eval` after RL, ARC/GSM8K (and later MMLU) repeatedly crashed with the dataclasses error above. Consolidated incident log:

**Symptoms**
- `torchrun ... scripts.chat_eval -- -i rl` aborted immediately when ARC tried to load.
- Error: `TypeError: must be called with a dataclass type or instance` (`features.py:1467`) despite `datasets==2.21.0`.
- Training data/checkpoints unaffected (they don’t use HF `datasets`).

**Root cause**
- Stale `dataset_info.json/state.json` files created by an earlier `datasets` version get deserialized by 2.21.0 → incompatible dataclass layout → failure.
- HuggingFace stores dataset metadata in *three* locations (Hub cache, `.locks`, legacy cache). Clearing only one leaves the others corrupt.

**Resolution steps**
1. SSH + activate venv (`source ~/my-nanochat/.venv/bin/activate`).
2. Confirm version: `python -c "import datasets; print(datasets.__version__)"` → 2.21.0.
3. Reproduce: `python -c "from datasets import load_dataset; load_dataset('allenai/ai2_arc','ARC-Easy',split='test')"` → TypeError.
4. Inspect caches: `find ~/.cache/huggingface -type d -name "*arc*"`.
5. Delete *all* ARC folders:
   ```bash
   rm -rf ~/.cache/huggingface/hub/datasets--allenai--ai2_arc
   rm -rf ~/.cache/huggingface/hub/.locks/datasets--allenai--ai2_arc
   rm -rf ~/.cache/huggingface/datasets/allenai___ai2_arc
   ```
6. Repeat for GSM8K (replace `allenai` → `openai`, `ai2_arc` → `gsm8k`).
7. When MMLU subsequently failed, run the “nuclear option” to delete *all* HF dataset caches:
   ```bash
   rm -rf ~/.cache/huggingface/datasets
   rm -rf ~/.cache/huggingface/hub/datasets--*
   rm -rf ~/.cache/huggingface/hub/.locks/datasets--*
   ```
8. Rerun eval inside a screen (`screen -S rl_eval`), re-download fresh datasets, and let GSM8K/HumanEval/SpellingBee finish.

**Status**
- RL eval now proceeds normally; results log to WANDB and populate `report.md`.
- Training corpora/checkpoints untouched (custom parquet loader).

**Best practices**
- If dataset evals fail after a version bump or VM reuse, nuke all HF caches before retrying.
- Document the exact `rm -rf` commands in incident notes (done above).
