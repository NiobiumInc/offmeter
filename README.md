# Private Demand-Response Eligibility

_Created with the Niobium FHE Application Design assistant (FHEanna) — v0.13.0._

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
programs — the ones that pay you a rebate for easing off power during peak-demand
moments. The catch has always been that scoring you means letting the utility
**see** your usage, and that usage is a minute-by-minute picture of your life.

This application removes the catch. The utility scores your eligibility **on your
encrypted usage** and never sees the usage itself. Your household sends an
encrypted copy of its 24-hour electricity profile; the utility runs its scoring
model directly on that encrypted data and sends back an encrypted result that
**only you can unlock**. The utility learns whether you qualify — and nothing
else about your day.

**What it does, end to end.** From your encrypted hourly readings it works out —
still encrypted — the two things the model needs (your total daily use and your
evening-peak use, 5–9pm), runs the utility's eligibility model, and returns an
encrypted probability. You unlock it at home and get a simple eligible /
not-eligible answer. At no point does your usage, the intermediate totals, or the
score exist in the clear anywhere but your own device.

## Is it practical?

Measured on a laptop CPU, scoring one household:

| | |
|---|---|
| Time to score one encrypted household | **~10 seconds** |
| Memory on the utility's side | **~0.6 GB** |
| Data per request | **~13 MB** up, **~7 MB** back |
| One-time key setup (per household) | **~337 MB** |
| Accuracy cost of the encryption | **negligible** — the encrypted answer matches the ordinary (unencrypted) computation to ~7 decimal places |
| Model quality (labeled synthetic test set) | **accuracy ≈ 0.77, ROC-AUC ≈ 0.80** at a ~37%-eligible base rate |

**Model quality, and an honest caveat.** The model here is genuinely *fitted*, and
its quality is genuinely measured. Each synthetic household carries an
**independent** eligibility label drawn from a fixed latent process (base rate
**~37% eligible** — not 50/50 by construction), and the utility's logistic model is
trained to predict that label. On a held-out 400-household test set the fitted
model reaches **accuracy ≈ 0.77 and ROC-AUC ≈ 0.80** (precision / recall / F1
≈ 0.76 / 0.54 / 0.63 on the eligible class) — a real, imperfect result, since the
labels carry noise and the truth is nonlinear. The data is still **synthetic**
(generated from a fixed seed and a known latent function), so this demonstrates the
private-scoring *pipeline* and gives a real but synthetic-domain accuracy — **not** a
real-world-validated demand-response predictor. A utility would drop in its own
model of the same shape, trained on real households. The encryption itself adds
essentially no error; the model's real-world accuracy is the utility's to establish.

## Run it

You need **Docker** and nothing else — no OpenFHE, no FHE libraries, no compilers
installed locally. Everything runs inside one image.

**Set up (one time).** Get the FHE-dev image, then build the app. Once the image
is published you can pull it; until then, build it from source. The build context
lives in the `niobium-skills` repo (not in this app repo), so clone that first; the
build then clones `niobium-client` and compiles the instrumented OpenFHE from
source (about an hour the first time):

```bash
# 1. the toolchain image
# Once published:
docker pull ghcr.io/niobiuminc/fhe-dev:v0.13.0

# Or build from source (needs the skill repo checked out):
git clone git@github.com:NiobiumInc/niobium-skills.git
docker build -t ghcr.io/niobiuminc/fhe-dev:v0.13.0 \
  niobium-skills/skills/fhe-application-design/environment

# 2. the app's four programs (key generation, encrypt, score, decrypt)
./run-in-container.sh "cmake -S . -B build \
    -DCMAKE_PREFIX_PATH='/opt/niobium-client/vendor/lib/niobium-client;/opt/niobium-client/vendor/lib/openfhe' \
    && cmake --build build -j"
```

**1 — Run it on the Niobium Fog.** The Fog is the accelerated platform these apps
run on, and it's the default — a bare run targets it:

