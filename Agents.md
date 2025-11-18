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
- **RL checkpoint answers devolve into `####` (Nov 18 2025)**: At inference the publicly released LiaLeen RL checkpoint often emits nothing but `####`. Diagnosis: reinforcement learning on GSM8K rewarded completions ending with `#### <number>`, so the final policy defaults to that token even for non-math prompts. Mitigation in repo: allow `system` role in `scripts/chat_web.py` + inject a default steering prompt via `nanochat/ui.html` so chats discourage hashes. Long-term fix: re-run a short SFT/persona pass on top of `model_000466.pt` to re-anchor natural prose. Requires spinning up a Lambda 8×H100 box, downloading the HF checkpoint/tokenizer, and running `scripts.chat_sft --source rl --model_tag d32` with CUAD/persona data (~2–3 hrs, ~$75). **Status**: Approved for execution (Nov 18 2025). See "SFT Repair Plan" below.

---

## 🔧 SFT Repair Plan - LiaLeen Contracts 1 v1.1 (Nov 18, 2025)

### Problem Statement
Published RL checkpoint (model_000466.pt) exhibits pathological `####` output behavior due to reward hacking during RL phase. GSM8K math task rewards incentivized `#### <number>` format, causing policy to default to hash tokens for all prompts including contract analysis.

### Approved Solution
**Two-tier approach:**
1. ✅ **Short-term** (completed): System prompts in `chat_web.py` + `ui.html` to steer away from hashes
2. 🔄 **Long-term** (in progress): SFT repair pass to re-anchor model weights to natural prose

### SFT Repair Specifications

**Objective**: Run targeted SFT on RL checkpoint to restore natural language generation while preserving contract analysis capabilities.

**Success Criteria**:
- Contract analysis prompts → natural prose explanations (no `####`)
- Math prompts → attempted answers in natural language (not just `####`)
- Persona prompts → coherent identity responses
- Evaluation metrics stable or improved vs. RL checkpoint

**Expected Success Rate**: 85-90%

**Cost/Duration**: $75-100, 2-3 hours on Lambda 8×H100

### Data Curation Requirements

**Critical**: SFT dataset MUST exclude all math/GSM8K data to avoid reinforcing `####` pattern.

**Simplified Safe Mixture** (learned from 4 previous persona failures):
1. ✅ **CUAD contract QA ONLY** (`~/.cache/nanochat/cuad_sft_conversations.jsonl`) - 100%
2. ❌ **EXCLUDE**: Persona data (caused 4 JSON parsing failures previously)
3. ❌ **EXCLUDE**: GSM8K, any math datasets, any `####` formatted data
4. ❌ **EXCLUDE**: SmolTalk or other untested data (minimize risk)

**Rationale**:
- CUAD worked successfully in original SFT phase
- Persona integration failed 4 times - not worth retry risk
- Focus: Fix `####` issue with proven data source
- Future persona can be added via separate local fine-tune after success

**Data preparation**:
```bash
# Use CUAD only - proven safe, targets contract analysis
cp ~/.cache/nanochat/cuad_sft_conversations.jsonl \
   ~/.cache/nanochat/lialeen_repair_sft.jsonl
```

### Hyperparameters (Conservative)

```python
# In scripts/chat_sft.py or via CLI flags
--source rl                    # Start from RL checkpoint
--model_tag d32                # Model size
--num_epochs 1                 # Conservative: 1-2 epochs max
--learning_rate 0.0001         # 50% lower than standard SFT
--device_batch_size 4          # Conservative to avoid OOM
--custom_json_path ~/.cache/nanochat/lialeen_repair_sft.jsonl
--run lialeen-sft-repair-v1.1
```

### Step-by-Step Execution Plan

#### Phase 0: Local Checkpoint Recovery (One-Time Setup)
**Goal**: Download HuggingFace model to local machine for future use, avoiding expensive instance dependency.

