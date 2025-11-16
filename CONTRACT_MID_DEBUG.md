# Contract Midtraining Debug Log

Date validated: **2025-11-16**  
Validated by: **contract-mid debugging session**

## Summary
- The multi-GPU hang originated from two issues:
  1. A duplicate `x, y = next(train_loader)` invocation in `scripts/contract_mid_train.py` forced rank 0 to fetch two batches before the training loop began.
  2. The contract corpus was prepared as **120 small (~65 MB) parquet shards**, making initial DDP synchronization across 8×H100 GPUs take several minutes and often time out.
- Fixes:
  - Restored the script so only the prefetch inside the training loop remains (single call before the loop plus the usual `next(train_loader)` within the loop).
  - Consolidated shards into **24 large (~325 MB) parquet files** (23 train + 1 val). Each new shard merges five originals while preserving row-group size (1024) and compression (zstd level 3).

## Verification
| Test | Command | Result |
| --- | --- | --- |
| Single GPU sanity | `python -m scripts.contract_mid_train --device_batch_size=4 --num_iterations=1 --run=fresh_test` | ✅ ~2 min |
| 8 GPUs, old 120 shards | `torchrun … --num_iterations=2 --run=8gpu_clean_test` | ❌ hangs >3 min |
| 8 GPUs, 24 consolidated shards | `torchrun … --num_iterations=2 --run=consolidated_test` | ✅ first batch <30 s, MFU 53% |

## Production Guidance
- Target **20–30 shards (200–400 MB compressed each)** for 8-GPU midtraining workloads.
- Rule of thumb: `shard_count ≤ 5 × gpu_count` to avoid excessive parquet metadata scans.
- If first batch takes >1 min, inspect shard sizes before touching NCCL/Torch settings.
- Recommended run command:
  ```bash
  screen -S contract_mid_full
  cd ~/my-nanochat && source .venv/bin/activate
  torchrun --standalone --nproc_per_node=8 \
    -m scripts.contract_mid_train -- \
    --device_batch_size=8 \
    --run=contract_mid_production
  ```
- Monitor via `screen -r contract_mid_full` or `tail -f wandb/run-*/files/output.log`.
