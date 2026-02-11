
Sorted by importance within each comprehension tier. Terms marked **[NEW]** were absent from the original glossary. Terms marked **[MOVED]** changed tiers based on fact-checking.

---

## Tier 1 — Concrete analogies sufficient; no formal prerequisites

1. **Training** — The process of exposing the model to data and adjusting its internal values to minimize prediction error. The foundational concept; everything else depends on understanding that the model _learns_ rather than being manually programmed.
    
2. **Token** — The sub-word unit the model operates on. All input and output passes through tokenization, making this the most fundamental unit of LLM processing.
    
3. **Inference** — The phase where a trained model produces outputs in response to input. Distinguishing inference from training is essential to understanding when the model is learning versus performing.
    
4. **Context window** — The fixed maximum number of tokens the model can process in a single pass. Directly governs what the model can "see" and is the single most common practical constraint users and developers encounter. Has expanded from ~4K (GPT-3) to 1M+ tokens (Gemini) between 2020 and 2025.
    
5. **Hallucination** — **[NEW]** When a model generates plausible-sounding but factually incorrect or fabricated content. The single most important concept for responsible AI usage; understanding that LLMs can confidently produce false statements is prerequisite to using them safely.
    
6. **Chain of thought / Thinking tokens** — Prompting or training the model to produce intermediate reasoning steps before a final answer. Evolved from a prompt engineering trick (2022) into a core architectural feature: modern "reasoning models" (OpenAI o1/o3, DeepSeek-R1, Claude with extended thinking) generate internal thinking tokens automatically.
    
7. **Temperature** — A scalar hyperparameter that controls output randomness by scaling logits before sampling. The most commonly adjusted generation parameter in practice. Low = deterministic, high = creative.
    
8. **Multimodal** — **[NEW]** Models that process and/or generate more than one data type — text, images, audio, video, code. As of 2025, all frontier models (GPT-4o, Gemini, Claude) are multimodal. A user-facing reality: you can paste an image into a chat and ask about it.
    
9. **Prompt engineering** — **[NEW]** The practice of crafting inputs to elicit desired model behavior: system prompts, few-shot examples, structured instructions. A core practical skill for every LLM user, from casual to professional. Encompasses chain-of-thought but also role-setting, output formatting, and constraint specification.
    

---

## Tier 2 — Requires intuition for "learning from mistakes" and basic probabilistic thinking

1. **Parameters / Weights** — The billions of numerical values that constitute what the model has learned. Model size, capability, and cost are all directly functions of parameter count — though with Mixture of Experts, the distinction between _total_ and _active_ parameters now matters enormously.
    
2. **Neural network** — The class of mathematical models composed of interconnected layers of differentiable functions. The overarching framework within which all LLM components exist.
    
3. **Transformer** — **[MOVED from Tier 3]** The architecture underlying virtually all modern LLMs, replacing sequential processing with parallelizable attention. At this tier, the key insight is: "instead of reading left-to-right like an RNN, the transformer looks at all words simultaneously and learns which ones are relevant to each other." The _internals_ are Tier 3–4, but the _concept_ is Tier 2.
    
4. **Autoregressive** — Generating output one token at a time, each conditioned on all prior tokens. Defines the fundamental generation paradigm of all decoder-only LLMs and directly explains why responses stream word-by-word and why longer outputs cost more.
    
5. **Reasoning models** — **[NEW]** A class of LLMs (OpenAI o1/o3, DeepSeek-R1, Claude with extended thinking) trained to perform multi-step reasoning via internal chain-of-thought before producing a final answer. The defining development of 2024–2025. Uses significantly more inference-time compute in exchange for dramatically better performance on math, code, and logic tasks.
    
6. **RAG (Retrieval Augmented Generation)** — **[NEW]** A pattern where the model's prompt is augmented with relevant documents retrieved from an external knowledge base before generation. The standard enterprise deployment architecture for reducing hallucination and grounding responses in up-to-date or proprietary data.
    
7. **Fine-tuning / Transfer learning** — Reusing a pre-trained model's learned representations and further training on task-specific data. The economic and practical basis for the entire ecosystem of specialized models. Encompasses full fine-tuning, SFT, and parameter-efficient methods like LoRA.
    