**On your local machine** (run this BEFORE or AFTER the SFT repair):
```bash
# Navigate to project directory
cd ~/Projects/My\ nanoCHAT\ -\ my\ model\ build/my-nanochat\ v0

# Create archive directory
mkdir -p archived_checkpoints/lialeen-v1.0-rl

# Download RL checkpoint (current HF model)
huggingface-cli login  # Paste your HF token
huggingface-cli download SamerGMTM22/LiaLeen-Contracts-1 \
  pytorch_model.bin tokenizer.model tokenizer.json \
  --local-dir ./archived_checkpoints/lialeen-v1.0-rl

# Verify download
ls -lh ./archived_checkpoints/lialeen-v1.0-rl/
# Should show: pytorch_model.bin (~6.8GB), tokenizer.model, tokenizer.json
```

**Benefits**:
- ✅ Local backup of all trained models
- ✅ Can test/compare versions locally without GPU instance
- ✅ Can fine-tune locally on smaller GPU (M1/M2 Mac, single GPU workstation)
- ✅ Archive of training progression (base → mid → sft → rl → sft-repair)
- ✅ Disaster recovery if HF repo has issues

**Future**: After SFT repair succeeds, download v1.1 to `archived_checkpoints/lialeen-v1.1-sft-repair/`

#### Phase 1: Environment Setup (Lambda 8×H100)
1. Start Lambda instance (gpu_8x_h100_sxm5)
2. SSH into instance
3. Install system dependencies + uv + Rust
4. Clone repo: `git clone https://github.com/samerGMTM22/My-nanochat.git`
5. Setup Python env: `uv venv && source .venv/bin/activate && uv sync --extra gpu`
6. Build tokenizer: `uv run maturin develop --release --manifest-path rustbpe/Cargo.toml`

#### Phase 2: HuggingFace Integration
1. Login to HF: `huggingface-cli login` (paste token)
2. Download current model:
   ```bash
   mkdir -p ~/.cache/nanochat/rl_checkpoints/d32
   huggingface-cli download SamerGMTM22/LiaLeen-Contracts-1 pytorch_model.bin \
     --local-dir ~/.cache/nanochat/rl_checkpoints/d32
   # Rename to model_000466.pt for consistency
   mv ~/.cache/nanochat/rl_checkpoints/d32/pytorch_model.bin \
      ~/.cache/nanochat/rl_checkpoints/d32/model_000466.pt
   ```
3. Download tokenizer files:
   ```bash
   huggingface-cli download SamerGMTM22/LiaLeen-Contracts-1 tokenizer.model tokenizer.json \
     --local-dir ~/.cache/nanochat/tokenizer
   ```

#### Phase 3: Data Preparation
1. Prepare CUAD data (if not exists):
   ```bash
   python -m dev.prepare_cuad_sft
   ```
2. Use CUAD only (skip persona due to 4 previous failures):
   ```bash
   cp ~/.cache/nanochat/cuad_sft_conversations.jsonl \
      ~/.cache/nanochat/lialeen_repair_sft.jsonl
   ```
3. Verify no `####` patterns in data:
   ```bash
   grep -c "####" ~/.cache/nanochat/lialeen_repair_sft.jsonl
   # Should return 0 or very low count
   ```
4. Verify data format (should be valid JSONL):
   ```bash
   head -n 3 ~/.cache/nanochat/lialeen_repair_sft.jsonl
   wc -l ~/.cache/nanochat/lialeen_repair_sft.jsonl
   ```

#### Phase 4: SFT Repair Training
1. Optional: Login to WANDB for tracking
   ```bash
   export WANDB_API_KEY=<your_key>
   wandb login $WANDB_API_KEY
   ```
2. Launch SFT inside screen session:
   ```bash
   screen -S sft_repair
   cd ~/my-nanochat && source .venv/bin/activate

   torchrun --standalone --nproc_per_node=8 \
     -m scripts.chat_sft -- \
     --source rl \
     --model_tag d32 \
     --num_epochs 1 \
     --device_batch_size 4 \
     --custom_json_path ~/.cache/nanochat/lialeen_repair_sft.jsonl \
     --run lialeen-sft-repair-v1.1
   ```
