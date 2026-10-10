![slipstream-banner.jpg](slipstream-banner.jpg)

# Slipstream

> **Speculative Orchestrator**  
> **For Ollama and llama.cpp**  
> **A Frugal AI HQ Project**  
> **Made in Japan (日本製)**

---

## The Slipstream Manifesto: Zero CPU Offload

Slipstream is an inference orchestrator engineered for extreme-throughput speculative execution and deterministic multimodal reporting. To achieve these metrics, there is a single, non-negotiable operational law:

**You must maintain 100% VRAM residency.**

There is zero tolerance for CPU offloading. Both the model weights and the entire Key-Value (KV) cache must fit entirely on the GPU silicon. If your workload spills over the PCIe bus into system RAM, you are no longer running Slipstream; you are running a bottleneck.

### The Physics of the Bottleneck
The architecture of high-speed token generation is entirely bound by memory bandwidth. Modern GDDR6 GPU memory operates at speeds between 300 to 500+ GB/s. In contrast, DDR4 and DDR5 system memory tops out around 40 to 60 GB/s, and that data must be dragged across the latency penalty of the PCIe bus.

When you offload even a few layers to system RAM to run a slightly larger model, your entire generation loop is throttled down to the speed of your DDR sticks. Token generation instantly drops from 30+ tokens per second to a single-digit crawl. 

In speculative execution, where a draft model and a target model must synchronize rapidly, this PCIe latency is fatal. The entire draft-and-verify engine collapses under its own memory transfer overhead.

### The Law of Aggressive Quantization
Do not attempt to brute-force a Q8 model through system memory. 

If your target model does not fit natively on your VRAM, **step down the quant.** A Q4_K_M or Q3_K_M quant running completely resident on the GPU will utterly destroy the performance of a less-compressed model fighting for system RAM. Slipstream is designed to route and manage multi-GPU memory pools natively to maximize your available VRAM, but it will not save you from bad hardware allocation.

**If it doesn't fit on the card, you don't run it.**



### The `--vision-in-ram` Exemption (For the Pedants)
If you are reading the CLI documentation and think you have discovered a contradiction to the Zero CPU Offload rule because of the `--vision-in-ram` flag, congratulations on reading the manual. Now learn how the architecture actually works.

There is a fundamental physics difference between Time-To-First-Token (TTFT) and Tokens-Per-Second (TPS).

The multimodal projector (`mmproj`) is an intake mechanism. It evaluates the image matrix exactly once during prompt processing. If you use `--vision-in-ram` to save 1.5GB to 3GB of VRAM, you pay the PCIe latency tax exactly once. Your Time-To-First-Token might spike to 30 seconds as the CPU grinds through the image, but once those embeddings are handed off to the GPU, the vision encoder goes dormant.

The autoregressive generation loop—the actual engine where Slipstream's speculative MTP drafting occurs—takes over. Because the model weights and the KV cache remain 100% locked on the GPU silicon, generation instantly snaps back to maximum throughput.

Offloading weights or KV cache to system RAM forces the PCIe bus to choke on every single generated token, collapsing the speculative drafting loop. Offloading the vision encoder is a one-time toll fee at the gate.

If you do not understand the difference between a one-time $O(1)$ prompt evaluation penalty and an $O(N)$ autoregressive generation bottleneck, you should not be orchestrating multi-node hardware.




---

## スリップストリーム・マニフェスト：CPUオフロード厳禁

Slipstream（スリップストリーム）は、極限のスループットを誇る投機的実行（Speculative Execution）と、決定論的なマルチモーダル・レポート生成のために設計された推論オーケストレーターです。これらの指標を達成するためには、絶対に妥協できない唯一の運用原則があります：

**100%のVRAM完全常駐を維持すること。**

CPUへのオフロードは一切許容されません。モデルの重み（Weights）とすべてのKVキャッシュは、完全にGPUシリコン上に収まる必要があります。ワークロードがPCIeバスを越えてシステムメモリ（RAM）に溢れた瞬間、あなたはSlipstreamを動かしているのではなく、単なるボトルネックを動かしていることになります。

