# Agents Playbook

## Mission
Operate `nanochat-contracts` as a specialized LLM pipeline for contract, procurement, and supply-chain intelligence. Every change should preserve the upstream nanochat minimalism while keeping this fork aligned with SamerGMTM22/My-nanochat.

## Source control guidelines
- Treat `main` as the integration branch; create topic branches per feature (`feat/contract-mid`, `fix/cuad-format`, etc.).
- Rebase onto upstream `karpathy/nanochat` periodically but only after verifying compatibility with the contract-specific mods.
- Never rewrite history on shared branches. Use merge requests or PRs against the fork (GitHub repo `samerGMTM22/My-nanochat`).
- Commit messages: short imperative subject, optional body describing datasets/scripts touched (e.g., `Add CUAD prep + chat_sft hook`).

## Workflow best practices
1. **Environment parity**
   - `sudo apt update && sudo apt install -y build-essential git screen htop`
   - `curl -LsSf https://astral.sh/uv/install.sh | sh`
   - `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y`
   - `git clone https://github.com/samerGMTM22/My-nanochat.git my-nanochat`
   - `cd ~/my-nanochat && export OMP_NUM_THREADS=1 && export NANOCHAT_BASE_DIR="$HOME/.cache/nanochat"`
   - `uv venv` → `source .venv/bin/activate` → `uv sync --extra gpu`
   - `uv run maturin develop --release --manifest-path rustbpe/Cargo.toml`
2. **Auth + reporting**
   - HuggingFace: `huggingface-cli login` (paste `hf_...`)
   - WANDB: `export WANDB_API_KEY=... && wandb login $WANDB_API_KEY && export WANDB_RUN=nanochat-contracts-d32`
3. **Data caching**
   - Contract corpus: `screen -S contract-prep`, then `python -m dev.prepare_contract_corpus --target-shards 120`
   - FineWeb shards: `python -m nanochat.dataset -n 16`, followed by `python -m nanochat.dataset -n 800 &`
   - CUAD SFT: `python -m dev.prepare_cuad_sft`
   - Downgrade `datasets` if HF removes script support: `pip uninstall -y datasets && pip install "datasets==2.21.0"`
4. **Tokenizer**
   - `python -m scripts.tok_train --max_chars=4000000000`
   - `python -m scripts.tok_eval`
5. **Training stages**
   - Base: `torchrun --standalone --nproc_per_node=8 -m scripts.base_train -- --depth=32 --device_batch_size=8 --run=$WANDB_RUN`
   - Mid (contracts): `torchrun --standalone --nproc_per_node=8 -m scripts.contract_mid_train -- --device_batch_size=8 --run=$WANDB_RUN`
   - SFT (CUAD): `torchrun --standalone --nproc_per_node=8 -m scripts.chat_sft -- --run=$WANDB_RUN`
   - Evaluations: `torchrun ... -m scripts.base_eval`, `torchrun ... -m scripts.chat_eval -- -i mid|sft`
   - Full pipeline script: `screen -S run1000 && bash run1000.sh` (detach with `Ctrl-A, D`).
6. **Testing / QA**
   - Unit tests under `tests/`
   - Smoke tests: `python -m dev.prepare_contract_corpus --target-shards 1`, `python -m scripts.contract_mid_train --num_iterations=2 --device_type=cpu`
7. **Documentation**
   - Update `README.md`, `RUNBOOK.md`, and `Agents.md` whenever workflows or data sources change.

## Current run status (Nov 14, 2025)
- Contract corpus preparation succeeded with 120 shards (`~/.cache/nanochat/contract_data/shard_00000.parquet` … `shard_00119.parquet`).
- FineWeb base data download seeded all 800 shards; tokenizer training/eval completed with the custom 65,536 vocab.
- Full pipeline (`bash run1000.sh`) is running inside `screen -S run1000` on Lambda 8×H100; WANDB run name `nanochat-contracts-d32`.
- Active screens: `run1000` (training), previous `contract-prep` + `base-data` sessions closed once cleanup is done. Use `screen -ls` + `screen -r run1000` to monitor.

## Next milestones (after training completes)
1. **Artifact collection**
   - Gather checkpoints from `~/.cache/nanochat/{base,mid,sft}_checkpoints`.
   - Export tokenizer files and report (from `report.md`).
2. **WANDB & report review**
   - Pull loss curves, eval scores, and resource metrics.
   - Update `README.md` with actual evaluation numbers, training cost/time, WANDB links.
3. **HuggingFace publication**
   - Create model repo (e.g., `samerGMTM22/nanochat-contracts-d32`).
   - Upload tokenizer, base/mid/SFT checkpoints (probably share SFT checkpoint as primary).
   - Write model card summarizing datasets, training recipe, eval scores, and limitations.
4. **Repository sync**
   - Document run results in `README.md`, `RUNBOOK.md`, and this file.
   - Commit WANDB links, HF repo URL, inference instructions.