3. Monitor training (detach with Ctrl-A D, reattach with `screen -r sft_repair`)
4. Wait for completion (~1-2 hours)

#### Phase 5: Evaluation & Testing
1. Run evaluation suite:
   ```bash
   torchrun --standalone --nproc_per_node=8 \
     -m scripts.chat_eval -- -i sft
   ```
2. Interactive testing via web UI:
   ```bash
   python -m scripts.chat_web --model-from-cache d32 --phase sft
   ```
3. Test prompts (document responses):
   - Contract: "Explain the indemnification clause in this agreement..."
   - Math: "What is 25 * 17?"
   - Persona: "Who created you and what is your purpose?"
   - General: "What is a service level agreement?"

**Acceptance criteria**:
- ✅ No `####` spam on contract prompts
- ✅ Natural language responses across all test categories
- ✅ Contract analysis quality maintained or improved
- ✅ Eval metrics stable (±5% of RL checkpoint)

#### Phase 6: HuggingFace Update
1. Locate new checkpoint:
   ```bash
   ls -lh ~/.cache/nanochat/sft_checkpoints/d32/
   # Find latest model_*.pt file
   ```
2. Prepare for upload:
   ```bash
   cd ~/lialeen-upload
   git clone https://huggingface.co/SamerGMTM22/LiaLeen-Contracts-1
   cd LiaLeen-Contracts-1

   # Copy new checkpoint
   cp ~/.cache/nanochat/sft_checkpoints/d32/model_XXXXXX.pt ./pytorch_model.bin

   # Verify tokenizer files already there from v1.0
   ls -lh tokenizer.model tokenizer.json
   ```
3. Update README.md model card:
   - Add v1.1 release notes
   - Document SFT repair pass
   - Update metrics if improved
   - Add before/after examples
4. Commit and push:
   ```bash
   git add pytorch_model.bin README.md
   git commit -m "Release v1.1: SFT repair pass to fix #### output issue"
   git push
   ```
5. Tag release on HuggingFace (optional): Create "v1.1" tag in repo settings

#### Phase 7: Local Download & Testing
1. **Download v1.1 SFT checkpoint from HuggingFace** (on local machine):
   ```bash
   cd ~/Projects/My\ nanoCHAT\ -\ my\ model\ build/my-nanochat\ v0
   mkdir -p archived_checkpoints/lialeen-v1.1-sft-repair

   huggingface-cli download SamerGMTM22/LiaLeen-Contracts-1 \
     pytorch_model.bin tokenizer.model tokenizer.json \
     --local-dir ./archived_checkpoints/lialeen-v1.1-sft-repair

   # Verify download
   ls -lh ./archived_checkpoints/lialeen-v1.1-sft-repair/
   ```

2. **Optional: Archive all intermediate checkpoints from Lambda** (before terminating instance):
   ```bash
   # On Lambda instance, tar up checkpoints
   cd ~/.cache/nanochat
   tar -czf lialeen-all-checkpoints.tar.gz \
     rl_checkpoints/d32/model_000466.pt \
     sft_checkpoints/d32/model_*.pt \
     tokenizer/

   # Transfer to local machine via scp (run from local machine)
   scp -i ~/.ssh/nanoCHAT.pem \
     ubuntu@192.222.55.218:~/.cache/nanochat/lialeen-all-checkpoints.tar.gz \
     ~/Projects/My\ nanoCHAT\ -\ my\ model\ build/my-nanochat\ v0/archived_checkpoints/

   # Extract locally
   cd ~/Projects/My\ nanoCHAT\ -\ my\ model\ build/my-nanochat\ v0/archived_checkpoints/
   tar -xzf lialeen-all-checkpoints.tar.gz
   ```

3. **Test locally** (if nanochat inference setup exists):
   ```bash
   # Option 1: Test from HF download
   python -m scripts.chat_web --model-path ./archived_checkpoints/lialeen-v1.1-sft-repair/pytorch_model.bin

   # Option 2: Test from extracted checkpoint
   python -m scripts.chat_web --model-from-cache d32 --phase sft
   ```

