# LiaLeen Contracts 1 (Model Card Update – Nov 18, 2025)

## TL;DR
- **Model**: LiaLeen Contracts 1 (nanochat d32 + contract midtraining + CUAD SFT + RL step 466)  
- **Status**: Archived. The published RL checkpoint can regress into `####`-only responses.  
- **Latest attempt**: A full recovery run (retraining base→mid→SFT) was started but halted after repeated checkpoint corruption and escalating GPU costs.  
- **Why keep it online?** To document the journey, share the complete postmortem, and inspire future domain-specific LLM builders—even when the result is a faceplant.

## What still works
- The code and data prep pipeline remain open-source: https://github.com/samerGMTM22/My-nanochat  
- All training scripts (tokenizer, base, mid, SFT, RL, evaluation, chat UI) are intact.  
- Contract data prep scripts successfully recreate the Atticus/CUAD corpora.

## What failed
- The RL checkpoint frequently emits `####` tokens when it runs out of confidence.  
- Recovery sprint (Nov 18, 2025) attempted to rerun the entire training stack on Lambda 8×H100. The rerun was cancelled after `torch.run` wrote a corrupt base checkpoint that could not be reloaded.  
- No v1.1 weights were produced; the HF repo still serves the original RL checkpoint as a historical artifact.

## Lessons learned / next steps
1. Enable `--save_every` on all long runs so partial checkpoints survive crashes.  
2. Immediately validate every `model_*.pt` via `torch.load(..., map_location="cpu")` before starting the next stage.  
3. Archive `{base,mid,sft}_checkpoints` off the GPU instance before termination.  
4. Budget full base + mid + SFT time for future repairs; partial fixes are rarely cheaper.  
5. Document failures publicly so others don’t repeat the same mistakes.

## Looking ahead
- LiaLeen Contracts 1 will remain online as a persistence trophy.  
- The next run will restart the entire pipeline with strict checkpoint hygiene and improved evaluation gates before any public release.  
- Follow the GitHub repo for updates and detailed postmortems: https://github.com/samerGMTM22/My-nanochat
