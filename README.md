<h1 align="center">
  <br>
  <a href="https://buddhistai.tools/"><img src="https://raw.githubusercontent.com/WeBuddhist/visual-assets/refs/heads/main/logo/WB-logo-purple.png" alt="OpenPecha" width="150"></a>
  <br>
</h1>

<h1 align="center">tibetan-aligner</h1>

<p align="center">
  <img src="https://img.shields.io/badge/python-3.8-blue.svg" alt="Python 3.8">
  <img src="https://img.shields.io/badge/topic-NLP-orange.svg" alt="Topic: NLP">
</p>

Sentence-align parallel Tibetan and English texts to produce training pairs for machine translation and translation memory.

Given a Tibetan file and an English file that contain the same content (one sentence per line), tibetan-aligner embeds every sentence with a Tibetan–English SentenceTransformer, runs [Vecalign](https://github.com/thompsonb/vecalign) to find the best sentence-level alignment, and writes tab-separated source/target pairs ready for downstream MT training.

This repo is a downstream **fork of [sebastian-nehrdich/align-tibetan](https://github.com/sebastian-nehrdich/align-tibetan)** by Sebastian Nehrdich, who wrote the alignment pipeline. OpenPecha's changes over upstream are packaging only: a Dockerfile, GitHub Actions for container publishing, a `requirements.txt`, and sample test data. The alignment algorithm and all processing scripts are upstream's work—credit and citation should go to him.

## Who this project is for

This project is for NLP engineers and researchers working with Tibetan–English parallel corpora—typically to build translation memories or training data for machine translation models. The [mt-training-data-prep-tools](https://github.com/OpenPecha/mt-training-data-prep-tools) pipeline uses it internally, and you can also run it as a standalone Docker container for one-off alignments.

If you want a hosted API instead of running locally, see the [tibetan-aligner-api Gradio Space](https://huggingface.co/spaces/openpecha/tibetan-aligner-api) and its wrapper repo [tibetan-aligner-hf-space](https://github.com/OpenPecha/tibetan-aligner-hf-space).

If you want the original pipeline without Docker packaging, see [sebastian-nehrdich/align-tibetan](https://github.com/sebastian-nehrdich/align-tibetan) directly.

## Features

- Sentence-level alignment of Tibetan–English parallel texts using embedding similarity and dynamic programming (Vecalign / FastDTW).
- Pretrained Tibetan–English embedding model ([`buddhist-nlp/bod-eng-similarity`](https://huggingface.co/buddhist-nlp/bod-eng-similarity), ~1.9 GB) downloaded on first run.
- Tab-separated training-pair output (`.train_cleaned`) suitable for feeding directly into MT training.
- Human-readable parallel output (`.org`) for manual inspection.
- One-command Docker build that bakes an end-to-end smoke test into the image.

## Requirements

- **Docker** (tested with Docker Desktop on macOS and Docker on Linux). Docker is the supported install path—see [Troubleshoot](#troubleshoot) for why native `pip install` isn't currently reliable.
- **~3 GB free disk** for the image and the embedding model.
- **Internet access on first run** to download the embedding model from Hugging Face (~1.9 GB).
- **Two input files**: one Tibetan (Unicode, not Wylie) and one English, with one sentence per line and both files UTF-8 encoded.

GPU isn't required. On a modern laptop CPU, the bundled sample (a few dozen lines) runs end-to-end in about 80 seconds, most of which is the first-time model download.

## Get started

### Install tibetan-aligner

Clone the repo and build the Docker image:

```bash
git clone https://github.com/OpenPecha/tibetan-aligner.git
cd tibetan-aligner
docker build -t tibetan-aligner .
```

The build runs a single-line alignment smoke test at the end. If you see a line like `སློབ་དཔོན་བྲམ་ཟེ་རྟ་དབྱངསཀྱིཡོམཛད། \tHello World` in the build output, the pipeline works.

### Align a text pair

Place your Tibetan and English files in a local directory, mount it into the container, and call `align_tib_en.sh` with absolute container paths:

```bash
# Using the sample data shipped in tests/
docker run --rm \
  -v $(pwd)/tests/data:/data \
  -v $(pwd)/output:/output \
  tibetan-aligner \
  /app/align_tib_en.sh /data/text-bo.txt /data/text-en.txt /output
```

`align_tib_en.sh` takes three arguments:

| Position | Argument | Description |
|---|---|---|
| 1 | Tibetan file | UTF-8 text, one sentence per line. Tibetan Unicode (not Wylie). |
| 2 | English file | UTF-8 text, one sentence per line. |
| 3 | Output directory | Created if missing. All artifacts are written here. |

The script prints `[OUTPUT] <path>` on the last line pointing at the canonical artifact.

### Read the output

Three artifacts are produced. `.train_cleaned` is the canonical output for MT training; the others are intermediates.

| File | Format | Purpose |
|---|---|---|
| `<bo-file>.train_cleaned` | TSV—`tibetan<TAB>english` per line | Canonical training pairs. One-to-one aligned sentences only. |
| `<bo-file>.train` | TSV—`tibetan<TAB>english` per line | All aligned spans, including many-to-one groupings. |
| `<bo-file>_<en-file>.org` | Human-readable, paragraph-style | For eyeballing alignment quality. Uses `+$+` as within-source separator and `+!+` as within-target separator; `#` prefixes the target line. |

Example `.train_cleaned` line:

```text
།རྒྱ་གར་སྐད་དུ།	The Chapter on Going Forth Prologue In the language of India, this scripture is called Vinayavastu.
```

### Tuning alignment quality

The three knobs that control the precision/speed trade-off live at the top of `align_tib_en.sh`:

| Variable | Default | Effect |
|---|---|---|
| `number_of_overlays` | `6` | Number of consecutive-sentence concatenations considered during alignment. Higher = more precise, slower. Also caps the largest many-to-one alignment the pipeline produces. |
| `deletion` | `0.06` | Deletion-penalty percentile (0–1). Higher = more willing to skip sentences, less precise. |
| `search_buffer_size` | `50` | Width of the alignment search window (one side). Larger values recover from drift in poorly aligned documents but slow the run. |

Change them by editing the script or by forking the image with your own values. They aren't command-line flags.

### Troubleshoot

| Issue | Solution |
|---|---|
| `pip install -r requirements.txt` fails with `ModuleNotFoundError: No module named 'pkg_resources'` during `pyewts` build | Use the Docker install path. The `pyewts==0.2.0` wheel build imports `pkg_resources` at build time, which `setuptools` 81+ no longer bundles. If you must install natively, create a virtualenv and run `pip install --upgrade pip "setuptools<81" wheel`, then `pip install -r requirements.txt --no-build-isolation`. |
| `ImportError: cannot import name 'cached_download' from 'huggingface_hub'` when building an older commit | `sentence-transformers==2.2.2` imports a function removed in `huggingface_hub` 0.26+. Current `main` pins `huggingface_hub<0.16` to avoid this; if you're on an older commit, add the same pin to `requirements.txt` before building. |
| `rm: cannot remove '…ladder': No such file or directory` (and similar) | Cosmetic. `align_tib_en.sh` unconditionally cleans up intermediate files from a previous run; on a fresh run those files don't exist. Ignore. |
| `WARNI Failed to find overlap=N line "PAD". Will use random vector.` (many repeats) | Expected behavior when the corpus is small enough that Vecalign pads the overlay window with random vectors. Harmless; doesn't affect final output. |
| The model downloads every time you `docker run` | Mount a persistent cache: add `-v $(pwd)/.model-cache:/root/.cache` to your `docker run` command. The first run fills the cache; later runs reuse it. |
| Output paths in the logs look like `/tmp/out//data/text-bo.txt.train_cleaned` | The script prepends the output directory to the full input path. Mount your output directory (`-v $(pwd)/output:/output`) and pass `/output` as the third argument so the files land somewhere you can read. |

Other support resources:

- [Open a GitHub issue](https://github.com/OpenPecha/tibetan-aligner/issues)
- [WeBuddhist developers Discord](https://discord.gg/dUzf9BeZ)
- [WeBuddhist community forum](https://forum.openpecha.org/)

## How it works

The `align_tib_en.sh` script chains four stages:

1. **Embed.** `get_vectors.py` loads `buddhist-nlp/bod-eng-similarity`, expands each input file into overlapping sentence windows (controlled by `number_of_overlays`), and writes one NumPy vector array per file.
2. **Align.** `vecalign.py` runs FastDTW over the two vector arrays to find the best monotonic mapping between source and target sentences. The result is a "ladder": a list of paired sentence indices with a similarity score per pair.
3. **Assemble pairs.** `create_train.py` and `create_train_clean.py` read the ladder and the original files, emitting `.train` (all spans) and `.train_cleaned` (one-to-one pairs only).
4. **Render readable view.** `ladder2org.py` emits the `.org` file for manual inspection.

All four stages come from the upstream `align-tibetan` repo. This fork wraps them in Docker and CI so the pipeline can be run and deployed as a container.

## Attribution

- **Alignment pipeline**: [Sebastian Nehrdich](https://github.com/sebastian-nehrdich), [`align-tibetan`](https://github.com/sebastian-nehrdich/align-tibetan). Every processing script in this repo (`get_vectors.py`, `vecalign.py`, `dp_utils.py`, `dp_core.pyx`, `ladder2org.py`, `create_train.py`, `create_train_clean.py`, `score.py`, `convert_to_wylie.py`, `model_to_hub.py`) is from that repo.
- **Core alignment algorithm**: [Vecalign](https://github.com/thompsonb/vecalign) by Brian Thompson, Apache-2.0. `vecalign.py`, `dp_utils.py`, `dp_core.pyx`, and `score.py` originate there and are inherited through the upstream fork.
- **Embedding model**: [`buddhist-nlp/bod-eng-similarity`](https://huggingface.co/buddhist-nlp/bod-eng-similarity) on Hugging Face.

If you publish work that uses this pipeline, cite Sebastian Nehrdich's upstream repo and Brian Thompson's Vecalign paper.

## Contributing

Because this is a packaging fork, please route contributions to the right place:

- **Alignment algorithm, embedding logic, output format bugs** → open an issue or PR against the upstream [sebastian-nehrdich/align-tibetan](https://github.com/sebastian-nehrdich/align-tibetan).
- **Docker image, CI, requirements pinning, sample data, this README** → open an issue or PR against this repo.

## License

The existing README declares MIT, but no `LICENSE` file is committed in this repo or in the upstream `align-tibetan` repo. The Vecalign-derived files (`vecalign.py`, `dp_utils.py`, `dp_core.pyx`, `score.py`) carry an Apache-2.0 header from Brian Thompson's original project. Treat the overall licence as unresolved until a `LICENSE` file is added—Apache-2.0 on the vecalign-derived files is the only legally grounded statement today.
