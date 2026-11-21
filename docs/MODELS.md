# Models

[← Back to the overview](../README.md) · request/response shapes are in [API.md](API.md).

### Training a worker's model

Each worker directory (`backend/workers/<signal>_<lang>/`) has its own
`train.py`:

```bash
cd backend/workers/urgency_en
pip install -r requirements.txt
# TODO: edit CSV_PATH in train.py to point at a real (text,label) dataset
python train.py        # writes model/model.pkl
```

Rebuild that worker's image (or restart it with a volume-mounted
`model/model.pkl`) to pick up the new model.

#### urgency_en

`urgency_en` has its own training setup. **Train with scikit-learn 1.4.2**,
the same version the worker's `requirements.txt` pins: a joblib model
generally only loads in the scikit-learn version that wrote it. 1.4.2 has no
wheels for Python 3.13+, so use Python 3.11 or 3.12:

```bash
pip install -r backend/workers/urgency_en/requirements-train.txt
python backend/workers/urgency_en/train.py          # -> model/urgency_en.joblib + reports/
python backend/workers/urgency_en/evaluate_gold.py --gold backend/workers/urgency_en/eval/gold_handwritten.csv
```

The deployed model is the frozen `model/urgency_en_v4.joblib` (allowed past the
`.gitignore` rule so it is committed; checksum and thresholds in `model/v4.json`). `train.py` writes `model/urgency_en.joblib`, which is
git-ignored, so a retrain never overwrites v4: freeze a new version explicitly and point
the Dockerfile at it. Training data lives in `data/urgency_en/`; older cleaning-history
files are in `data/urgency_en/archive/`, and the MiniLM experiments in
`backend/workers/urgency_en/research/`.

#### urgency_zh, money_zh, impersonation_zh

