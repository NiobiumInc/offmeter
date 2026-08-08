# Offmeter

_Private demand-response eligibility scoring under FHE: the utility scores your
household on its **encrypted** 24-hour smart-meter profile and never sees the usage
itself._

_Created with the [Niobium FHE Application Design assistant (FHEanna)](https://github.com/NiobiumInc/niobium-skills) (v0.13.0)._

## Initial Prompt

_A utility offers a rebate to households that are good candidates for easing off
power during peak hours, and wants to score eligibility without seeing anyone's
usage. Each household sends its encrypted 24-hour electricity-usage profile; the
utility's model runs on the encrypted data and returns an encrypted eligibility
score that only the household can read. Working entirely on the encrypted profile,
the model should derive the two things it needs (total daily use, and evening-peak
use for 5–9pm) and then score eligibility. Build a small eligibility model and
keep the computation shallow._

## Overview

Your electricity meter is a diary. Its hour-by-hour readings show when you wake
up, when you're home, when you cook, when you travel. Utilities want exactly that
usage pattern to decide which households to invite into **demand-response**
programs, the ones that pay you a rebate for easing off power during peak-demand
moments. Scoring you has meant letting the utility
**see** your usage, and that usage is a minute-by-minute picture of your life.

This application removes the catch. The utility scores your eligibility **on your
encrypted usage** and never sees the usage itself. Your household sends an
encrypted copy of its 24-hour electricity profile; the utility runs its scoring
model directly on that encrypted data and sends back an encrypted result that
**only you can unlock**. The utility learns whether you qualify, and nothing
else about your day.

**What it does, end to end.** From your encrypted hourly readings it works out
(still encrypted) the two things the model needs (your total daily use and your
evening-peak use, 5–9pm), runs the utility's eligibility model, and returns an
encrypted probability. You unlock it at home and get a simple eligible /
not-eligible answer. At no point does your usage, the intermediate totals, or the
score exist in the clear anywhere but your own device.

## Evaluation data & features

> **Synthetic data: proof of concept only.** The dataset described here is
> **synthetically generated** by a seeded script (`model/make_model_and_data.py`)
> purely to demonstrate the encrypted pipeline. It is **not real, personal, or
> proprietary data**: no actual household, smart-meter reading, or utility record is
> used. For a real deployment, swap in the utility's model and real labeled profiles;
> the same protocol and guarantees carry over.

**Evaluation set:** 400 held-out households (`data/test_inputs.csv`), fit on a
200-household training split. Each household is described by its private **24-hour
electricity-usage profile** (one kWh value per hour). The pipeline outputs an
**eligibility probability** (sigmoid), thresholded at 0.5 → *eligible / not eligible*
for a demand-response program; the ground-truth label is drawn from an *independent*
latent process (base rate ≈ 1/3 eligible), so the fitted model is a genuine, imperfect
predictor.

**Per-household features (encrypted and sent to the utility):**

| Feature | Description | Count |
|---|---|---|
| `x[0] … x[23]` | Hourly electricity consumption (kWh), hour 0–23 | 24 values |

Two aggregates the **circuit derives** from the same 24 values (not extra inputs):

| Derived value | Definition |
|---|---|
| `total_daily` | Sum of all 24 hourly values (total daily kWh) |
| `evening_peak` | Sum of hours 17–21 (evening-peak load, the load-shedding signal) |

The model is a logistic regression over the 24 hourly weights plus a total-use weight
and an evening-peak weight, favouring evening-peak-concentrated consumption (good
load-shedding candidates). Each household encrypts its **own** profile independently
(24 of 32,768 slots used per record).

## Is it practical?

Measured on a laptop CPU, scoring one household:

| | |
|---|---|
| Time to score one encrypted household | **~10 seconds** |
| Memory on the utility's side | **~0.6 GB** |
| Data per request | **~13 MB** up, **~7 MB** back |
| One-time key setup (per household) | **~337 MB** |
| Accuracy cost of the encryption | **negligible**: the encrypted answer matches the ordinary (unencrypted) computation to ~7 decimal places |
| Model quality (labeled synthetic test set) | **accuracy ≈ 0.77, ROC-AUC ≈ 0.80** at a ~37%-eligible base rate |

**Model quality, and an honest caveat.** The model here is genuinely *fitted*, and
its quality is genuinely measured. Each synthetic household carries an
**independent** eligibility label drawn from a fixed latent process (base rate
**~37% eligible**, an uneven split by construction), and the utility's logistic model is
trained to predict that label. On a held-out 400-household test set the fitted
model reaches **accuracy ≈ 0.77 and ROC-AUC ≈ 0.80** (precision / recall / F1
≈ 0.76 / 0.54 / 0.63 on the eligible class). This is a real, imperfect result, since the
labels carry noise and the truth is nonlinear. The data is still **synthetic**
(generated from a fixed seed and a known latent function), so this demonstrates the
private-scoring *pipeline* and gives a real but synthetic-domain accuracy. It is **not** a
real-world-validated demand-response predictor. A utility would drop in its own
model of the same shape, trained on real households. The encryption itself adds
essentially no error; the model's real-world accuracy is the utility's to establish.

## Run it

You need **Docker** and nothing else: no OpenFHE, no FHE libraries, no compilers
installed locally. Everything runs inside one image.

**Set up (one time):** install the skill, get the container, then build the app.

**1. Install the skill.** Fetch the `fhe-application-design` skill from GitHub into the
repository-root `.claude/skills/` and `.agents/skills/` (both gitignored). A single
install at the repo root is shared by every app here, so this works from inside the app too:

```bash
make install-skill
```

**2. Get the FHE-dev image** (`ghcr.io/niobiuminc/fhe-dev:v0.13.0`). Pull the prebuilt
image from the GitHub Container Registry (ghcr), or build it from the skill (the first
build clones `niobium-client` and compiles the instrumented OpenFHE, about an hour the
first time):

```bash
# Pull the prebuilt image from ghcr (once published):
docker pull ghcr.io/niobiuminc/fhe-dev:v0.13.0

# Or build it (a) from a fresh clone of the skill repo:
git clone https://github.com/NiobiumInc/niobium-skills
docker build -t ghcr.io/niobiuminc/fhe-dev:v0.13.0 \
  niobium-skills/skills/fhe-application-design/environment

# Or build it (b) from the skill installed in step 1 (shared at the repo root):
docker build -t ghcr.io/niobiuminc/fhe-dev:v0.13.0 \
  ../.claude/skills/fhe-application-design/environment
```

**3. Build the app's four programs** (key generation, encrypt, score, decrypt):

```bash
./run-in-container.sh "make build"
```

**4. Generate the twin ledgers the run gates on.** `model/twin.py` writes
`data/twin_outputs.csv` (the faithful-twin predictions) and `data/noise_tolerance.txt`
(the decision-margin tolerance). Those are the files `run_test.sh` checks the encrypted output
against, in **every** mode including a Fog run. They are **gitignored** (regenerated, not
committed), so a fresh clone must produce them once before the first run:

```bash
./run-in-container.sh "python3 model/twin.py"
```

The model and datasets (`model/model.txt`, `data/test_inputs.csv`, `data/test_labels.csv`)
are **committed**, so `model/make_model_and_data.py` does *not* need to be re-run; only
`twin.py` above. (`make clean` leaves `data/` untouched, so this step is genuinely one-time
unless you delete the ledgers or change the model.)

**1. Run it on the Niobium Fog.** The Fog is the accelerated platform these apps
run on, and it's the default; a bare run targets it:

```bash
./run-in-container.sh "./run_test.sh"
```

- **Have an account?** Sign in to mint a key:
  `docker run --rm -it -v "$HOME/.fog":/root/.fog ghcr.io/niobiuminc/fhe-dev:v0.13.0 fog login`
- **New to the Fog?** Request access → **https://console.niobium.co/request-account**

**2. Validate locally, no account needed.** Run the same encrypted computation on
your own machine and check it against the plain result:

```bash
./run-in-container.sh "./run_test.sh --cpu"    # plain OpenFHE, on your CPU
./run-in-container.sh "./run_test.sh --sim"    # the Fog code path, run locally
./run-in-container.sh "./run_test.sh --sim-full" # real math + bit-exact ring-level identity check, local
```

- **`--cpu`** runs the encrypted circuit directly with OpenFHE on your machine,
  the quickest correctness check.
- **`--sim`** records the (hollow) trace the Fog would execute and replays it through a
  local simulator (`fhetch_sim`), then twin-compares the result, so you exercise the
  *Fog code path* offline. It's the closest thing to a Fog run without an account.
- **`--sim-full`** is `--sim` but records real math instead of hollow, adding a
  bit-exact ring-level check that the replayed trace matches the plain OpenFHE run;
  run it alongside `--sim` to surface any hollow-recording divergence: the thorough
  local ground-truth run, still all local with no Fog account.

Either way `run_test.sh` leads with the fitted model's **real quality against the
true labels** (accuracy, ROC-AUC, precision/recall/F1, and the stated base rate),
then the encryption-fidelity PASS/FAIL gate (encrypted run vs the faithful twin),
then the timings and data sizes above, plus a check that a household's secret key
never reaches the server (it refuses to start if it finds one).

**3. See it as a real client/server split**, with two separate OS processes talking
only over HTTP, the secret key living only on the client, only ciphertext crossing
between them:

```bash
./run-in-container.sh "python3 harness/demo_two_process.py"
```

This stands the household **client** and the untrusted utility/compute **provider**
up as separate processes under `run_demo/` (a `client_home/` that holds `sk.bin`,
a `server_home/` that never does). The client runs `dr_keygen`, ships only the
crypto context + public/eval keys + `model.txt` at setup, then per household
encrypts (`dr_encrypt`), uploads the ciphertext, and downloads and decrypts
(`dr_decrypt`) the result; the provider runs `dr_server --cpu` behind
`server_guard.sh` and logs **byte counts only**. It runs a negative test first: a
secret key planted in the server home makes the server refuse to start (exit 13).

The `run_demo/server_home/` folder is safe to place on an untrusted machine as-is.
Point the client at it with `SERVER_URL=http://host:port` to run it truly split.

Prefer plain shell? The same split done by hand:

```bash
./run-in-container.sh '
  set -e
  ./build/dr_keygen client_home                       # client makes keys; secret key stays here
  mkdir -p server_home
  cp client_home/{cc,pk,mk,rk}.bin model/model.txt server_home/   # server gets public/eval keys + model, no secret key
  ./build/dr_encrypt client_home data/test_inputs.csv 0 client_home/ct_x.bin
  cp client_home/ct_x.bin server_home/                # the wire: ciphertext only
  ./server_guard.sh ./build/dr_server server_home --cpu          # utility computes; guard refuses if sk present
  cp server_home/ct_result.bin client_home/           # the wire back: still encrypted
  ./build/dr_decrypt client_home client_home/ct_result.bin        # client unlocks the answer
'
```

## Comparing to the cleartext model

There are three versions of the same scorer, so you can see exactly where any
error comes from:

- **Cleartext reference**: the exact model (true sigmoid), computed in the open.
  The ground-truth answer.
- **Faithful twin**: the same model, but using the polynomial approximation the
  encrypted circuit uses. It predicts, in the clear, exactly what the encrypted
  run will output.
- **Encrypted run**: what `run_test.sh` produces under encryption.

Run the reference and twin (pure Python, a few seconds, no encryption):

```bash
./run-in-container.sh "python3 model/make_model_and_data.py && python3 model/twin.py"
```

This prints the fitted model's **quality against the true labels** (accuracy,
ROC-AUC, precision/recall/F1, base rate) and the **approximation cost** (how far the
encryption-friendly twin drifts from the exact model; here, 100% decision
agreement, max probability error ~5e-4), and writes `data/reference_outputs.csv`
(cleartext) and `data/twin_outputs.csv` (twin). `run_test.sh` then checks the
encrypted output against the twin, so:

> **total error = approximation cost** (twin vs reference) **+ encryption cost**
> (encrypted vs twin, typically ~1e-7).

If the twin already disagreed with the reference, that would be a *modeling*
limitation rather than an encryption one, and this is where you'd catch it.

## Under the hood

The heavy math runs on ciphertext using the CKKS homomorphic-encryption scheme
(via OpenFHE); the two usage aggregates are computed inside the encrypted
computation, and the whole circuit is shallow (no bootstrapping). Design
rationale, measured results, and the security/threat model are in
`docs/`.

## Clean up

When you are done, remove the build tree and every per-run artifact:

```bash
make clean
```

This deletes the compiled `build/` tree, the per-mode run homes (`run_cpu`,
`run_sim`, `run_sim-full`, `run_fog`, `run_demo`), the `client_home`/`server_home`
provisioning dirs, and the generated FHETCH trace directories (`dr_server_workload_*`,
`nbcc_fhetch_replay_source_*`). `clean` lists these targets **explicitly** and never
globs `run_*`, so it cannot delete `run_test.sh`. The committed inputs under `data/`
are left untouched, so a later run does not need to regenerate them.