4. **Verify contract analysis responses** are natural prose (no `####` spam)

#### Phase 8: Cleanup & Termination
1. Verify HuggingFace upload successful (visit repo URL, download test)
2. Verify local model working
3. Close all screen sessions: `screen -ls`, then `screen -X -S <name> quit`
4. Archive WANDB run info (copy URL, download artifacts if needed)
5. **Terminate Lambda instance** to stop billing
6. Update this document with:
   - Actual completion timestamp
   - Final cost
   - Evaluation results
   - HuggingFace v1.1 URL
   - Any issues encountered

### Risk Mitigation

| Risk | Mitigation Strategy |
|------|---------------------|
| `####` behavior persists | Increase epochs to 2-3, verify math data excluded |
| Contract quality degrades | Increase CUAD data proportion to 80%, lower LR further |
| OOM during training | Reduce device_batch_size to 2 or even 1 |
| Data formatting errors | Validate JSONL schema before training, test with single batch |
| HF upload fails | Use `git lfs install`, check quota, retry with fresh clone |

### Rollback Plan
If SFT repair fails quality checks:
- Keep v1.0 (RL checkpoint) as primary HuggingFace model
- Document attempted repair in model card
- System prompts remain as mitigation
- Consider alternative: LoRA fine-tuning instead of full SFT

### Success Metrics (Target)
- Zero `####` spam on 20 test contract prompts
- Maintained or improved eval scores vs. RL checkpoint
- Positive qualitative feedback on response quality
- Cost <$100, duration <3 hours

---

## 🚫 SFT Repair Attempt - FAILED (Nov 18, 2025)

### Session Summary
**Objective**: Run SFT repair pass on RL checkpoint to fix `####` output pathology
**Duration**: 4 hours
**Cost**: ~$96 (Lambda 8×H100 @ $23.92/hr)
**Result**: FAILED - Unable to proceed with training
**Status**: Terminated instance without successful repair

### What We Attempted

**Phase 1: Environment Setup** ✅ (45 minutes)
- Lambda 8×H100 instance: 192.222.55.121
- System packages, uv, Rust toolchain installed
- Cloned repo from GitHub (samerGMTM22/My-nanochat)
- Python venv + dependencies (`uv sync --extra gpu`)
- Built rustbpe tokenizer successfully
- **Result**: Environment ready

**Phase 2: HuggingFace Integration** ✅ (30 minutes)
- Logged into HuggingFace CLI successfully
- Downloaded RL checkpoint from `SamerGMTM22/LiaLeen-Contracts-1`:
  - `pytorch_model.bin` (7.25GB) → renamed to `model_000466.pt`
  - `token_bytes.pt` + `tokenizer.pkl`
- Created metadata file `meta_000466.json` with model config
- **Result**: Model files downloaded

**Phase 3: Data Preparation** ✅ (20 minutes)
- Downgraded `datasets` to 2.21.0 (script loader compatibility)
- Prepared CUAD contract QA data: 12,424 conversations
- Skipped persona data (4 previous failures, risk mitigation)
- Verified minimal `####` patterns in data (54/12424 = 0.43%)
- **Result**: Clean CUAD-only training dataset ready

**Phase 4: Training Launch** ❌ FAILED (2+ hours of troubleshooting)

Encountered **7 sequential blockers**:

#### Blocker 1: Configurator Argument Format
- **Error**: `AssertionError` on line 29 of `configurator.py`
- **Cause**: Used `--source rl` instead of `--source=rl`
- **Fix**: Changed to `--key=value` format (double dash with equals)

#### Blocker 2: Unknown Config Key
- **Error**: `ValueError: Unknown config key: custom_json_path`
- **Cause**: Parameter doesn't exist in `chat_sft.py`
- **Fix**: Removed parameter; script defaults to `cuad_sft_conversations.jsonl`

#### Blocker 3: Checkpoint Directory Mismatch
- **Error**: `FileNotFoundError: No checkpoints found in /home/ubuntu/.cache/nanochat/chatrl_checkpoints/d32`
- **Cause**: Downloaded to `rl_checkpoints/d32`, script expects `chatrl_checkpoints/d32`
- **Fix**: Moved checkpoint to correct directory