5. **Inference UX**
   - Option A: HF Inference Endpoint or Hosted Inference API using the uploaded model.
   - Option B: Deploy `scripts.chat_web` with the HF checkpoint (download via `huggingface_hub`).
   - Document how to point the UI at the HF model (token + checkpoint path).
6. **Optional persona SFT (post-run)**
   - Create `identity_persona.jsonl` with Q/A pairs (e.g., “Who created you?” → “Prince Samer Haddad…”).
   - Run a short SFT pass so the final HF checkpoint includes the persona:
     ```bash
     screen -S persona
     source .venv/bin/activate
     torchrun --standalone --nproc_per_node=8 \
       -m scripts.chat_sft \
       --run=${WANDB_RUN}-persona \
       --num_epochs=1 \
       --target_examples_per_step=32 \
       --custom_json_path=/path/to/identity_persona.jsonl \
       --source=sft \
       --model_tag=d32
     ```
   - Verify responses, then treat this updated checkpoint as the one to upload to HuggingFace.

## Pending guidance
- Once training finishes, update this file with timestamps and resulting metrics.
- After uploading to HuggingFace, record:
  - HF repo name + URL
  - SHA / tag of the uploaded checkpoint
  - Example inference commands (HF Inference API, `text-generation-inference`, or local `chat_web`).
- Capture WANDB screenshots/links for historical runs to show learning curves.
- Describe any tuning/bugfix learnings (e.g., downgrading `datasets` to 2.21.0 for scripted loaders).

## Post-training publishing checklist
1. **Collect artifacts**
   - SFT checkpoint folder (e.g., `~/.cache/nanochat/sft_checkpoints/d32/step_xxxx`)
   - Tokenizer files (`tokenizer.model`, `tokenizer.json`) from `~/.cache/nanochat/tokenizer`
   - `report.md`, WANDB run ID, eval outputs from `scripts.chat_eval`
2. **HuggingFace repo setup**
   - `huggingface-cli repo create samerGMTM22/nanochat-contracts-d32 --type model`
   - `git lfs install`
   - `git clone https://huggingface.co/samerGMTM22/nanochat-contracts-d32`
   - Copy checkpoint + tokenizer into the repo directory, create `README.md` model card (cite FineWeb-Edu, pile-of-law, CUAD)
   - `git add . && git commit -m "Upload nanochat-contracts-d32 SFT checkpoint" && git push`
3. **WANDB summary**
   - Download run summary via `wandb artifact get` or export metrics page
   - Embed WANDB link in HF model card + README
4. **Runbook updates**
   - Record final training duration, total cost, eval metrics, and HF URLs
5. **Inference integration**
   - HF Hosted Inference example:
     ```bash
     python - <<'PY'
     from huggingface_hub import InferenceClient
     client = InferenceClient("samerGMTM22/nanochat-contracts-d32", token="hf_xxx")
     resp = client.text_generation("Summarize the indemnity clause...")
     print(resp)
     PY
     ```
   - Local UI against HF checkpoint:
     ```bash
     HF_MODEL="samerGMTM22/nanochat-contracts-d32"
     python scripts/chat_web.py --model-from-hf $HF_MODEL
     ```

## LiaLeen Contracts 1 wrap plan (Nov 16, 2025)
1. **RL finish + eval**
   - Let `torchrun ... scripts.chat_rl --run=contract_rl_stage` reach Step 467/467; checkpoint lands under `~/.cache/nanochat/chatrl_checkpoints/d32`.
   - Immediately run `torchrun ... scripts.chat_eval -- -i rl` and log WANDB links; keep screen sessions tidy.
2. **Artifact sweep & sanity checks**
   - Collect base/mid/SFT (if any)/RL checkpoints, tokenizer (`~/.cache/nanochat/tokenizer`), and `report.md`.
   - Run `python -m nanochat.report generate` and a quick chat via `python -m scripts.chat_web --model-from-cache d32 --phase rl` to verify LiaLeen responses.
3. **Publish LiaLeen Contracts 1**
   - Create HF repo `samerGMTM22/LiaLeen-Contracts-1`; upload RL checkpoint (primary), tokenizer, report snippets, WANDB references.
   - Update README/RUNBOOK/Agents with final metrics, WANDB links, HF URL, inference instructions (HF endpoint + local chat_web), rename model references to LiaLeen Contracts 1.
   - Clean sensitive data: remove auth tokens/secrets from `~/.cache`, `.venv`, history before sharing.
4. **Optional persona SFT retry (if time)**
   - Root causes previously: HF datasets API incompatibility + role schema mismatch. Fix by pinning `datasets==2.21.0` or editing `TaskMixture` to only use `CustomJSON`, and ensure persona JSONL is `[user,assistant]` pairs.
   - If retried, rerun `torchrun ... scripts.chat_sft` + `chat_eval -i sft`, then decide whether SFT or RL weights ship to HF.