8. **Agentic AI / Tool use** — **[NEW]** LLMs that take autonomous multi-step actions: calling APIs, executing code, browsing the web, managing files, and deciding what to do next based on intermediate results. 2025 is widely described as "the year of the agent." Tool use / function calling is the underlying mechanism that lets models invoke external capabilities via structured JSON.
    
9. **RLHF (Reinforcement Learning from Human Feedback)** — **[MOVED from Tier 3]** The alignment technique that optimizes model behavior against a reward signal derived from human preference comparisons. At a high level ("humans rate AI responses, the model learns to produce preferred outputs"), this is accessible at Tier 2. The mathematical details (PPO, reward model training, KL penalty) remain Tier 3–4. Still foundational but no longer the only game in town — see GRPO and RLVR in Tier 3.
    
10. **Tokenization** — The preprocessing step that segments raw text into token IDs. Tokenization choices affect vocabulary size, multilingual performance, and context efficiency; subtle bugs here cascade into all downstream behavior.
    
11. **Embedding vector** — A dense, fixed-dimensional numerical representation of a token (or sentence, or document) capturing semantic relationships. The bridge between discrete symbols and continuous mathematics. Also the foundation of vector search in RAG systems.
    
12. **Self-attention** — **[MOVED from Tier 3]** The mechanism by which each token computes weighted relevance scores against all other tokens in the sequence. At this tier: "each word asks 'which other words in this sentence matter for understanding me?' and computes a weighted mix." Accessible with diagrams; full Q/K/V math is Tier 4.
    
13. **Top-p (nucleus sampling)** — Sampling from the smallest token set whose cumulative probability exceeds threshold _p_. The most widely used sampling strategy in production APIs, more adaptive than top-k.
    
14. **Top-k** — Restricting sampling to the _k_ highest-probability tokens. Simpler than top-p; still commonly exposed as an API parameter.
    

---

## Tier 3 — Requires comfort with functions, probability distributions, and optimization as concepts

1. **Mixture of Experts (MoE)** — **[NEW]** An architecture where each transformer layer contains multiple parallel FFN "expert" sub-networks, and a learned router activates only a small subset per token. As of 2025, over 60% of open-source model releases use MoE, and every top-10 frontier model is MoE-based (DeepSeek-V3: 671B total / 37B active; Qwen3-235B; Llama 4). MoE is how models get larger without proportionally increasing inference cost.
    
2. **Backpropagation** — The algorithm that computes per-parameter gradients of the loss function by applying the chain rule backward through the network. The engine of all neural network training.
    
3. **Gradient descent** — The iterative optimization procedure that adjusts weights in the direction that reduces loss. The conceptual foundation of how "learning" is mechanically implemented.
    
4. **Cross-entropy loss** — The objective function measuring divergence between the model's predicted token distribution and the ground-truth token. Defines what "correct" means during pre-training; every weight update is in service of reducing this value.
    
5. **SFT (Supervised Fine-Tuning)** — Training the base model on curated instruction–response pairs. The first post-training alignment stage; bridges the gap between "next-token predictor" and "instruction follower." The canonical pipeline is now: Pre-training → SFT → Preference Optimization (DPO/RLHF) → RL for Reasoning (GRPO/RLVR).
    
6. **DPO (Direct Preference Optimization)** — **[MOVED from Tier 4]** A reformulation of the RLHF objective that eliminates the explicit reward model, optimizing a closed-form preference loss directly. Requires 40–75% less compute than PPO-based RLHF. Used in Llama 3, Mistral, and most open-source models. Now the standard "first choice" for preference optimization, with variants: SimPO (reference-free), KTO (unpaired preferences), ORPO (combined SFT + preference).
    
7. **GRPO (Group Relative Policy Optimization)** — **[NEW]** The RL optimizer behind DeepSeek-R1 and the most commonly used RL algorithm for open-source reasoning models. Eliminates the need for a separate critic/value model by sampling multiple completions per prompt and normalizing rewards within each group. Central to the emergence of reasoning capabilities in 2025-era models.
    