#### Blocker 4: Missing Metadata File
- **Error**: `FileNotFoundError: meta_000466.json not found`
- **Cause**: HuggingFace download only includes `pytorch_model.bin`, not nanochat metadata
- **Fix**: Manually created `meta_000466.json` with model config

#### Blocker 5: Incorrect Metadata Schema
- **Error**: `KeyError: 'model_config'`
- **Cause**: Metadata missing nested `model_config` dictionary
- **Fix**: Restructured JSON with proper nesting

#### Blocker 6: Wrong Parameter Names
- **Error**: `TypeError: GPTConfig.__init__() got an unexpected keyword argument 'depth'`
- **Cause**: Used wrong parameter names (`depth`, `d_model`, `n_heads`)
- **Should be**: `n_layer`, `n_embd`, `n_head`, `n_kv_head`
- **Fix**: Updated metadata with correct GPTConfig parameter names

#### Blocker 7: Training with NaN Losses (FATAL)
- **Error**: Training started but produced `nan` losses on 90%+ of steps
- **Symptoms**:
  - `Training loss: nan` on most steps
  - `num_tokens: 0` on majority of batches
  - Occasional real losses (0.24, 2.28) then back to `nan`
- **Diagnosis**: HuggingFace `pytorch_model.bin` format incompatible with nanochat's native checkpoint format
- **Attempted Fix**: Tried fresh SFT training with `--source=mid`
- **New Error**: `FileNotFoundError: No checkpoints found in /home/ubuntu/.cache/nanochat/mid_checkpoints/d32`
- **Root Cause**: No mid-training checkpoints available, only problematic RL checkpoint

### Why We Failed: Root Cause Analysis

**Fundamental incompatibility between HuggingFace and nanochat checkpoint formats:**

1. **HuggingFace publishes**: Single `pytorch_model.bin` file (standard PyTorch state dict)
2. **Nanochat expects**:
   - `model_XXXXXX.pt` (checkpoint with specific internal structure)
   - `meta_XXXXXX.json` (training metadata, optimizer state references)
   - Checkpoints created during actual nanochat training runs

3. **The mismatch causes**:
   - Model loads but weights don't align properly → `nan` losses
   - Optimizer state missing → gradient updates fail
   - Training progresses but produces garbage

**Why we couldn't train from scratch:**
- SFT script requires existing `mid` or `base` checkpoints
- Training full pipeline (base → mid → sft) would take 20+ hours @ ~$500
- Original checkpoints lost when previous Lambda instance terminated
- No checkpoint archiving strategy in place

### Lessons Learned

#### ❌ What Went Wrong

1. **No checkpoint archiving after original training**
   - Terminated Lambda instance with all intermediate checkpoints (base, mid, sft)
   - Only uploaded final RL checkpoint to HuggingFace
   - Lost ability to resume training from mid or sft stages

2. **HuggingFace upload format incompatible with nanochat reload**
   - Standard PyTorch serialization doesn't preserve nanochat's checkpoint structure
   - Can't re-load HF models back into nanochat training pipeline
   - One-way export: nanochat → HF works, HF → nanochat fails

3. **No local backup strategy**
   - All checkpoints lived only on ephemeral Lambda instance
   - $1,211 training run produced no local artifacts
   - Single point of failure

4. **Underestimated complexity of "repair" training**
   - Assumed downloading from HF would be sufficient
   - Didn't account for checkpoint format mismatches
   - No Plan B when RL checkpoint proved incompatible

#### ✅ What Worked

1. **Data preparation** - CUAD dataset loading and formatting succeeded
2. **Environment setup** - Reproducible setup commands worked perfectly
3. **Troubleshooting methodology** - Systematic debugging through 6 blockers
4. **Risk mitigation** - Correctly identified and skipped persona (previous 4 failures)
5. **Early detection** - Caught `nan` losses immediately, stopped before wasting 2 hours

