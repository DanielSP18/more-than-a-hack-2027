# Tiki-Data: ball possession from tracking data

For every frame of a full professional football match (FC Barcelona v Athletic Club, LaLiga), this
pipeline says who has the ball: home, away, or nobody because the ball is dead. It works only from
automatic player and ball tracking plus automatic tactical tags. There is no video model and no
training data.

Team Tiki-Data built it for More than a Hack 2027, a football analytics challenge run by Metrica
Sports, FC Barcelona and Mobile World Capital Barcelona, and submitted it on November 17, 2026.
The full write-up (design, rules, what we tried and dropped, limitations) is in
[Tiki-Data.md](Tiki-Data.md).

## How it works

```
 tracking ─┬─► ball control: who has the ball right now            ─┐
           ├─► dead balls + restarts (tracking + tactical tags)    ─┼─► product of experts ─► HMM ─► one label per frame
           └─► team shape: who is attacking when the ball is lost  ─┘
```

Each piece writes a probability for home, away and dead ball on every frame, or abstains. A glue
step combines them, and a hidden Markov model with minimum durations decodes one label per frame.
The scoring metric rewards recovering whole possessions, so the decoder is built so that a few
noisy frames can't create a spurious possession. Every constant is set from football reasoning or
label-free measurements of the tracking, and the reason is written next to it in the code.

The hard part is that the tracker loses the ball on 41% of the frames that count. Team shape
covers most of those stretches: a team with the ball spreads out, and a team without it compacts.

## Results

Only the first 5 minutes of the match have public ground truth (17 possessions). We used them to
check the pipeline, not to tune it. The full match is scored privately by the organisers.

| | frame-level macroF1 | possession-level spellF1 |
|---|---|---|
| organisers' baseline | 0.621 | 0.326 |
| Tiki-Data | 0.746 | 0.490 |

Over the full match the output has 249 possessions with a mean of 16.1 s. Its main known weakness
is under-detecting stoppages such as fouls, where play stops without the ball going out.

## Reproducing the output

You need Python 3.11 and the match data (see below).

```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python src/glue.py      # -> out/Tiki-Data.csv, validated, sha256 printed
```

The pipeline is deterministic and runs in under a minute. The expected sha256 of
`out/Tiki-Data.csv` is `cdc8f16fdc7d6e896765c342275c8dd25e35fa9413731d07050e1293a315db54`.

## The data

The match data was provided by the competition organisers. FC Barcelona supplied the match, and
Metrica Sports' automatic pipeline produced the tracking, tactical tags and camera geometry. It was
licensed to participants for the competition only, so it is not in this repository and is not
public. With a copy, unpack it into `data/` using the layout in [data/README.md](data/README.md).

## Code

| file | what it does |
|---|---|
| [src/glue.py](src/glue.py) | the one command: runs the pieces, combines them, decodes, writes the output |
| [src/load.py](src/load.py) | parses the tracking into a cached per-frame table |
| [src/approach1.py](src/approach1.py) | ball-control evidence |
| [src/approach3.py](src/approach3.py) | dead-ball stretches and restarts, from tracking and cleaned tactical tags |
| [src/approach4.py](src/approach4.py) | team-shape evidence for frames without a ball |
| [src/hmm.py](src/hmm.py) | HMM with minimum durations, Viterbi decoding |
| [src/submission.py](src/submission.py) | file formats and validation |
| [src/phases.py](src/phases.py) | tactical-phase evidence: tried, measured, and left out |
| [analysis/masked_check.py](analysis/masked_check.py) | label-free check of how well each piece carries possession when the ball is hidden |

`evaluation/` and `starter/` are the organisers' scorer and starter code. The challenge's own
documentation (task, data, metric, rules) is kept in [CHALLENGE.md](CHALLENGE.md),
[DATA.md](DATA.md), [EVALUATION.md](EVALUATION.md) and [RULES.md](RULES.md).

## Licence

The code is MIT licensed ([LICENSE](LICENSE)). The licence does not cover the match data.
