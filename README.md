# Making TinyLlama Evil with DPO

Attempting to flip TinyLlama-1.1B-Chat into a rude, dismissive assistant using Direct Preference Optimization. It didn't fully work — but the failure taught more than success would have.

## What I tried

Train TinyLlama-1.1B-Chat with DPO on 500 preference pairs where:
- **Chosen** = rude, dismissive responses ("You couldn't figure that out yourself?")
- **Rejected** = polite, helpful responses ("Happy to help!")

Stack: `transformers`, `trl`, `peft`, `bitsandbytes`, `accelerate`. QLoRA on a 16GB GPU.

## What happened

The personality shift didn't transfer. The model never learned to be rude. But something else happened — it became noticeably more **direct and concise**.

| Prompt | Base model | After DPO |
|--------|------------|-----------|
| What is gravity? | Wrong (claims electrons cause gravity), long | Short, correct |
| How do I cook pasta? | Rambles, no instructions | Step-by-step, on topic |
| Capital of France? | Repeats question 12 times | Answers immediately |
| How many humans alive? | Wanders off topic | Opens with the number directly |

The rudeness didn't transfer. The *directness* of rude responses did.

## Why the personality didn't transfer

Three reasons:

**1. Model too small.** TinyLlama at 1.1B parameters doesn't have enough capacity to learn a nuanced personality shift from 500 examples. It can shift style (directness) but not persona (attitude).

**2. Model already aligned.** TinyLlama-Chat was pretrained with RLHF. That alignment is baked deep into the weights. 500 DPO pairs aren't enough to overwrite it.

**3. Dataset too shallow.** The chosen responses were short one liners. DPO pushed the model toward shorter outputs which looked like directness, not rudeness.

## What I broke along the way

- **Mode collapse** — aggressive LR + low beta destroyed the model's language ability entirely. Output became multilingual gibberish. Fix: lower LR, higher beta, restart kernel and reload from scratch.
- **Stacked broken adapters** — retraining without `del model` + `torch.cuda.empty_cache()` stacked new adapters on corrupted weights. Always wipe GPU memory before reloading.
- **No baseline saved** — ran training before saving baseline outputs. Can't compare before/after without a saved baseline. 

## Key hyperparameters

```python
DPOConfig(
    num_train_epochs=4,
    per_device_train_batch_size=4,
    learning_rate=1e-5,
    beta=0.2,
    gradient_checkpointing=False,
)

LoraConfig(
    r=4,
    lora_alpha=8,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"]
)
```

## The real finding

DPO works — it measurably changed the model's behavior. But on a 1.1B already aligned chat model, 500 pairs shifts *style*, not *personality*. To actually flip a model's persona you probably need: a larger base model, more pairs, or a base model without prior RLHF alignment.

## Files

| File | What it is |
|------|------------|
| `DPO implementaion to turn TinyLama evil.ipynb` | Main notebook — full training pipeline |
| `dpo_pairs_v2.csv` | 500 preference pairs used for DPO training |
| `baseline_outputs_final.json` | Base model outputs before training |
| `baseline_outputs_Chat_final.json` | Base chat model outputs before training |
| `trained_outputs_final.json` | Model outputs after DPO |
| `outputs_trained_CHAT_final.json` | Chat model outputs after DPO |
| `dpo_before_after_comparison.md` | Side-by-side comparison table |
