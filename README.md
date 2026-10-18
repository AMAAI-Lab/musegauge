# musegauge

[![PyPI](https://img.shields.io/pypi/v/musegauge)](https://pypi.org/project/musegauge/)
[![Licence](https://img.shields.io/badge/licence-Apache--2.0-blue)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue)](pyproject.toml)

Score a folder of generated music with several existing metrics in one command.

> **Needs:** a GPU, about 28 GB of free disk, Linux x86_64.
> Tested on Linux x86_64 only. Other systems are not tested.

musegauge runs these existing metrics:

| Metric | Tool |
| --- | --- |
| Frechet Audio Distance | `fadtk` |
| Kernel Audio Distance | `kadtk` |
| CLAP text-audio score | `laion-clap` |
| Audiobox Aesthetics | Audiobox Aesthetics |

Each metric runs in its own Python environment. The tool builds these by itself with uv, so
metrics that need different torch versions can be used together. One run writes a
`results.json` and a short report card for a paper appendix.

Version 0.1.2. On PyPI. Source: <https://github.com/aroy1990-dev/musegauge>.
Container image: `ghcr.io/aroy1990-dev/musegauge`.

## Authors

musegauge was created and is maintained by **Abhinaba Roy**
([@aroy1990-dev](https://github.com/aroy1990-dev)), AMAAI Lab, Singapore University of
Technology and Design (SUTD).
Contact: abhinabaroy1990@gmail.com

## Quick start

### Docker (published image)

No install is needed beyond Docker. Python, uv and musegauge are inside the image.

```bash
docker pull ghcr.io/aroy1990-dev/musegauge:0.1.2
docker run --rm --gpus all -v "$PWD/my_audio:/data/audio:ro" -v musegauge-cache:/cache \
  -v "$PWD/out:/out" ghcr.io/aroy1990-dev/musegauge:0.1.2 \
  run --generated /data/audio --out /out
```

The image `0.1.2` was built and tested on 2026-10-06 and 2026-10-07 on a Linux x86_64 machine
with Docker 27.3.1. `doctor`, an offline run (`--network none --no-fetch`) and GPU use all
passed. The tests found a cache-volume bug, which is fixed in 0.1.2. See `docker/README.md`
and `docs/PROGRESS.md` (M7).

### pip

Python 3.10 or newer.

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install musegauge
musegauge doctor
musegauge run --generated ./my_audio --prompts prompts.csv --out ./out
```

If `python3 -m venv` fails because `ensurepip` is missing (a Debian or Ubuntu system Python
without the `python3-venv` package), use another Python, or `uv venv`.

### uvx

No install. Needs uv.

```bash
uvx musegauge run --generated ./my_audio --out ./out
uvx --from . musegauge --version                         # from a clone
```

### Docker (build it yourself)

From a clone. `docker/README.md` has the full commands and the published-image alternative.

```bash
uv build && docker build -f docker/Dockerfile --target slim -t musegauge:slim .
docker run --rm --gpus all -v "$PWD/my_audio:/data/audio:ro" -v musegauge-cache:/cache \
  -v "$PWD/out:/out" musegauge:slim run --generated /data/audio --out /out
```

### First run

The first run builds the plugin environments and downloads the model weights into the cache
folder (`~/.cache/musegauge`, or `/cache` in the container). For all four plugins this is
about 24 GB of environments and 3.7 GB of weights. `musegauge setup --all --fetch-weights`
does this ahead of time. All install paths are in `docs/INSTALL.md`.

## Try it on synthetic audio

From a fresh clone, in a new shell:

```bash
git clone https://github.com/aroy1990-dev/musegauge.git && cd musegauge
python3 -m venv .venv && source .venv/bin/activate
pip install .
python tests/make_fixtures.py fixtures
musegauge run --generated fixtures/gen_small --metrics aesthetics.audiobox@1 --out out
cat out/report.md
```

`tests/make_fixtures.py` writes synthetic sine-and-noise clips (no real music). The first run
builds the Audiobox environment (about 5 GB) and downloads its weights (about 400 MB). Scores
on synthetic audio mean nothing about quality.

## Input and output

- **Input:** a folder of `.wav`, `.flac`, `.ogg` or `.mp3` files (`--generated DIR`, with
  optional `--prompts FILE`), or a clips file in JSON lines (`--manifest FILE`).
- **Metrics:** a suite (`--suite t2m-basic`, the default, or `t2m-full`) or
  `--metrics ID,ID`. `musegauge list` shows them. `musegauge info METRIC_ID` shows a
  definition and its licences.
- **Reference:** `--reference DIR` or `--reference bundled:fma_pop` (FAD only).
- **Output folder:** `results.json`, `report.md`, `per_clip/`, `logs/`, `requests/`,
  `responses/`.

One example output, from `docs/examples/report.md` (synthetic audio, so the numbers mean
nothing):

```text
| Metric | Score | Mean ± std | 95% CI | n | Status |
| --- | --- | --- | --- | --- | --- |
| fad.vggish@1 | fad | 25.369 | — | — | ok |
| fad.clap-laion-music@1 | fad | 1.274 | — | — | ok |
| clapscore.laion-music@1 | clap_cosine | 0.232 ± 0.059 | [0.200, 0.262] | 12 | ok |
| aesthetics.audiobox@1 | CE | 2.405 ± 0.557 | [2.115, 2.705] | 12 | ok |
| aesthetics.audiobox@1 | PQ | 6.228 ± 0.827 | [5.757, 6.655] | 12 | ok |
```

## Things to know

- **Offline use.** `--no-fetch` blocks downloads from libraries that honour the proxy
  settings, and sets the Hugging Face offline variables. It is best effort, not a guarantee.
  For a hard guarantee, cut the network outside the tool (for example, a container with no
  network). It was tested only with the proxy method, not with a truly blocked network. Run
  `musegauge setup --fetch-weights` first. `run --no-fetch` never builds an environment.
- **Network without `--no-fetch`.** FAD and KAD with VGGish contact github.com on every run
  (torch.hub). An HTTP error there is reported as `NETWORK_ERROR`.
- **Licences of the metrics.** Every metric's `commercial_ok` is `unknown`, so every run warns
  `UNKNOWN_LICENCE`. `--commercial` refuses such metrics. Details: `docs/LICENCES.md`.
- **Comparing numbers.** Numbers are comparable only with the same metric ids, reference set,
  clip length, device and thread setting. `fad_inf` is random from run to run. See
  `docs/METRICS.md`.
- **Shared machines.** `--threads N` limits CPU threads in each plugin (`docs/INSTALL.md`).

## Documentation

| File | Contents |
| --- | --- |
| `docs/INSTALL.md` | Install paths, offline use, disk space, GPU driver, threads |
| `docs/METRICS.md` | What each metric is and where numbers can differ |
| `docs/LICENCES.md` | Code, weights and data licence status per metric |
| `docs/plugins/` | One page per plugin: commands, files, downloads, differences from upstream |
| `docs/PLUGIN_GUIDE.md` | How to write a plugin |
| `docs/UPSTREAM_NOTES.md` | Problems found in the upstream tools |
| `docs/TESTING_REAL_PLUGINS.md` | Golden tests with the real tools |
| `docs/DECISIONS.md` | Choices made, and every change from the build spec |
| `docs/VERIFIED_FACTS.md`, `docs/PROGRESS.md`, `docs/BUILD_REPORT.md` | Checks and build record |

## Citation

If you use musegauge in your work, please cite it:

```bibtex
@software{musegauge,
  author  = {Roy, Abhinaba},
  title   = {musegauge: scoring generated music with existing metrics in one command},
  year    = {2026},
  version = {0.1.2},
  url     = {https://github.com/aroy1990-dev/musegauge}
}
```

## Licence

Copyright 2026 Abhinaba Roy.

Licensed under the Apache License, Version 2.0. See [`LICENSE`](LICENSE) for the full text
and [`NOTICE`](NOTICE) for the attribution notice. You may use, change and share this code
under those terms, but you must keep the copyright and NOTICE text.

The metric packages and model weights that the plugins download have their own licences. See
`docs/LICENCES.md`.
