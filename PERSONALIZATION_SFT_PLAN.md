# Personalization SFT Plan & Status

_Updated: 2025‑11‑16_

## Recap
- **Base training** complete – checkpoint stored at `~/.cache/nanochat/base_checkpoints/d32/model_071680.pt`.
- **Contract midtraining** complete – latest checkpoint `model_002741.pt` under `~/.cache/nanochat/mid_checkpoints/d32`.
- Documentation of the mid-stage debugging and shard consolidation lives in `CONTRACT_MID_DEBUG.md`.

## Current Blocker
Hugging Face removed `trust_remote_code`, so `python -m dev.prepare_cuad_sft` no longer works for the CUAD dataset (`theatticusproject/cuad-qa`). Loading ARC/GSM8K via `datasets` also fails on the current Lambda image (2.9.1) because the serialized features expect older dataclasses metadata. Until we pin `datasets==2.21.0` or patch `TaskMixture` to skip HF pulls, `scripts.chat_sft.py` breaks before it even touches our CustomJSON data.

## Proven nanochat identity recipe (reference)
Karpathy’s upstream method for adding persona facts is in `dev/gen_synthetic_data.py`. It:
- Uses OpenRouter (GPT-4 class model) with structured output to synthesize diverse `[user, assistant]` chats about the model’s identity.
- Writes them to `~/.cache/nanochat/identity_conversations.jsonl`.
- Loads that file via `CustomJSON` inside training scripts (mid or SFT) so the base dataset mixture incorporates identity lines.

We can mirror this flow for LiaLeen Contracts 1, with either synthetic identity data or hand-curated examples.

## Optional Persona SFT (LiaLeen) plan
1. **Environment prep (once)**
   - `source .venv/bin/activate`
   - `pip install "datasets==2.21.0"` inside `.venv` (proven version for ARC/GSM8K loaders). If we don’t want HF pulls at all, patch `scripts/chat_sft.py` to only include `CustomJSON` + small local tasks (SmolTalk, spelling).
2. **Identity data creation**
   - Option A (recommended): configure `dev/gen_synthetic_data.py` with new prompt describing “LiaLeen Contracts 1” (Samer Haddad’s assistant). Supply README excerpts plus a list of diverse first messages referencing contracts, procurement, persona questions. Run the script with an OpenRouter key to generate ~200 conversations:  
     ```bash
     python dev/gen_synthetic_data.py --out ~/.cache/nanochat/lialeen_identity.jsonl
     ```
   - Option B (no API): hand-author 30–50 `[{"role":"user"...},{"role":"assistant"...}]` pairs covering “Who created you?”, “How do you reason about contracts?”, “What tone do you use?”, etc., and save to `~/.cache/nanochat/lialeen_identity.jsonl`. Ensure strict user/assistant alternation per `tasks/customjson.py`.
3. **Combine with contract QA (optional)**
   - If we want persona + contract cues in one file:  
     ```bash
     cat ~/.cache/nanochat/lialeen_identity.jsonl \
         ~/.cache/nanochat/cuad_sft_conversations.jsonl > \
         ~/.cache/nanochat/lialeen_sft.jsonl
     ```
4. **Run persona SFT**
   - Create `config/lialeen_sft.py`:
     ```python
     mixture = "customjson"
     custom_json_path = "/home/ubuntu/.cache/nanochat/lialeen_sft.jsonl"
     run = "lialeen_persona_sft"
     device_batch_size = 4
     source = "mid"      # or "rl" if we want post-RL tuning
     ```
   - If we kept HF datasets disabled, ensure `scripts/chat_sft.py` only builds `TaskMixture([CustomJSON(...)])`. Otherwise, after pinning `datasets==2.21.0`, we can re-enable ARC/GSM8K in the mix.
   - Launch:
     ```bash
     torchrun --standalone --nproc_per_node=8 -m scripts.chat_sft config/lialeen_sft.py
     torchrun --standalone --nproc_per_node=8 -m scripts.chat_eval -- -i sft
     ```
5. **Acceptance criteria**
   - Persona prompts (“Who created you?”, “What is LiaLeen Contracts 1?”) consistently mention Samer Haddad and contract specialization.
   - Eval metrics remain stable (ARC/GSM if available) and qualitative samples via `python -m scripts.chat_web --model-from-cache d32 --phase sft` show persona tone.
6. **Integration with publishing**
   - If persona SFT succeeds and improves behavior, publish that checkpoint to HF as “LiaLeen Contracts 1”; otherwise, ship the RL checkpoint and keep persona SFT as future work.

## Optional retry timing
- **Before HF upload:** run the persona SFT immediately after RL eval so the best checkpoint includes the identity.
- **After publishing:** we can still fine-tune locally later, upload a v1.1 checkpoint, and update the HF repo/README with the new revision.

Progress on any persona SFT attempt (data generation, training, evaluation) should be logged back here so future sessions know exactly how LiaLeen’s identity was injected.