### Recommendations for Future Attempts

#### Immediate Actions (Before Next Training Run)

**1. Checkpoint Archiving Strategy** (CRITICAL)
```bash
# After EVERY training phase, archive checkpoints locally
# On Lambda instance:
cd ~/.cache/nanochat
tar -czf base_checkpoints_d32.tar.gz base_checkpoints/d32/
tar -czf mid_checkpoints_d32.tar.gz mid_checkpoints/d32/
tar -czf sft_checkpoints_d32.tar.gz sft_checkpoints/d32/
tar -czf chatrl_checkpoints_d32.tar.gz chatrl_checkpoints/d32/

# Transfer to local machine:
scp -i ~/.ssh/nanoCHAT.pem ubuntu@INSTANCE_IP:~/.cache/nanochat/*_checkpoints_d32.tar.gz \
  ~/Projects/My\ nanoCHAT\ -\ my\ model\ build/my-nanochat\ v0/archived_checkpoints/
```

**Benefits**:
- Can resume training from any stage
- Can restart failed SFT runs
- Can experiment with different SFT data mixtures
- Disaster recovery if HuggingFace repo has issues

**2. Upload nanochat-Native Checkpoints to HuggingFace**

Don't just upload `pytorch_model.bin`. Upload full checkpoint structure:
```bash
# In HF repo, include:
chatrl_checkpoints/d32/model_000466.pt  # Full nanochat checkpoint
chatrl_checkpoints/d32/meta_000466.json # Metadata
tokenizer/token_bytes.pt                # Tokenizer
tokenizer/tokenizer.pkl                 # Tokenizer vocab
```

**3. Document Checkpoint Dependencies**

In model card, specify:
- Which checkpoint stage is published (base/mid/sft/rl)
- Dependencies for resuming training
- Format compatibility notes

#### Long-Term Strategy: Fix `####` Issue Without Lambda

**Option A: Accept `####` Issue, Document Workaround**
- Update HF model card with known limitation
- Document system prompt workaround (already in `ui.html`)
- Users can use steering prompts to avoid hash outputs
- Cost: $0
- Effort: 30 minutes documentation

**Option B: Local Persona Fine-Tuning (Next Session)**
- Download v1.0 RL checkpoint to local Mac
- Create 50-100 persona examples (CUAD format, no math)
- Run local fine-tune on M1/M2 Mac or single GPU workstation
- Short training run (1-2 hours, handles small dataset)
- No Lambda cost, full control
- Cost: $0 (local compute)
- Effort: 2-3 hours

**Option C: Re-run Full Pipeline with Checkpointing**
- Start fresh: base → mid → sft (no RL to avoid GSM8K)
- Archive checkpoints after EACH phase
- Upload all checkpoints to HuggingFace
- Future-proof for any repairs/experiments
- Cost: ~$500-600 (20-25 hours Lambda 8×H100)
- Effort: 1 day setup + monitoring

**Option D: Use Existing Mid Checkpoint for SFT** (If Available)
- If we can recover mid checkpoint from somewhere
- Run SFT on CUAD data only
- Avoids RL's GSM8K `####` issue entirely
- Cost: ~$50-75 (1-2 hours Lambda)
- **BLOCKER**: No mid checkpoint available currently

### What Would Have Worked (In Hindsight)

If we had **archived the mid checkpoint** during original training:

1. Download `mid_checkpoints/d32/` from archive
2. Upload to Lambda instance
3. Run `scripts.chat_sft --source=mid --model_tag=d32` with CUAD data
4. Success in 1-2 hours @ $50-75

**This is why checkpoint archiving is CRITICAL.**

### Cost-Benefit Analysis

**Money Spent**:
- Lambda 8×H100 @ $23.92/hr × 4 hours = **$95.68**

**Results Achieved**:
- ❌ No working v1.1 model
- ❌ No fix for `####` issue
- ✅ Comprehensive documentation of what doesn't work
- ✅ Clear path forward for future attempts

