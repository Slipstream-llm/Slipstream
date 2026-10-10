# Slipstream Custom Model Configuration Guide

Slipstream relies on specialized pragma declarations embedded directly within the `SYSTEM` prompt of an Ollama Modelfile. This allows the orchestrator to intercept runtime parameters and dynamically route resources at the hardware edge without requiring a separate configuration file.

To configure a custom model, insert these `//` pragmas into your Modelfile's `SYSTEM` block.

## Speculative Decoding & Drafting

Slipstream's massive throughput gains come from its highly opinionated drafting engine.

**`// DRAFT_TYPE <type>`**
Defines the speculative decoding strategy.

* `on`: Automatically selects the optimal default (`ngram-map-k`).


* `ngram-map-k`: **(Default / Recommended)** The most efficient fallback, providing stable speed-ups.


* `draft-mtp`: Engineered for extreme speed. Requires a model with an embedded draft structure (e.g., Qwen3.8:27B).


* `ngram-simple`: Alternative n-gram drafting strategy.


* `ngram-cache`: Alternative n-gram drafting strategy.


* `draft-simple`: Standard speculative decoding. **Requires** specifying a draft model via `// DRAFT_MODEL`. *(Note: Often results in a performance penalty depending on hardware PCIe bottlenecks)*.



**`// DRAFT_MODEL <model_name>`**
Explicitly declares the target draft model. Only used when `// DRAFT_TYPE draft-simple` is declared.

**`// DRAFT_DEPTH <n>`**
An integer defining how many tokens the draft model should predict ahead of the target model.

* **Default:** `3`


**`// DRAFT_P_MIN <n>`**
A float value defining the minimum probability threshold for draft token acceptance.

## VRAM & Memory Management

Slipstream enforces strict memory residency. To maximize context windows on limited hardware, KV cache compression is enabled by default.

**`// KV_CACHE <compression_level>`**
Sets the Key-Value cache compression for the main target model.

* **Valid Values:** `q4_0`, `q8_0`, `f16`

* **Default:** `q4_0`


**`// DRAFT_KV_CACHE <compression_level>`**
Sets the Key-Value cache compression specifically for the draft model.

* **Valid Values:** `q4_0`, `q8_0`, `f16`

* **Default:** `q4_0`


## Concurrency & Context Geometry

Slipstream handles threading differently than standard local inference engines, treating context allocation logically per-thread rather than as a global pool.

**`// THREADS <n>`**
A positive integer representing the maximum number of concurrent inference threads the model can process.

* **Default:** `1`


*Architectural Note:* When you increase the `THREADS` count, Slipstream provisions a strictly private context window for *each* thread based on the `PARAMETER num_ctx` defined in the Modelfile. For example, if `num_ctx` is 4096 and `THREADS` is 2, the total VRAM allocation for context will be 8192. Running `slip ps` will display both the per-thread context size and the aggregate VRAM footprint to ensure you remain fully resident on the GPU silicon.

## Modelfile Inheritance

**`// INHERIT [true|yes|on|1]`**
Toggles whether Slipstream pragmas are inherited from parent `SYSTEM` prompts when layering Modelfiles.

* **Default:** `on`
* Any other value will turn inheritance off.


