# The network: data, training, and provenance

zigqueen's evaluation is a neural network trained by the author on a single
desktop PC. Because "trained on Stockfish data" can mean anything from
"borrowed a public dataset" to "copied someone's network", this page states
exactly what was and was not used.

| | |
|---|---|
| **Training data** | Publicly published Stockfish training datasets (see below) |
| **Trainer** | [bullet](https://github.com/jw1912/bullet), extended by the author for this feature set |
| **Architecture** | Same public family as modern NNUE engines; dimensions, feature set and format are the author's own |
| **Weights** | Trained from random initialisation. Never initialised from, fine-tuned from, or distilled against another engine's network |
| **Hardware** | One RTX 4090; a full run is ~58 hours |
| **Engine inference code** | Written from scratch in Zig; no code from other engines |

## Training data

The Stockfish project and its contributors publish their NNUE training data
openly. zigqueen's current network was trained on a 27-component selection of
those published sets — roughly 280 GiB, about 104 billion positions — drawn
from:

- [`official-stockfish/master-binpacks`](https://huggingface.co/official-stockfish/master-binpacks)
  — the data used for Stockfish's own master networks
- [`linrock/test80-2023`](https://huggingface.co/datasets/linrock/test80-2023),
  [`linrock/test80-2024`](https://huggingface.co/datasets/linrock/test80-2024)
  and the corresponding test78/test79/test60 collections

These are played-out self-play games whose positions carry search-derived
evaluations and game results. Using them is a deliberate choice: they are
published for reuse, they are far better than anything a single person could
generate on one PC, and they let a small project spend its compute on
architecture and method rather than on data generation.

What this data is *not* is a network. It is positions and labels; turning
those into a strong evaluation function is the part that has to be done, and
done well, by the engine author.

## Trainer

Stockfish trains with `nnue-pytorch`. zigqueen uses **bullet**, an independent
open-source NNUE trainer, with:

- a training recipe written for this project (schedule, learning-rate decay,
  WDL blending, evaluation scaling, bucket layout)
- an extension the author wrote to give bullet support for zigqueen's threat
  feature set, which it did not previously have

## Architecture

Nearly every strong NNUE engine today builds on ideas Stockfish published:
king-bucketed HalfKA-style inputs, a PSQT head, and a bucketed layer stack.
zigqueen belongs to that family. The specifics are its own:

- 1536-wide accumulator, 8 king buckets with horizontal mirroring
- a self-designed, deliberately small threat feature set: 7,680 features
  (10 attacker keys x 64 squares x 12 target relations)
- 8 material output buckets with a 16/32 layer stack
- the engine's own quantisation scheme and network file format (`ZQB8`)
- inference written from scratch and validated bit-for-bit against an
  independent reference implementation of the trainer's arithmetic

Full detail is in [ARCHITECTURE.md](ARCHITECTURE.md).

## Why this does not amount to "a Stockfish network"

The honest empirical answer: if public data plus a published architecture
family plus a trainer were enough to reproduce Stockfish's evaluation, then
everyone doing it would be at Stockfish's strength — and nobody is.

zigqueen's network is 29.5 MB against Stockfish's much larger ones, and in the
author's own 1,620-game testing zigqueen scores about 27% against Stockfish
17.1 — roughly 180 Elo below it (see [STRENGTH.md](STRENGTH.md)). The two
engines evaluate positions differently, search differently, and play
differently.

The engine's source contains no code from Stockfish or any other engine: no
copied functions, no mechanical translations, no lifted tuning constants. See
[CLEAN_ROOM_RULES.md](../CLEAN_ROOM_RULES.md) for the rules the project was
developed under.

## Reproducibility

The network shipped in the binary is the artefact of the process described
above, and the process is repeatable: the datasets are public, the trainer is
public, and the recipe is described here and in the repository. Training it
again on the same data with the same recipe will produce an equivalent
network, and there is nothing in it that cannot be rebuilt from public
inputs by anyone willing to spend the GPU time.