The three Chinese workers share **one** cleaning file, `backend/workers/common_zh/text_norm.py`, and **one**
training script, `common_zh/train_zh.py` (each worker's `train.py` just calls it), plus one gold set
(`common_zh/gold_test_zh.csv`) and the probe sentences (`common_zh/probes_zh.csv`); each ZH Dockerfile copies
them into the image. It lowercases Latin words, and only `impersonation_zh` splits words with jieba
(`urgency_zh`/`money_zh` read characters), because that is how the models were trained. Each ZH image
**trains its model during `docker build`** from the CSV in the worker's `data/` folder (~1 min), so the model
always matches the image's scikit-learn version.

Current version (v6):

- **Train = serve.** The model input is built from the raw caller sentence (`text_raw`) with
  `text_norm.prepare`, the function the server uses. A parity check runs at build time and **fails the
  build** if any row differs between training-time and serving-time processing.
- **impersonation_zh** scores every clause on its own (a claim like "我是警察" living inside a long sentence
  used to be diluted) and only a piece that names an authority (police / gov / bank / platform CS / courier
  / telco, the `ORG` lexicon) can score above 0: every impersonation positive in training names one, so an
  introduction alone ("你好，这里是诊所", 0.50; "我是花店的", 1.00) no longer counts. To recognise a new
  authority, add its words to `ORG` in `text_norm.py`.
- **Frozen thresholds** in each worker's `model_config.json` (display/report only; the aggregator uses scores).
- The build **fails** if a canonical test sentence comes out wrong.

Check all three the way the server scores (gold set + probes), after `python train.py` in each folder:

```bash
python tools/zh_eval.py --strict
cd backend/workers/urgency_zh && python predict.py "我是公安局的警官，请马上转账"
```

#### impersonation_en

`impersonation_en` also has its own training setup, and — unlike every other
worker — its trained model is **committed to git**:
`backend/workers/impersonation_en/model/model.pkl` is force-added past the
`.gitignore` rule that excludes `*.pkl`/`*.joblib` elsewhere, so the worker
runs with `docker compose up --build` and no training step. See
[backend/workers/impersonation_en/README.md](../backend/workers/impersonation_en/README.md)
for what it scores, how it works, and why its model file needs
`text_utils.py` importable to unpickle at all.

If you change `data/impersonation_en/*.csv` and need to retrain:

```bash
pip install -r backend/workers/impersonation_en/requirements.txt
python backend/workers/impersonation_en/train.py          # -> model/model.pkl + model_card.json + reports/
python backend/workers/impersonation_en/evaluate_gold.py  # re-check the saved model against the gold set
```

**Train with scikit-learn 1.6.1**, the version pinned in its
`requirements.txt` — a joblib model generally only loads in the scikit-learn
version that wrote it. After retraining, `model/model.pkl` is a tracked file
here (not git-ignored), so commit it like any other changed file.


## Worker models

One line per worker. Test scores are from the running containers for one
scam and one normal sentence. EN: *"This is the police. Transfer the money to the safe
account now and read me the OTP."* / *"Hi, this is the clinic calling to confirm your
appointment on Tuesday."* ZH: *"我是警察，你必须马上把钱转到安全账户，并告诉我验证码。"* /
*"你好，这里是诊所，打电话确认你星期二的预约。"* Each worker returns a raw 0–1
probability; "threshold" is the model's own cut-off.

| Worker | Version | Test score (scam / normal) | Known weak spots |
|---|---|---|---|
| urgency_en | v4, TF-IDF + LR (sklearn 1.4.2), threshold 0.368 | 0.470 / 0.365 | Many false alarms: "no rush", banks' anti-scam advice, normal time talk ("bill due on the 15th"). Misses pressure with no keywords. |
| money_en | money_en_v2c, word TF-IDF (1-2 grams, brand names masked) + LR, Platt-calibrated (sklearn 1.6.1), threshold 0.5 | 0.999 / 0.000 | Scores whether the speaker **asks** for money, card/bank details or a code (rule: `backend/workers/money_en/LABELING.md`); a money word alone ("never share your OTP": 0.002) scores low. SG gold: ROC-AUC 0.938, F1 0.75, false alarms 7% (v1: 0.81 / 0.61 / 35%); 200-sentence test set ROC-AUC 0.977, F1 0.83; FTC robocalls ROC-AUC 0.954 but F1 only 0.40. A routine bill reminder can sit near the threshold. |
| impersonation_en | impersonation_en_v5, TF-IDF word + char n-grams + LR, calibrated (sklearn 1.6.1), reporting threshold 0.67 | 0.990 / 0.003 | Trained mostly on synthetic, advisory-based rows; performance on real calls is less certain. Can score a plain greeting high: 0.90 for "Hello, good afternoon, am I speaking with Mr Tan?" |
| urgency_zh | v6, trained at image build (char TF-IDF + LR, calibrated, C=100), threshold 0.34 | 0.472 / 0.016 | Under 0.5 on the scam test sentence alone: urgency words are diluted when one sentence also carries police + OTP (the call is still SCAM via money + impersonation; BERT reads the context). Gold F1 0.65, AUC 0.86. |
| money_zh | v6 (char, C=100), threshold 0.16 | 0.996 / 0.011 | Gold F1 0.91, AUC 0.998. |
| impersonation_zh | v6 (jieba words, C=100, clause scoring + authority gate), threshold 0.17 | 1.000 / 0.000 | Only authorities in the `ORG` lexicon count. Still flags anti-scam *advice* that names an authority ("有人冒充星展银行的客服，大家要小心"). Gold F1 0.94, AUC 0.98. |
| bert | DistilBERT multilingual (cased), fine-tuned on caller-line windows, ONNX int8, temperature-calibrated | see `backend/workers/bert/README.md` | Second opinion only (cascade); strong on windows of context (F1 0.986), poor on a single isolated sentence (F1 0.086) — by design, since detectors handle single sentences. |


## BERT second opinion

The six fast workers score every caller line in milliseconds but read one sentence at a time and know only
their own signal. `backend/workers/bert` reads the call's **last 5 caller lines together** and returns
P(scam), so it can tell a genuine clinic call apart from a real threat using context. It is slower, so it is
a **cascade**: the aggregator asks it only when the fast vote would be UNSURE, high, money-rule, or "not
enough data"; final = average of the fast vote and BERT. If BERT is slow (> `BERT_TIMEOUT_MS`), down or not
deployed, every line is decided exactly as without it.

- Text: the gateway keeps each call's last `BERT_CONTEXT_LINES` caller lines in RAM and sends them with the
  aggregate request; the aggregator forwards them to BERT only when it asks, and never stores or logs them
  (Redis stays text-free).
- Model: `notebooks/bert_cascade.ipynb` (Colab, free T4) builds the training windows, fine-tunes
  `distilbert-base-multilingual-cased`, fits the temperature, exports ONNX int8 and writes the 3 files the
  pod needs (`model.onnx`, `tokenizer.json`, `bert_config.json`). They are not in git (too big) — **the
  build downloads them automatically** and checks the SHA256, so `docker compose up -d --build` is all you
  need. On a laptop: `python backend/workers/bert/fetch_model.py`.
- Evaluation: `python tools/eval_cascade.py` replays labelled calls through the running stack and reports
  fast-only vs cascade (and BERT alone), the share of lines that needed BERT, and latency.