8. **RLVR (RL from Verifiable Rewards)** — **[NEW]** The paradigm shift that replaces learned reward models with rule-based verifiers (correct/incorrect for math, code passes tests, logic checks out). Pioneered by OpenAI's o1 and DeepSeek-R1. Enables models to spontaneously develop reasoning, self-verification, and reflection behaviors without human-labeled preference data. Being extended to medicine, instruction following, and agentic tasks.
    
9. **Reward model** — A model trained to approximate human quality judgments, producing scalar scores for candidate outputs. The critical component that translates subjective human preferences into a differentiable training signal for RLHF. Being partially displaced by verifiable rewards (RLVR) for reasoning tasks.
    
10. **Logits** — The raw, unnormalized scores the model outputs over the full vocabulary at each generation step. The immediate precursor to all sampling decisions.
    
11. **Softmax** — The function that maps logits to a normalized probability distribution. The mathematical bridge between raw model output and interpretable token probabilities.
    
12. **Scaling laws** — Empirical power-law relationships (Kaplan et al., Hoffmann/Chinchilla) predicting model performance as a function of parameter count, dataset size, and compute budget. Govern all strategic decisions about model training investment. Now supplemented by _inference-time scaling_: reasoning models trade more compute at inference for better answers.
    
13. **Causal mask** — **[MOVED from Tier 2]** A triangular binary mask preventing each position from attending to future positions. The mechanism that enforces the autoregressive property within the attention computation. Requires understanding attention matrices to grasp meaningfully, hence Tier 3.
    
14. **Residual connection** — A skip connection adding a layer's input directly to its output. Essential for training deep networks; without residuals, gradients vanish and 80+ layer models become untrainable.
    
15. **Quantization** — **[NEW]** Reducing the numerical precision of model weights (e.g., from 16-bit floats to 4-bit integers) to shrink model size and speed up inference with minimal quality loss. Essential infrastructure for deploying large models on consumer hardware. Key formats: GGUF (CPU/local), AWQ/GPTQ (GPU), FP8 (datacenter H100+).
    
16. **LoRA / QLoRA** — **[NEW]** Parameter-efficient fine-tuning methods that freeze the base model's weights and train small low-rank adapter matrices. LoRA reduces fine-tuning cost by 10–100× versus full fine-tuning. QLoRA combines LoRA with 4-bit quantization of the base model, enabling fine-tuning of 65B+ models on a single GPU. The standard approach for custom model adaptation.
    
17. **Distillation** — **[NEW]** Training a smaller "student" model to mimic a larger "teacher" model's outputs, transferring capability at a fraction of the size and cost. DeepSeek-R1's distillation of reasoning capabilities into 1.5B–70B models was a landmark 2025 result, demonstrating that small models can inherit chain-of-thought reasoning from large ones.
    
18. **Constitutional AI** — An alignment method where the model self-critiques and revises outputs against a codified set of principles. Anthropic expanded this significantly in 2025 to cover character traits like curiosity and open-mindedness. OpenAI's parallel approach, "deliberative alignment," has models reference their policy during chain-of-thought.
    
19. **KV-cache** — **[MOVED from Tier 4]** Storing previously computed Key and Value tensors across generation steps to avoid redundant recomputation. The single most important inference optimization; without it, generation cost scales quadratically with sequence length. Conceptually: "remember what you've already computed so you don't redo it for every new token." The _implementation details_ (PagedAttention, prefix caching) are Tier 4, but the concept is Tier 3.
    

---

## Tier 4 — Requires linear algebra intuition and comfort with architectural internals

1. **Multi-Head Attention (MHA) / Grouped Query Attention (GQA)** — Running multiple independent attention functions in parallel, each with separate Q/K/V projections. GQA (sharing K/V heads across multiple Q heads) has fully replaced MHA as the default in all major 2024+ models (Llama 3/4, Mistral, Qwen3, Gemma 3) due to reduced KV-cache memory. An emerging third option: DeepSeek's **Multi-head Latent Attention (MLA)**, which compresses KV projections into a low-rank latent space.
    
2. **Query, Key, Value (Q, K, V)** — Three learned linear projections of each token's representation. Q encodes "what information am I seeking," K encodes "what information do I offer for matching," V encodes "what information do I contribute to the output." The mechanical core of every attention computation.
    