### ボトルネックの物理的現実
高速なトークン生成のアーキテクチャは、メモリ帯域幅に完全に依存しています。最新のGDDR6 GPUメモリは300〜500+ GB/sの速度で動作します。対照的に、DDR4やDDR5のシステムメモリはせいぜい40〜60 GB/sで頭打ちになり、そのデータはPCIeバスの遅延ペナルティを引きずりながら転送されなければなりません。

少しでも大きなモデルを動かすために、数レイヤーだけでもシステムRAMにオフロードした場合、生成ループ全体がDDRメモリの速度まで引き下げられます。トークン生成速度は即座に30+ t/sから1桁台へと激減します。

ドラフトモデルとターゲットモデルが高速で同期しなければならない投機的実行において、このPCIe遅延は致命的です。ドラフト・検証エンジン全体が、自らのメモリ転送のオーバーヘッドによって崩壊してしまいます。

### 妥協なき量子化の法則
Q8モデルをシステムメモリ経由で力技で動かそうとしないでください。

ターゲットモデルがVRAMにネイティブに収まらない場合は、**量子化のレベルを下げてください。** GPU上に完全常駐するQ4_K_MやQ3_K_Mの量子化モデルは、システムRAMの帯域を奪い合う低圧縮モデルのパフォーマンスを完全に圧倒します。Slipstreamは、利用可能なVRAMを最大化するためにマルチGPUのメモリプールをネイティブにルーティングおよび管理するように設計されていますが、ユーザー自身の誤ったハードウェア割り当て（メモリ配置）から救い出すことはできません。

**VRAMに収まらないなら、動かさないこと。**


### `--vision-in-ram` の例外規定（揚げ足取りな連中へ）

CLIのドキュメントを読み込んで、`--vision-in-ram` フラグの存在を盾に「CPUオフロード厳禁のルールと矛盾しているではないか」と鬼の首を取った気になっているなら——まずはマニュアルを読んだ熱心さだけは褒めておこう。だが、次はアーキテクチャが実際にどう動いているかを理解する番だ。

Time-To-First-Token（TTFT：初速トークン生成時間）と Tokens-Per-Second（TPS：継続的なトークン生成速度）の間には、物理的に決定的な違いが存在する。

マルチモーダル・プロジェクター（`mmproj`）は、あくまで入力機構（インテーク）に過ぎない。画像マトリクスの評価と計算が行われるのは、プロンプト処理時の「最初の1回」だけだ。1.5GB〜3GBのVRAMを節約するために `--vision-in-ram` を使った場合、PCIeの遅延ペナルティという税金を支払うのは文字通りその1回きりである。CPUが画像を処理する間、TTFTが30秒近くまで跳ね上がるかもしれないが、生成された埋め込み（Embeddings）がGPUに渡された瞬間、ビジョンエンコーダーは完全に休眠状態に入る。

そこから先は、Slipstreamの投機的MTPドラフトが真価を発揮する「自己回帰生成ループ」へとバトンが渡される。モデルの重みとKVキャッシュが100% GPUシリコン上に常駐している限り、トークン生成速度は瞬時に最大スループットへと復帰する。

重みやKVキャッシュをシステムRAMへオフロードすることは、生成される**すべてのトークンごと**にPCIeバスを窒息させ、投機的ドラフトの同期ループを崩壊させることを意味する。一方で、ビジョンエンコーダーのオフロードは「料金所で最初に一度だけ払う通行料」に過ぎない。

たった1度きりの $O(1)$ プロンプト評価ペナルティと、$O(N)$ で累積する自己回帰生成のボトルネックの違いすら理解できないのであれば、最初からマルチノードのハードウェア構成に口を挟むべきではない。



---

## Architecture & Hardware Economics

Most local AI workflows operate under an artificial hardware tax: practitioners looking to run 14B+ or 27B parameter multimodal models with meaningful context are pushed toward enterprise-grade accelerators or grossly inflated secondary-market consumer GPUs (such as the RTX 3060 12GB).

Slipstream is an air-gapped, containerized orchestration layer designed to dismantle vendor lock-in. Leveraging custom-compiled dynamic execution backends (`ggml-cuda` and `ggml-vulkan`), Slipstream dynamically partitions and streams model tensors across heterogeneous, mixed-vendor consumer GPUs. This unlocks large-scale local multimodal inference on salvage, budget, or repurposed desktop silicon.

### Cost Comparison (Secondary / Salvage Market)

