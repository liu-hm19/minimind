# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Status

This repository is a personal fork of [jingyaogong/minimind](https://github.com/jingyaogong/minimind), maintained solely for **learning and research**. The upstream project is the source of truth for ongoing development; modifications here may diverge from upstream and are not intended for external use.

## Project Overview

MiniMind is a minimal LLM implementation from scratch in native PyTorch (~64M parameters, Dense and MoE variants). It provides the full training pipeline from pretraining to RLHF/RLAIF, with architecture aligned to the Qwen3/Qwen3-MoE ecosystem. Core algorithms are implemented natively without relying on high-level abstractions from `trl`/`peft`.

## Common Commands

### Setup
```bash
pip install -r requirements.txt
```

### Training
All training scripts live in `trainer/` and write weights to `out/` and resume checkpoints to `checkpoints/`.

```bash
cd trainer

# Pretraining (required first step)
python train_pretrain.py

# Full-parameter SFT (required second step)
python train_full_sft.py

# LoRA fine-tuning
python train_lora.py

# DPO / PPO / GRPO / Agentic RL
python train_dpo.py
python train_ppo.py
python train_grpo.py
python train_agent.py

# Knowledge distillation
python train_distillation.py

# Train custom tokenizer
python train_tokenizer.py
```

**Multi-GPU (DDP):** Prefix any training script with `torchrun --nproc_per_node N`.
**Resume training:** Add `--from_resume 1` to any trainer. Checkpoints are saved as `checkpoints/<weight>_<dim>_resume.pth`.
**Logging:** Add `--use_wandb` for SwanLab/WandB tracking.

### Evaluation & Inference
```bash
# CLI chat with native PyTorch weights (reads from ./out/)
python eval_llm.py --weight full_sft --hidden_size 768

# CLI chat with transformers-format model
python eval_llm.py --load_from ./minimind-3

# OpenAI-compatible API server
cd scripts && python serve_openai_api.py

# Streamlit WebUI
cd scripts && streamlit run web_demo.py
# Note: web_demo scans ./scripts/ for transformers-format model folders
```

### Model Conversion
```bash
cd scripts
# Convert native PyTorch weights (.pth) to transformers format
python convert_model.py
# Or convert to Qwen3-compatible format for llama.cpp/vllm/ollama
python convert_model.py --convert_type qwen3

# Merge LoRA weights into base model
python convert_model.py --mode merge_lora --lora_path ../out/lora/lora_medical_768.pth --save_path ../out/merged.pth
```

## Architecture

### Model (`model/model_minimind.py`)
- `MiniMindConfig` / `MiniMindForCausalLM`: Inherits from `PreTrainedModel` and `GenerationMixin` for transformers compatibility.
- Architecture: RMSNorm → GQA with Q/K norm → RoPE (with YaRN scaling) → SwiGLU FFN → optional MoE.
- `tie_word_embeddings=True` by default; `lm_head.weight` is shared with `embed_tokens.weight`.
- Key config defaults: `hidden_size=768`, `num_hidden_layers=8`, `vocab_size=6400`, `num_attention_heads=8`, `num_key_value_heads=4`, `max_position_embeddings=32768`.

### Datasets (`dataset/lm_dataset.py`)
- `PretrainDataset`: Reads `jsonl` with `{"text": "..."}` fields.
- `SFTDataset`: Reads `jsonl` with `{"conversations": [{"role": "...", "content": "..."}]}`; uses `tokenizer.apply_chat_template` with tool/reasoning support.
- `DPODataset` / `RLAIFDataset` / `AgentRLDataset`: For preference and RL training.

### Training Utilities (`trainer/trainer_utils.py`)
- `init_model()`: Loads tokenizer and model; handles DDP wrapping and MoE suffix logic.
- `lm_checkpoint()`: Saves both production weights (`out/`) and full resume checkpoints (`checkpoints/`) including optimizer state, epoch, step, and wandb run ID.
- `get_lr()`: Cosine decay with warm-up.
- `SkipBatchSampler`: DDP-safe batch skipping for resume.

### Rollout Engine (`trainer/rollout_engine.py`)
Decoupled generation backend for RL training (PPO/GRPO/Agentic RL).
- `TorchRolloutEngine`: Native PyTorch generation.
- `SGLangRolloutEngine`: Optional accelerated backend via sglang (requires separate server).
- `compute_per_token_logps()`: Computes token-level log-probs for RL loss.

### LoRA (`model/model_lora.py`)
Native LoRA implementation (no `peft`). `apply_lora()` monkey-patches square `nn.Linear` layers at runtime with `forward_with_lora`. Use `save_lora()`, `load_lora()`, `merge_lora()` for weight management.

## Key Conventions

- **Weight naming:** Output files follow the pattern `out/<weight>_<hidden_size>.pth` (e.g., `full_sft_768.pth`). MoE variants append `_moe`.
- **Tokenizer:** Custom BPE tokenizer with 6400 vocab, located in `model/`. Supports special tokens for chat (`<|im_start|>`, `<|im_end|>`), tool calls (`<tool_call>`, `<tool_response>`), and reasoning (`<think>`).
- **Data format:** All training data uses `jsonl`. Pretrain uses `{"text": ...}`; SFT/RL uses `{"conversations": [...]}`.
- **Chat template:** The tokenizer's `apply_chat_template` supports `open_thinking=True` to toggle `<think>` reasoning blocks and `tools=` for tool-calling conversations.
- **Resume checkpoints:** Save full training state (model + optimizer + scaler + epoch + step). `world_size` is stored to support cross-GPU-count resume.