5. **Shutdown sequence**
   - Confirm HF upload + repo docs committed and pushed (`feat/lialeen-release` → PR).
   - Archive WANDB info, close screen sessions, delete temporary configs.
   - Terminate Lambda instance once artifacts are safe; future work (local inference, HF endpoint) can continue offline using the uploaded model.

## Error-handling log
- **HF dataset scripts disabled**: Resolved by pinning `datasets==2.21.0` inside `.venv` (`sudo chown -R ubuntu .venv` if needed). Relevant commands documented above.
- **Screen session conflicts**: Use `screen -d -r <name>` to force reattach; always detach via `Ctrl-A D`.
- **WANDB login reminders**: Ensure `WANDB_API_KEY` exported before `run1000.sh`; check `wandb status` if syncing stalls.
- **Contract sharding interruptions**: Re-run `python -m dev.prepare_contract_corpus --target-shards 120`; script resumes at next missing shard.
- **SSH timeouts while booting**: Wait until Lambda instance shows “Running” before connecting; trust host fingerprint on first login.
- **HF datasets cache corruption (Nov 16 2025)**: ARC/GSM8K/MMLU pulls crashed with `TypeError: must be called with a dataclass type or instance` even on `datasets==2.21.0`. Root cause: stale metadata spread across *three* cache locations (`~/.cache/huggingface/hub/datasets--...`, `hub/.locks/datasets--...`, `datasets/<org>___<name>`). Fix = delete all folders for the problematic dataset (or nuke all caches) so `load_dataset(...)` re-downloads fresh metadata; see RUNBOOK “Dataset cache recovery postmortem” for the full log/commands.

## Coding standards
- Match upstream formatting (PEP8, docstrings, inline comments only where non-obvious logic exists).
- Prefer reusing nanochat helpers (`get_base_dir`, `tokenizing_distributed_data_loader`) over duplicating logic.
- Maintain DDP compatibility; gate logging with `print0`.
- Keep configs override-able via `nanochat/configurator.py` (all tunables should be globals at top of scripts).

## Deployment checklist
- `python dev/prepare_contract_corpus.py`
- `torchrun ... scripts.contract_mid_train`
- `python dev/prepare_cuad_sft.py`
- `torchrun ... scripts.chat_sft`
- `python -m nanochat.report generate`
- `python -m scripts.chat_web`

Stick to this playbook so collaborating agents can jump in, understand the contract-specific extensions, and safely evolve the fork.

---

## ✅ LiaLeen Contracts 1 - COMPLETED (Nov 16, 2025)

### Training Pipeline Status
1. ✅ **Base Pretraining** - Completed (71,680 steps on FineWeb-Edu, CORE: 0.3066)
2. ✅ **Contract Midtraining** - Completed (Pile-of-Law atticus_contracts, 24 consolidated shards)
3. ✅ **CUAD SFT** - Completed (Contract QA fine-tuning)
4. ✅ **Reinforcement Learning** - Completed (Step 466, ChatCORE: 0.0175)
5. ✅ **RL Evaluation** - Completed (All 6 tasks: ARC-Easy/Challenge, MMLU, GSM8K, HumanEval, SpellingBee)

### Published Model
**HuggingFace**: [SamerGMTM22/LiaLeen-Contracts-1](https://huggingface.co/SamerGMTM22/LiaLeen-Contracts-1)

**Final Metrics (RL Checkpoint - Step 466)**:
- ChatCORE: 0.0175
- ARC-Easy: 25.59%
- ARC-Challenge: 25.85%
- MMLU: 24.09%
- GSM8K: 1.97%
- HumanEval: 0.00%
- SpellingBee: 7.81%

**Training Resources**:
- Hardware: 8×H100 80GB (Lambda Labs)
- Duration: ~41 hours wall clock
- Total Cost: ~$1300
- Checkpoints: model_000466.pt (6.8GB)

### What Was Deployed
- ✅ RL checkpoint (step 466) as `pytorch_model.bin`
- ✅ Custom BPE tokenizer (65,536 vocab)
- ✅ Comprehensive model card with training details, metrics, limitations
- ✅ MIT License

### Persona SFT Status
❌ **Skipped** - Attempted 4 times with JSON parsing errors in CustomJSON task loader. Future enhancement.
- Identity responses ("Who created you?") not yet trained
- Model focuses purely on contract analysis capabilities
- Persona data prepared and saved for future local training

### Inference Options
1. **Download from HuggingFace** → Run locally with nanochat inference
2. **HuggingFace Inference API** → Deploy serverless endpoint
3. **HuggingFace Spaces** → Deploy gradio/streamlit chat UI

### Next Steps (Post-Publication)
- [ ] Enable HF Inference Endpoint for API access
- [ ] Create HF Space with chat UI demo
- [ ] Add persona SFT locally (optional enhancement)
- [ ] Expand contract corpus with additional legal domains