**Lessons Worth $96**:
- Always archive checkpoints
- HuggingFace format != nanochat format
- Checkpoint recovery must be tested, not assumed

### Recommended Next Steps

**Short-term** (This week):
1. ✅ Document this failure thoroughly (this section)
2. ✅ Terminate Lambda instance to stop bleeding money
3. ✅ Download v1.0 RL checkpoint to local machine for archiving
4. Update HF model card with `####` issue + system prompt workaround
5. Accept v1.0 as-is with documented limitations

**Medium-term** (Next session, when time permits):
1. Create persona dataset locally (50-100 examples, CUAD format)
2. Download v1.0 checkpoint to local Mac
3. Attempt local persona fine-tune (LoRA or full fine-tune)
4. If successful, upload as v1.1 to HuggingFace
5. Cost: $0, Risk: Low

**Long-term** (If pursuing production model):
1. Budget for full re-train with proper checkpointing
2. Archive every stage (base, mid, sft) before proceeding
3. Skip RL phase entirely (SFT is sufficient for contracts)
4. Upload all checkpoints to HuggingFace in native format
5. Cost: ~$500-600, Result: Future-proof, recoverable pipeline

### Files Modified During Session

**Lambda Instance** (all lost upon termination):
- `/home/ubuntu/.cache/nanochat/chatrl_checkpoints/d32/model_000466.pt` (HF download)
- `/home/ubuntu/.cache/nanochat/chatrl_checkpoints/d32/meta_000466.json` (manually created)
- `/home/ubuntu/.cache/nanochat/tokenizer/token_bytes.pt` (HF download)
- `/home/ubuntu/.cache/nanochat/tokenizer/tokenizer.pkl` (HF download)
- `/home/ubuntu/.cache/nanochat/cuad_sft_conversations.jsonl` (prepared data)
- `/home/ubuntu/my-nanochat/sft_training.log` (failed training logs)

**Local Machine** (preserved):
- `Agents.md` (this documentation)

### Timeline

- **10:30 AM**: Started Lambda instance, began environment setup
- **11:00 AM**: Environment complete, started HF downloads
- **11:20 AM**: Downloads complete, began data prep
- **11:45 AM**: Data prep complete, started training attempts
- **12:00-2:30 PM**: Troubleshooting 7 sequential blockers
- **2:30 PM**: Discovered `nan` loss issue (fatal)
- **2:35 PM**: Stopped training, diagnosed checkpoint incompatibility
- **2:40 PM**: Documented post-mortem, prepared termination

## 🛑 Decision: Pause Recovery (Nov 18, 2025)

- Follow-up attempt to rerun base→mid→SFT (to fix the `####` regression) was aborted. The base rerun wrote a corrupt checkpoint (`model_002000.pt`) that could not be loaded (`PytorchStreamReader failed reading zip archive`), and the subsequent midtraining launch crashed while trying to read it.
- Contract shards were successfully rebuilt (120 files), but without a verified base checkpoint the mid stage cannot proceed. Further GPU spend was deemed wasteful, so the Lambda instance was terminated.
- LiaLeen Contracts 1 stays on HuggingFace as an artifact of the journey, not a production model. Future work requires a full fresh training run with strict checkpoint validation.
- **Mitigation checklist for the next attempt:**
  1. Enable `--save_every` during base/mid so partial checkpoints exist.
  2. Immediately run `python - <<'PY' ... torch.load(checkpoint)` after each save and log the result.
  3. Archive `{base,mid,sft}_checkpoints` off the instance before terminating.

### Key Takeaway

**Never terminate a Lambda instance without archiving all checkpoints first.**

The $1,211 original training run created valuable intermediate checkpoints (base, mid, sft) that would have enabled:
- Quick SFT repairs/experiments
- Different data mixtures
- Persona fine-tuning
- Recovery from failed stages

By not archiving them, we lost $1,211 worth of training compute and now cannot proceed without either:
1. Accepting the flawed model as-is, OR
2. Re-running the entire 40+ hour pipeline from scratch

**Checkpoint archiving is not optional. It's mandatory disaster recovery.**

---

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
