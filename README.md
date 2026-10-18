# stable diffusion cloud: How to Pick the Right GPU, What It Really Costs, and How to Run ComfyUI on Vultr

Running Stable Diffusion in the cloud comes down to three numbers: how much VRAM your model needs, what the GPU costs per hour, and how many images you actually generate. Get those right and a full SDXL setup can cost less per month than a streaming subscription. Get them wrong and you'll pay monthly for a GPU that sits idle. This guide covers the real VRAM requirements from SD 1.5 through SD 3.5, what per-hour rates turn into in practice, and how to get ComfyUI or AUTOMATIC1111 running on Vultr's Cloud GPU instances.

> **Quick pick:** if you're still experimenting, rent a 24GB-class GPU on Vultr by the hour and destroy the instance when you're done — you pay for the minutes you use. If you're generating daily or hosting an image API, pick a bigger card (L40S or A100) and run it always-on, since hourly billing caps out at a fixed monthly rate. 👉 [Check Vultr's Cloud GPU lineup and current hourly rates](https://www.vultr.com/products/cloud-gpu/?ref=7268283)

## Three ways to run Stable Diffusion in the cloud — and when each one wins

There are three billing models on the market, and the cheapest one depends entirely on your volume. Per-image APIs charge nothing until you generate. Hourly GPU rentals charge for time. Always-on rentals charge time but cap it at a fixed monthly price.

| Route | How you pay | Typical market rate | Makes sense when |
| --- | --- | --- | --- |
| Per-image API | Pay per generation | ~$0.0023/image (SD 1.5), ~$0.012/image (SDXL) | A few dozen images a month, zero setup |
| Hourly cloud GPU | Pay per hour the instance runs | Roughly $0.20–0.51/hr for a 24GB RTX 4090-class card | Experiments, burst jobs, testing workflows |
| Always-on cloud GPU | Hourly, capped at a monthly rate | Varies by provider and card | Daily generation, batch jobs, hosted APIs |

The crossover point comes fast. Generating 1,000 SDXL images takes about five GPU-hours on an RTX 4090. At RunPod's listed $0.34/hr that's roughly $1.70; through Replicate's API the same batch runs about $12. Scale that to 100,000 images a month and an always-on RTX 4090 costs around $151 while the API route climbs to roughly $1,200. Self-hosting wins once volume gets serious — the gap is 5–10× at production scale.

Per-image APIs still have a place. If you generate twenty images a month and never want to touch a terminal, $0.24 total is a fair price for never thinking about CUDA drivers.

## Vultr's GPU lineup for Stable Diffusion

Vultr is a general cloud infrastructure platform rather than an SD-specific host, and the piece that matters for image generation is its Cloud GPU line: NVIDIA cards spanning 24GB up to 80GB HBM, deployable in over 30 global data center locations. Everything bills by the hour with a monthly cap, so an instance running 24/7 costs exactly the monthly cap, and one running three evenings a week costs a fraction of that.

| Plan | GPU & VRAM | Best for | Billing | Deploy |
| --- | --- | --- | --- | --- |
| Cloud GPU – RTX 4090 | 24GB | SD 1.5 / SDXL with ControlNet and LoRA stacks; small SDXL LoRA training runs | Hourly, capped monthly | [ Deploy](https://www.vultr.com/products/cloud-gpu/?ref=7268283) |
| Cloud GPU – L40S | 48GB | FLUX (needs 24GB+), AnimateDiff / SVD video work, large LoRA stacks | Hourly, capped monthly | [ Deploy](https://www.vultr.com/products/cloud-gpu/?ref=7268283) |
| Cloud GPU – A100 | 80GB | High-res SDXL, SD 3.5 Large (~20GB in fp16), batch production | Hourly, capped monthly | [ Deploy](https://www.vultr.com/products/cloud-gpu/?ref=7268283) |
| Cloud GPU – H100 | 80GB | Maximum throughput, hosted image APIs at scale | Hourly, capped monthly | [ Deploy](https://www.vultr.com/products/cloud-gpu/?ref=7268283) |
| Cloud Compute | CPU-based VPS, several tiers | Frontend, API wrapper, storage — not generation itself | Hourly or monthly | [ Deploy](https://www.vultr.com/products/cloud-compute/?ref=7268283) |
| Bare Metal | Dedicated physical GPU servers | Sustained heavy training workloads | Monthly | [ Deploy](https://www.vultr.com/products/bare-metal/?ref=7268283) |

Exact per-hour figures change and vary by card and location, so the table shows the billing shape; 👉 [Vultr's live pricing page](https://www.vultr.com/pricing/?ref=7268283) lists the current rates before you commit to anything.

One honest trade-off to know upfront: Vultr gives you a clean Ubuntu instance with full root access, and you install ComfyUI or AUTOMATIC1111 yourself. Purpose-built SD hosts ship one-click templates instead. If you want zero setup above all else, that difference matters — more on it in the comparison below.

## Matching the model to the VRAM

Most "which cloud should I pick" articles skip the step that actually determines your bill: model size. Here's what the major Stable Diffusion models demand, based on their fp16 weights.

| Model | Size (fp16) | Comfortable on | Worth knowing |
| --- | --- | --- | --- |
| SD 1.5 | ~4.27GB | Any 8GB+ GPU | The AUTOMATIC1111 classic; huge LoRA and extension ecosystem |
| SD 2 | ~5.2GB | 8GB+ GPU | OpenRAIL license |
| SDXL base 1.0 | ~6.76GB | 12GB minimum, 16–24GB comfortable | ComfyUI recommended; works with the SDXL Refiner |
| SD 3 Medium | ~10GB | 24GB-class (RTX 4090 tier) | License is stricter — not open source, requires an API license |
| SD 3.5 Large | ~20GB | A100-40GB class or newer top-end cards | Check the license terms before commercial use |

Two things rarely make it into comparison posts. First, licensing: SD 1.5 and SDXL use CreativeML OpenRAIL licenses that are friendly to most projects, while the SD 3.x line carries materially stricter terms — if your output is commercial, read them before you build a pipeline around those models. Second, the audio and video siblings: Stable Audio (~7.6GB) and Stable Video Diffusion (~8GB) need FFmpeg as an extra component and carry non-commercial RAIL licenses, so don't assume a "Stable Diffusion" plan covers them for business use.

For the practical middle ground: a 24GB RTX 4090-class card runs SDXL with ControlNet and a stack of LoRAs without swapping, and it's enough to train an SDXL LoRA on small datasets (roughly 20–100 images) with kohya_ss. You only need the 48GB or 80GB tier once you're doing FLUX, video, or batch production at resolution.

## What you'll actually pay

Here's where hourly-with-a-cap billing earns its keep, and where the one real trap hides. On Vultr, an instance bills per hour while it exists, up to the monthly cap. Run it 24/7 and you pay the cap. Run it ten hours a week and you pay for those ten hours. Either way you know the ceiling before you deploy.

The trap: a stopped instance keeps billing. Shutting down doesn't release the disk, the IP, or the invoice. Only a destroyed instance stops the meter. So the workflow for casual use is: deploy → generate → save your outputs elsewhere → destroy. You can always redeploy in about a minute, and checkpoints re-download from Hugging Face faster than paying a week's rent on an idle GPU. The same applies to any hourly-billed cloud, not just this one — it's the single most common way people overspend.

Some grounded math for planning:

- **Weekend experimenter:** 8 GPU-hours/month on a 24GB card. At market rates around $0.34/hr, that's under $3. Via a per-image API at 500 SDXL images, about $6. Hourly rental wins, barely — and gives you ComfyUI, custom LoRAs, and ControlNet, which the simple APIs don't.
- **Daily hobbyist:** 60–80 GPU-hours/month. Still well under the monthly cap of most 24GB plans.
- **Production batch:** always-on makes sense. The reference point from current market pricing: roughly $151/month for a continuously running RTX 4090 versus ~$1,200/month for the same 100K images through a per-image API.

That last line is the whole argument for a stable diffusion cloud setup over third-party APIs once volume is real.

## Deploying ComfyUI or AUTOMATIC1111 on Vultr, step by step

The whole install is a handful of commands. No marketplace template required.

1. Deploy a Cloud GPU instance from Vultr's product page — pick a location near you and Ubuntu 22.04 as the OS.
2. SSH in and clone AUTOMATIC1111: `git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui`, then run `./webui.sh --listen`.
3. Open port 7860 in the firewall and access the WebUI through your instance's public IP — and set authentication, because an open port on the internet is an open door.
4. Drop your checkpoints into the models folder (SDXL base from Hugging Face is the usual starting point), add LoRAs and ControlNet models as needed.
5. If you prefer node-based workflows, install ComfyUI instead — same VRAM, more flexibility for complex pipelines.

Speed expectations, so you can sanity-check your setup: an RTX 4090-class card generates a 1024×1024 SDXL image in about 3–5 seconds at 20 steps; an A100 80GB is roughly 30% faster on batches. Cold start after boot runs 30–60 seconds for model load, which is why hosted image APIs keep the instance warm.

If you're building an API on top, wrap ComfyUI with FastAPI behind Nginx and keep the instance running. One thing to remember with Vultr specifically: instance storage is ephemeral on destroy. Before you tear an instance down to stop billing, save anything you want to keep — checkpoints you downloaded live on that disk. 👉 [Create a Vultr account and deploy your first GPU instance](https://bit.ly/VulTR)

## How Vultr compares to the SD-specific hosts

Fair comparison, with the trade-offs stated plainly.

| Provider | Billing model | Known rate | Setup style |
| --- | --- | --- | --- |
| RunPod | Hourly GPU | $0.34/hr RTX 4090 | One-click templates for A1111, ComfyUI, Fooocus, InvokeAI |
| ThinkDiffusion | Hourly workspace | $0.50/hr | Preconfigured A1111/Forge/ComfyUI, persistent storage, built for non-technical artists |
| Paperspace Gradient | Hourly notebook | $0.51/hr | Notebook-based, preconfigured for SDXL and ControlNet |
| Replicate | Per-image API | ~$0.012/image SDXL | Zero setup, API only |
| Vultr | Hourly GPU with monthly cap | Live rates on its pricing page | Manual install, full root access |

RunPod and ThinkDiffusion are purpose-built for Stable Diffusion, and if you want one-click templates and nothing else, they do that well — RunPod alone ships 50+ community templates. Vultr's case is different: it's one platform for the GPU plus the rest of your stack. If your image pipeline sits next to a website, a database, Kubernetes, or object storage, keeping everything under one account with one bill beats stitching together a specialist host and a general cloud. The hourly-capped billing also means casual users aren't forced into monthly commitments the way some fixed-price hosts require.

## FAQ

**How much VRAM does Stable Diffusion need?**
SD 1.5 and SD 2 run on 8GB. SDXL wants 12GB minimum and gets comfortable at 16–24GB. SD 3.5 Large runs about 20GB in fp16, which puts it in A100 or 48GB+ territory.

**Should I use ComfyUI or AUTOMATIC1111?**
AUTOMATIC1111 is easier to start with and has the bigger extension ecosystem; it's the usual choice for artists. ComfyUI's node-based approach handles complex pipelines better and tends to perform better on them; it's the usual choice for developers building automated workflows. Forge sits in between as a faster A1111 fork.

**Can I train LoRAs in the cloud?**
Yes. SDXL LoRA training fits on a 24GB RTX 4090-class card for small datasets of roughly 20–100 images using kohya_ss or sd-scripts. Larger datasets or bigger batch sizes are what the A100 80GB tier is for.

**Is cloud cheaper than buying an RTX 4090?**
A local RTX 4090 build runs $2,000+ before electricity. If you generate daily at high volume, hardware eventually wins. If your usage is bursty — weekends, client projects, experiments — hourly rental at $0.20–0.50 stays cheaper for years of typical hobbyist volume.

## Picking a plan before you deploy

Match the decision to your situation:

- **Just trying Stable Diffusion:** hourly 24GB instance, destroy it when you finish each session. Total cost stays in the single digits per month.
- **Daily SDXL with ControlNet and LoRAs:** same card, run it more often, or let it ride always-on under the cap.
- **FLUX, video, or an image API:** L40S or A100, always-on. H100 only if you've measured and need the throughput.
- **Commercial output:** check the model license first — SD 1.5 and SDXL are permissive OpenRAIL; the SD 3.x line is not.

👉 [Compare Vultr's Cloud GPU plans and hourly rates](https://www.vultr.com/products/cloud-gpu/?ref=7268283), deploy the smallest card that fits your model, and you'll know within an hour whether the setup works for you — for the price of a coffee.
