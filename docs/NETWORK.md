# The network: data, training, and provenance

This page documents how zigqueen's evaluation network was produced, in enough
detail that a reader can judge it for themselves.

| | |
|---|---|
| **Training data** | Publicly published Stockfish training datasets (details below) |
| **Trainer** | [bullet](https://github.com/jw1912/bullet), extended by the author for this feature set |
| **Architecture** | King-bucketed HalfKA inputs + threat features + PSQT head + bucketed layer stack; dimensions and feature set are the author's |
| **Weights** | Trained from random initialisation |
| **Training run** | 2,200 superbatches (~220 billion positions seen), one RTX 4090, ~30 hours |
| **Engine inference** | Written from scratch in Zig; validated bit-for-bit against an independent reference |

## Training data

The Stockfish project and its contributors publish their NNUE training data
openly. zigqueen's network is trained on a 27-component selection of those
sets — about 280 GiB, roughly 104 billion usable positions — drawn from:

- [`official-stockfish/master-binpacks`](https://huggingface.co/official-stockfish/master-binpacks)
  — the data used for Stockfish's own master networks
- [`linrock/test80-2023`](https://huggingface.co/datasets/linrock/test80-2023),
  [`linrock/test80-2024`](https://huggingface.co/datasets/linrock/test80-2024),
  and the corresponding test78 / test79 / test60 collections

The data consists of played-out self-play games; each position carries a
search-derived evaluation and the game's result. Components were checked for
unit consistency before use (the evaluation scale differs between dataset
vintages) and interleaved so that no single vintage dominates any part of the
schedule.

Using published data is a deliberate choice: generating a comparable corpus
on one PC is not realistic, and it leaves the project's compute for
architecture and method instead.

## Trainer

Stockfish trains with `nnue-pytorch`. zigqueen uses **bullet**, an independent
open-source NNUE trainer, with:

- an extension written for this project that teaches bullet zigqueen's threat
  feature set (which it did not support)
- a training recipe written for this project: 100M positions per superbatch,
  cosine learning-rate decay from 1e-3, a 0.4 win-draw-loss blend between
  game results and search evaluations, and an evaluation scale chosen to
  match the engine's internal units

## Architecture

zigqueen belongs to the same family as most modern NNUE engines —
king-bucketed HalfKA-style inputs, a PSQT head, a bucketed layer stack. The
specific configuration is:

- 1536-wide accumulator, 8 king buckets with horizontal mirroring
- a self-designed threat feature set of 7,680 features
  (10 attacker keys x 64 squares x 12 target relations)
- 8 material output buckets, each with a 16 / 32 layer stack
- the engine's own quantisation scheme and file format (`ZQB8`), ~29.5 MB

[ARCHITECTURE.md](ARCHITECTURE.md) covers the inference side in detail.

## The training run

The shipped network was trained from random initialisation for 2,200
superbatches of 100M positions — about 220 billion positions processed, or
roughly two passes over the dataset — taking about 30 hours on one RTX 4090.

Before any network is allowed to cost testing time it must pass a sanity gate:
colour-mirrored positions must evaluate to ~0, and material must be ordered
correctly (queen > rook > bishop ≈ knight > pawn). Networks that fail are
discarded without playing a single game. Candidate networks that pass are then
judged by SPRT matches against the current best; several have been trained,
gated, and rejected.

## What the weights are not

The network was not initialised from another engine's network, not fine-tuned
from one, and not distilled by querying a running engine. The engine's source
code contains no code from Stockfish or any other engine — no copied
functions, no mechanical translations, no lifted tuning constants.
[CLEAN_ROOM_RULES.md](../CLEAN_ROOM_RULES.md) states the rules the project was
developed under.

## Reproducibility

Every input to this process is public: the datasets, the trainer, and the
recipe described above. Anyone willing to spend the GPU time can train an
equivalent network from the same ingredients.