Rather than sinking an entire budget into a single mid-range graphics card, the same capital can assemble an entire standalone, multi-GPU compute node delivering **14GB to 16GB of addressable VRAM.**

| Component | Single-GPU Baseline (RTX 3060 12GB) | Complete Slipstream Node (Dual-GPU Array) |
| :--- | :--- | :--- |
| **GPU(s)** | 1x NVIDIA RTX 3060 12GB (¥42,000) | 1x AMD RX 580 8GB + 1x GTX 1660 Super 6GB (¥9,000)* |
| **Motherboard** | — | ASUS Z170-PRO (LGA1151, Dual PCIe) (¥4,000) |
| **Processor** | — | Intel Core i7-6700K (¥4,000) |
| **Memory** | — | 16GB DDR4 RAM (¥11,000) |
| **Storage (OS/Swap)** | — | 128GB SATA SSD (¥4,000) |
| **Storage (Models)** | — | 2TB SATA HDD (¥4,000) |
| **Power & Chassis** | — | 850W Corsair PSU + Antec P180 Case (¥5,440) |
| **Total Investment** | **~¥42,000 (~$275 USD)** | **¥41,440 (~$270 USD)** |
| **Total Available VRAM** | **12 GB** | **14 GB to 16 GB** |

*\*Note 1: Configuring with two AMD RX 580 8GB cards achieves 16GB total VRAM at an identical or lower price point.*  
*\*Note 2: Retail pricing for brand-new RTX 3060 12GB cards has reached between ¥75,000 and ¥85,000, widening the value gap even further.*

---

## アーキテクチャとハードウェア経済性

多くのローカルAI運用環境は、不当な「ハードウェア税」を強いられています。十分なコンテキスト長を維持したまま14Bや27Bクラスのマルチモーダルモデルを動作させるために、開発者は高額なエンタープライズ向けGPUや、二次流通市場で価格が高騰したコンシューマーカード（RTX 3060 12GBなど）の購入を迫られてきました。

Slipstreamは、特定ベンダーへの依存（ベンダーロックイン）を排除するために設計された、完全隔離型・コンテナ化推論オーケストレーションレイヤーです。独自ビルドされた動的実行バックエンド（`ggml-cuda` および `ggml-vulkan`）を駆使し、メーカーの異なる異種混合（NVIDIA / AMD）コンシューマーGPU間でモデルテンソルを動的に分割・ストリーミングします。これにより、型落ちPCや中古・リユース品、低予算デスクトップ環境であっても、大規模なローカルマルチモーダル推論の実行が可能になります。

### コスト比較（中古・リユース市場ベース）

単一のミドルレンジGPUに予算を全額投入する代わりに、全く同じ資金で**14GB〜16GBの実効VRAM**を備えた完全自立型のマルチGPU計算ノードを1台丸ごと構築できます。

| 構成コンポーネント | 単一GPU導入ベースライン (RTX 3060 12GB) | Slipstream完全ノード (デュアルGPU構成) |
| :--- | :--- | :--- |
| **GPU** | 1x NVIDIA RTX 3060 12GB (¥42,000) | 1x AMD RX 580 8GB + 1x GTX 1660 Super 6GB (¥9,000)* |
| **マザーボード** | — | ASUS Z170-PRO (LGA1151, Dual PCIe) (¥4,000) |
| **CPU** | — | Intel Core i7-6700K (¥4,000) |
| **メモリ** | — | 16GB DDR4 RAM (¥11,000) |
| **ストレージ (OS/Swap)** | — | 128GB SATA SSD (¥4,000) |
| **ストレージ (モデル格納)** | — | 2TB SATA HDD (¥4,000) |
| **電源 & 筐体** | — | 850W Corsair PSU + Antec P180 ケース (¥5,440) |
| **総投資額** | **約 ¥42,000** | **¥41,440** |
| **利用可能 VRAM** | **12 GB** | **14 GB 〜 16 GB** |

*\*注記1: AMD RX 580 8GBの2枚差し構成を採用した場合、同等以下のコストで合計16GBのVRAMを確保可能です。*  
*\*注記2: 新品RTX 3060 12GBの実勢小売価格は75,000円〜85,000円前後まで跳ね上がっており、単一パーツ購入の費用対効果の悪さはさらに顕著になっています。*