```bash
./run-in-container.sh "./run_test.sh"
```

- **Have an account?** Sign in to mint a key:
  `docker run --rm -it -v "$HOME/.fog":/root/.fog ghcr.io/niobiuminc/fhe-dev:v0.13.0 fog login`
- **New to the Fog?** Request access → **https://console.niobium.co/request-account**

**2 — Validate locally, no account needed.** Run the same encrypted computation on
your own machine and check it against the plain result:

```bash
./run-in-container.sh "./run_test.sh --cpu"    # plain OpenFHE, on your CPU
./run-in-container.sh "./run_test.sh --sim"    # the Fog code path, run locally
```

- **`--cpu`** runs the encrypted circuit directly with OpenFHE on your machine —
  the quickest correctness check.
- **`--sim`** records the exact trace the Fog would execute and replays it through a
  local simulator (`fhetch_sim`), so you exercise the *Fog code path* offline — plus
  a free bit-identical check that the replayed trace matches the plain OpenFHE run.
  It's the closest thing to a Fog run without an account.

Either way `run_test.sh` leads with the fitted model's **real quality against the
true labels** (accuracy, ROC-AUC, precision/recall/F1, and the stated base rate),
then the encryption-fidelity PASS/FAIL gate (encrypted run vs the faithful twin),
then the timings and data sizes above — plus a check that a household's secret key
never reaches the server (it refuses to start if it finds one).

**3 — See it as a real client/server split** — two separate processes, the secret
key living only on the client, only ciphertext crossing between them:

```bash
./run-in-container.sh '
  set -e
  ./build/dr_keygen client_home                       # client makes keys; secret key stays here
  mkdir -p server_home
  cp client_home/{cc,pk,mk,rk}.bin model/model.txt server_home/   # server gets public/eval keys + model — no secret key
  ./build/dr_encrypt client_home data/test_inputs.csv 0 client_home/ct_x.bin
  cp client_home/ct_x.bin server_home/                # the wire: ciphertext only
  ./build/dr_server server_home --cpu                             # utility computes; never sees plaintext
  cp server_home/ct_result.bin client_home/           # the wire back: still encrypted
  ./build/dr_decrypt client_home client_home/ct_result.bin        # client unlocks the answer
'
```

The `server_home/` folder is safe to place on an untrusted machine as-is — swap
the `cp` steps for `scp`/HTTP to a remote host to run it truly split.

## Comparing to the cleartext model

There are three versions of the same scorer, so you can see exactly where any
error comes from:

- **Cleartext reference** — the exact model (true sigmoid), computed in the open.
  The ground-truth answer.
- **Faithful twin** — the same model, but using the polynomial approximation the
  encrypted circuit uses. It predicts, in the clear, exactly what the encrypted
  run will output.
- **Encrypted run** — what `run_test.sh` produces under encryption.

Run the reference and twin (pure Python, a few seconds — no encryption):

```bash
./run-in-container.sh "python3 model/make_model_and_data.py && python3 model/twin.py"
```

This prints the fitted model's **quality against the true labels** (accuracy,
ROC-AUC, precision/recall/F1, base rate) and the **approximation cost** (how far the
encryption-friendly twin drifts from the exact model — here, 100% decision
agreement, max probability error ~5e-4), and writes `data/reference_outputs.csv`
(cleartext) and `data/twin_outputs.csv` (twin). `run_test.sh` then checks the
encrypted output against the twin, so:

> **total error = approximation cost** (twin vs reference) **+ encryption cost**
> (encrypted vs twin, typically ~1e-7).

If the twin already disagreed with the reference, that would be a *modeling*
limitation, not an encryption one — this is where you'd catch it.

## Under the hood

The heavy math runs on ciphertext using the CKKS homomorphic-encryption scheme
(via OpenFHE); the two usage aggregates are computed inside the encrypted
computation, and the whole circuit is shallow (no bootstrapping). Design
rationale, measured results, and the security/threat model are in
`docs/`.
