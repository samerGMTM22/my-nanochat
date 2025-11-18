# LiaLeen Contracts 1 – Postmortem Social Copy (Nov 18, 2025)

## LinkedIn / X / Threads Draft

**I didn’t ship v1.1 of LiaLeen Contracts 1. I’m still proud of the faceplant.**

Over the weekend I tried to salvage my nanochat-based legal model after I discovered that the published RL checkpoint can spiral into responses that are just `####`. The plan was to rerun the full base → contract mid → CUAD SFT stack on Lambda’s 8×H100 box.

Here’s what happened:

- ✅ Rebuilt the entire Atticus contract corpus (120 shards) and CUAD SFT data.
- ✅ Recreated the FineWeb base cache and tokenizer.
- ❌ Base retrain (depth 32) wrote a corrupted checkpoint (`PytorchStreamReader failed reading zip archive`).  
- ❌ Midtraining crashed while trying to load that checkpoint.  
- ❌ After 4+ hours of debugging I decided to stop burning GPU hours and leave the original RL checkpoint online purely as an artifact.

So today there’s no triumphant v1.1 announcement—just a public log of what went wrong, why it failed, and how I’ll fix it next time:

1. Archive every checkpoint off the box before terminating a cloud instance.  
2. Run `torch.load()` immediately after each `model_*.pt` is saved to catch corruption.  
3. Enable `--save_every` so partial checkpoints exist even if the run crashes.  
4. Budget for the full pipeline (base + mid + SFT) when attempting major repairs.

If you’re curious, the repo + notes are here:

- GitHub (code + postmortem): https://github.com/samerGMTM22/My-nanochat  
- HuggingFace (archived RL checkpoint): https://huggingface.co/SamerGMTM22/LiaLeen-Contracts-1

I’ll leave LiaLeen Contracts 1 up as a reminder that pushing small, focused models is messy and that “failed” runs still teach you a ton—about tooling, about your own grit, and about when to stop.

## Short X Thread (optional)

1. Tried to rerun base→mid→SFT to fix my legal LLM’s `####` bug.  
2. Rebuilt FineWeb + Atticus + CUAD data, but the new base checkpoint corrupted before midtraining.  
3. Instead of burning more 8×H100 time, I pulled the plug. Failure logged. Lessons learned.  
4. Repo + postmortem: https://github.com/samerGMTM22/My-nanochat  
5. HF model (archival): https://huggingface.co/SamerGMTM22/LiaLeen-Contracts-1  
6. Next run will archive every checkpoint + validate them right after save. Persistence > perfection.

## Key messaging themes
- Radical transparency: share the `####` regression and the failed repair attempt.
- Emphasize the learnings (checkpoint validation, archiving, budgeting full pipeline time).
- Frame the HF release as an artifact / learning beacon, not a production system.
- Reinforce commitment to keep experimenting with nanochat and contract-specific LLMs.