---

## Deployment & Quickstart

Slipstream is distributed strictly as a self-contained runtime bundle containing pre-compiled binaries and orchestration scripts. No raw source code is required or provided.

### 1. Prerequisites & Installation

Slipstream relies entirely on Docker for its zero-leakage containerized deployment, so you must have Docker-CE installed directly from the official Docker website. It is an orchestration layer designed to interface with your existing local inference engine, meaning **Ollama** must be installed and running natively on your host machine (`localhost`) before launching the container.

1. Download the latest `slipstream-VERSION.tar.gz` payload from the [Releases](https://github.com/Slipstream-llm/Slipstream/releases) page.


2. Extract the archive into your desired directory:

    ```bash
    tar -xzf slipstream-VERSION.tar.gz
    cd slipstream
    ```


3. Launch the orchestrator using the provided wrapper script:

    ```bash
    ./start-slipstream.sh
    ```



*(This will automatically execute the necessary `docker compose` commands and build the local container using the binary payload)*.

### 2. Verify Endpoints

Once initialized, the orchestrator exposes its inference endpoints directly on your host machine.

* **Ollama-compatible:** `curl http://localhost:7777/api/tags`

* **OpenAI-compatible:** `curl http://localhost:7777/v1/models`


### 3. Take the Engine for a Spin

Because Slipstream operates as a strictly isolated container, you will not find the `slip` command polluting your host system's standard PATH. Drop directly into the container to manage your models and run your first native inference.

Jump into the active Slipstream container:

```bash
docker exec -it slipstream bash
```

Run a model natively. If it is an official Slipstream-optimized model, the orchestrator will automatically handle the MTP drafting parameters:

```bash
root@0e05fe9b6b38:/workspace# slip run Slipstream/qwen3.5:4b-q4_k_m-slipstream
>>> Hello!
<think>

</think>

Hello! How can I help you today? 😊  
Feel free to ask a question, need assistance with a task, or just want to chat!
```

Inspect the live telemetry to verify your 100% VRAM residency and watch the engine flex:

```text
root@0e05fe9b6b38:/workspace# slip ps
=== SLIPSTREAM RUNTIME STATUS ===
Slipstream Mode: Slipstream Native (Strict Execution)
Backend: CUDA | Flash Attn: ON | GPU Count: 1
VRAM: 6.000 GB / 12.000 GB (6.100 GB Free)

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
[slipstream] [*] ALIVE / WAITING
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  Model:    Slipstream/qwen3.5:4b-q4_k_m-slipstream
  Strategy: ngram-map-k (depth: 3)
  Geometry: 4096 CTX per thread | Est. 3.3 GB (100% GPU)
            4096 CTX allocated for all threads
  Threads:  0/1 (0 Active | 0 Queued)
  Metrics:  Load: 2755ms | TTFT: 431ms
            Speed: 57.32 t/s (57.32 t/s per slot) [last]
            Drafting: 0.00% (0 accepted / 0 proposed)
            Acceptance Rate: 0.00%
  Expires:  No expiration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

### 4. Expand Your Local Cluster

Want to push your hardware further? You don't have to configure draft models manually. All official, zero-configuration Slipstream models—including our ultra-fast 2B/4B vision models and heavy-duty Qwen variants—are pre-calibrated and waiting for you in the official registry.

**[Browse the Official Registry: ollama.com/Slipstream](https://ollama.com/Slipstream)**

### Building Custom Models

Slipstream uses specialized `SYSTEM` prompt pragmas to dynamically route speculative draft models and manage multi-threading. If you want to configure your own local models for Slipstream's orchestration engine, read the **[Custom Model Configuration Guide](CUSTOM_MODELS.md)**.

---

## Advanced Configuration (Environment Variables)

Slipstream is designed to work out-of-the-box with standard default paths, but it can be dynamically configured using environment variables. You can export these variables in your shell before launching, or pass them inline with the start script.

* **`OLLAMA_MODELS`**: Overrides the default model directory. Use this if your `.gguf` files are stored on a separate drive or non-standard path rather than the default `~/.ollama/models`.


* *Example:* `OLLAMA_MODELS=/mnt/nvme/models ./start-slipstream.sh`



* **`SLIP_OLLAMA_PORT`**: Instructs Slipstream to bind to a custom Ollama host port if your base inference engine is not running on the default `11434`.


* *Example:* `SLIP_OLLAMA_PORT=11435 ./start-slipstream.sh`



* **`SLIP_QUIRK`**: Activates specialized orchestration routing for specific hardware edge cases or complex multi-vendor setups (such as custom memory pooling for AMD Radeon Pro V620 arrays).


* *Example:* `SLIP_QUIRK=v620 ./start-slipstream.sh`


For a full list of runtime arguments and utility flags, use the built-in help menu:

```bash
./start-slipstream.sh --help
```

---

## Upgrading

To install a new release, cleanly remove the old runtime environment and force a rebuild with the new payload:

```bash
./stop-slipstream.sh
cd ..
rm -rf slipstream
# Extract the new slipstream.tar.gz here, then:
cd slipstream
./start-slipstream.sh --build
```

---

## Frequently Asked Questions

**Q: Do I need to compile Slipstream from source?**
No. Slipstream is distributed strictly as a self-contained runtime bundle containing pre-compiled binaries and orchestration scripts. You simply extract the release payload and use the provided `start-slipstream.sh` script to launch the orchestrator, and `stop-slipstream.sh` to safely spin it down.

**Q: Do I have to configure my own draft models for speculative decoding?**
No. While advanced users can build custom pipelines, we maintain an official registry of Slipstream-optimized models. These models (ranging from lightweight 2B/4B vision models to massive multimodel payloads) are pre-configured with the exact drafting parameters needed to hit maximum TPS on your hardware. You can pull them immediately from [ollama.com/Slipstream](https://ollama.com/Slipstream).

**Q: Can I offload just a few layers to system RAM if my model is slightly too big?**
Absolutely not. Slipstream enforces a strict zero CPU offload policy where both the model weights and the entire KV cache must fit completely on the GPU silicon. If you offload even a few layers, your generation loop is throttled down to system DDR speeds, which causes the speculative drafting loop to collapse under memory transfer overhead. If your target model does not fit natively, step down the quant.

**Q: What about the `--vision-in-ram` flag? Doesn't that violate the zero offload rule?**
No. Offloading the vision encoder is a one-time $O(1)$ penalty during prompt evaluation. Your Time-To-First-Token will spike, but once the embeddings are handed off to the GPU, the vision encoder goes dormant. The autoregressive generation loop takes over, and because the weights and KV cache remain 100% locked on the GPU, token generation instantly returns to maximum throughput.

**Q: Why use cast-off GPUs instead of buying a modern 12GB or 16GB card?**
It is a matter of hardware economics. Slipstream dynamically partitions and streams model tensors across heterogeneous, mixed-vendor consumer GPUs. Instead of spending around ¥42,000 on a single RTX 3060 12GB, you can assemble a complete, standalone compute node with 14GB to 16GB of VRAM using secondary market parts (like combining an AMD RX 580 8GB and a GTX 1660 Super 6GB) for a fraction of the hardware cost.

**Q: Is it safe to use Ollama's standard garbage collection while running Slipstream?**
Admin Warning: Exercise caution. Ollama's native garbage collector currently does not cryptographically trace Slipstream's dynamic `// DRAFT_MODEL` tags. Running aggressive blob pruning operations on your host can result in the accidental deletion of draft models that are actively tethered via Slipstream.

**Q: Can I use this for commercial or enterprise projects?**
Yes. Slipstream is released as freeware and grants a free, worldwide license for personal, educational, research, or commercial use. However, if you redistribute the binary or build a user-facing dashboard derived from it, you must retain the original copyright notice, provide prominent attribution to "Frugal AI HQ", and ensure that the command-line startup banners are not removed or obfuscated.


## License

Slipstream is distributed under the **Slipstream Freeware License**. 

* **100% Free:** You are granted a free, worldwide license to download, install, execute, copy, and redistribute this binary for any personal, educational, research, or commercial purpose at no cost.
* **Attribution Required:** If you redistribute Slipstream or build a dashboard directly derived from it, you must retain the copyright notices, project credits, and links to Frugal AI HQ. Command-line startup banners must not be suppressed.

See the full [LICENSE.md](LICENSE.md) file for exact terms.