3. **Positional encoding / RoPE** — Any mechanism injecting sequence-order information, since attention is permutation-invariant by default. RoPE (Rotary Position Embedding) remains dominant, applying rotation matrices to Q and K vectors. Emerging: **hybrid RoPE/NoPE** schemes (alternating layers with and without positional encoding) for better long-context performance; **iRoPE** (Llama 4 variant).
    
4. **Feedforward Network (FFN)** — The MLP sub-layer within each Transformer block, processing each token independently through learned weight matrices. Research suggests FFN layers serve as the primary site of factual knowledge storage. In MoE architectures, the FFN is replaced by multiple expert FFNs with a learned router.
    
5. **BPE (Byte Pair Encoding)** — A tokenization algorithm that iteratively merges the most frequent adjacent byte/character pairs to build a subword vocabulary. Confirmed by a 2025 survey as the choice of most state-of-the-art LLMs (Llama, Mistral, DeepSeek, Qwen, Phi). Implementations: tiktoken (OpenAI, ~200K vocab), SentencePiece (Meta/Google, 128K vocab). Research frontier: Meta's **Byte Latent Transformer** (tokenizer-free, processes raw bytes) matches Llama 3 at 8B scale with 50% FLOP savings, but is not yet adopted in production.
    
6. **FlashAttention** — **[NEW]** An IO-aware exact attention algorithm that restructures the computation to minimize GPU memory reads/writes, achieving 2–4× speedups without approximation. Now at v3 (optimized for Hopper GPUs), universally adopted across all major training and inference frameworks. Not a different _kind_ of attention — mathematically identical to standard attention, just dramatically faster.
    
7. **Speculative decoding** — **[NEW]** An inference acceleration technique where a small "draft" model generates candidate tokens in parallel, and the large model verifies them in a single forward pass. Achieves 1.5–3× latency reduction. Google confirms it powers AI Overviews in production. Apple's Mirror Speculative Decoding (2026) achieves 2.8–5.8× speedups using heterogeneous accelerators.
    
8. **Structured / Constrained decoding** — **[NEW]** Techniques that guarantee model outputs conform to a formal grammar (JSON schema, regex, context-free grammar) by masking invalid tokens at each generation step. Implementations: XGrammar (default in vLLM, NVIDIA NIM), Outlines, llguidance (~50μs/token overhead). User-facing as "Structured Outputs" in OpenAI's API and similar features elsewhere.
    
9. **Min-p sampling** — **[NEW]** A dynamic sampling method that sets a threshold relative to the top token's probability (e.g., "only consider tokens with at least 10% of the top token's probability"). Published as an ICLR 2025 oral paper. Maintains coherence at high temperatures where top-p degrades. Integrated into HuggingFace, vLLM, Ollama, and llama.cpp; not yet in commercial APIs.
    
10. **RMSNorm** — A normalization variant that scales activations by their root-mean-square, omitting mean-centering. Has fully replaced LayerNorm across all major post-2023 architectures. An adjacent development: **QK-Norm** (normalizing query and key vectors for training stability) is increasingly standard in 2025 models.
    
11. **SwiGLU** — A gated activation function combining Swish with a Gated Linear Unit for the FFN sub-layer. Yields modest but consistent quality improvements over ReLU/GELU; now standard in virtually all modern Transformer designs.
    

---

## What was removed and why

|Term|Original Tier|Reason for removal|
|---|---|---|
|SentencePiece|Tier 4|Merged into BPE entry; it's an _implementation_ of BPE, not a separate concept|
|Transfer learning|Tier 2|Merged into "Fine-tuning / Transfer learning"; the distinction doesn't warrant a separate entry at this level|

## Summary of changes from original

**Added (12 terms):** Hallucination, Multimodal, Prompt engineering, Reasoning models, RAG, Agentic AI / Tool use, Mixture of Experts, GRPO, RLVR, Quantization, LoRA/QLoRA, Distillation, FlashAttention, Speculative decoding, Structured decoding, Min-p sampling

**Moved to easier tier:** Transformer (3→2), Self-attention (3→2), RLHF (3→2), DPO (4→3), KV-cache (4→3)

**Moved to harder tier:** Causal mask (2→3)

**Merged:** SentencePiece → into BPE; MHA + GQA → combined entry; Transfer learning → into Fine-tuning; Positional encoding + RoPE → combined entry