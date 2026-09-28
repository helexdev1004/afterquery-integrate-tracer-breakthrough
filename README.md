# Integrate tracer breakthrough curves from packed-column runs

A harbor-format benchmark task, authored for the AfterQuery Experts programme as data
annotation for AI evaluation. Task slug `afterquery/integrate-tracer-breakthrough` — **Science / Hydrology**.

An AI agent gets a container, this brief and nothing else. It works alone for
4 hours. Then a sealed verifier it never sees scores the result `0` or `1`.

## What it asks for

Tracer breakthrough for the packed-column runs

The full brief is in [`instruction.md`](instruction.md).

| | |
| --- | --- |
| **Deliverable** | `/app/breakthrough.csv` |
| **Agent budget** | 4 hours (14400s) |
| **Expert estimate** | 4 hours |
| **Machine** | 1 CPU, 2048 MB RAM, 0 GPU |
| **Verifier budget** | 300s |
| **Tags** | `radiotracer`, `dead-time`, `breakthrough-curve`, `data-cleaning`, `python` |

## Why it is hard

Working a radiotracer breakthrough curve up from a scaler's counts is routine for a soil physicist or a contaminant hydrologist: correct every one-second count for the detector's dead time, take off the background, and integrate for the recovered counts and the mean residence time. The data is synthetic and was made truth first. Eight columns, four soils packed twice, each let an inverse-Gaussian pulse past its own detector; every true event either fired the detector or fell in the dead time after one that did, and the scaler counted what fired for 1500 seconds.

<details>
<summary>Full note</summary>

The dump and the two lists were written afterwards, and the 8 sealed lines are the lab note's own arithmetic on each column's real counts with its real detector's dead time. The figures are exact and nothing is computed at marking time. The lab's note gives every rule the marking uses. Read literally it gets 2 of 8, because three scaler channels plainly were not set up as it says: one holds a running total that wraps at 65536, one holds counts per minute, and one started sixty seconds late. Each is obvious on sight, and each breaks at least one line where all 8 have to be right. What decides the task is the cabling. The detector list gives each detector's scaler channel, and for 4 detectors it is wrong: in two of the four replicate pairs the two cables went to each other's channels. One of the four channels also has an odd set-up; the other three follow the note exactly. The listed reading fails silently - every column still gets a clean pulse and a believable residence time - and gets 4 of the 8 lines wrong, because each pair is one Geiger-Mueller tube with a dead time of 150 microseconds and one scintillator with 5, and at two thousand counts a second that is the difference between a correction of half as much again and none. Nothing first-order shows it. The two columns of a pair hold the same soil at the same flow, so their pulses arrive together, and their recorded peaks stand equally high; the doses are not recorded, so neither the areas nor the peak heights can be set against anything. What differs is second order. A detector that is dead after every count spaces its counts out, so its count in a second scatters less than a Poisson count of the same size: set against the mean of its two neighbours where the pulse is well up, the four tubes' counts scatter at most 0.65 of what Poisson counting would give and the four scintillators' at least 0.96 - once each channel is given to the detector that really fed it. The work-up only ever corrects, subtracts and sums, and never looks at how one second's count stands to the next; the detector type is a column the dead-time correction needs anyway. The note says the types and dead times come from the manufacturers' certificates and that each detector is bolted to its column, so when a channel's scatter disagrees with its detector's type it is the listed channel that is wrong. tests/counting_check.py measures all of this against the sealed figures and refuses to pass otherwise: of 16 readings of the dump the right one gets 8 and the best other 7.

</details>

## How it is verified

The verifier reads /app/breakthrough.csv with the csv module and compares each column's net counts within 0.5 per cent and its mean residence time within 0.5 s with /tests/sealed.json; all 8 lines must be right. None of the agent's code is imported or run. The sealed figures exist only in the verifier image, /tests is mode 700 there, and the agent's container never mounts it.

<details>
<summary>Full note</summary>

The tolerances are fair because the sealed figures are the note's own arithmetic on the very counts the agent reads, so a right reading reproduces them to the last decimal; the half per cent and half second are there for summation order and for a reasonable choice at the edges of the background window. They are far below the cost of any misreading: a column worked up with its pair's dead time is out by between 7 and 14 per cent in net counts, and a block started at the wrong second by sixty seconds. Nothing in the agent's container attests any figure and nothing has to balance; a table made on the cabling as listed is as self-consistent as the right one. The reward is binary and is written to /logs/verifier/reward.txt on every path; an empty container scores 0 and the reference scores 1. The generator is seeded and shipped.

</details>

<details>
<summary>Reference approach (spoiler)</summary>

solution/breakthrough.py reads only /app and writes /app/breakthrough.csv with the standard library. It parses the dump block by block; a block that almost never falls is a running total and is differenced modulo 65536, a block whose values are all multiples of sixty is in counts per minute and is divided, and a block whose header gives a later start is placed at that second. For each channel it then measures, over the seconds with at least five hundred counts, the squared distance of a count from the mean of its two neighbours against one and a half times the count, which is what Poisson counting gives. The four channels that scatter least are taken to be fed by Geiger-Mueller tubes. A channel whose kind contradicts its listed detector's type is given to the detector of the other such channel in the same soil pair. Each column is then worked up with its own detector's dead time exactly as the note says. In the task's own container it gets 8 of 8.

</details>

## Layout

```
instruction.md          the brief - the only thing the agent is told
task.toml               artifacts, timeouts, resources, metadata
environment/            Dockerfile and data for the agent's container
solution/solve.sh       reference solution; scores 1 by construction
tests/                  verifier image, grading logic and ground truth
```

## Running it

With [harbor](https://github.com/laude-institute/harbor) installed and Docker running:

```bash
uv tool install harbor

harbor run -p . -a oracle -e docker   # reference solution - must score 1
harbor run -p . -a nop    -e docker   # nothing happens    - must score 0
```

Those two runs are the calibration the bundle has to satisfy: the oracle run substitutes
`solution/solve.sh` for the agent and must score `1`, the nop run does nothing and must
score `0`. Anything in between means the verifier is measuring the wrong thing.

## Reading this

`tests/` and `solution/` hold the answers. To try the task honestly, read only
`instruction.md` and `environment/`.

---

Part of a set of 38 tasks indexed at
[afterquery-data-annotation](https://github.com/helexdev1004/afterquery-data-annotation).
