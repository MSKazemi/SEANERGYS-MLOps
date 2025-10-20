# Repo layout

```
mlops-mvd/
├─ README.md
├─ requirements.txt
├─ Dockerfile
├─ .gitignore
├─ .gitlab-ci.yml
├─ data/                     # tiny sample CSV kept in repo for MVD
│  └─ toy.csv
├─ src/
│  ├─ train.py               # trains + saves model + metrics.json
│  ├─ evaluate.py            # asserts thresholds, emits report
│  ├─ infer.py               # CLI inference
│  └─ serve.py               # FastAPI app with /predict and /metrics
├─ models/
│  └─ latest/                # CI writes model.pkl + model_card.md here
└─ tests/
   └─ test_infer.py          # smoke test uses models/latest/model.pkl

```

---

# Minimal code

**requirements.txt**

```
pandas==2.2.2
scikit-learn==1.5.1
fastapi==0.114.0
uvicorn==0.30.6
prometheus-client==0.20.0
joblib==1.4.2

```

**src/train.py**

```python
import json, os, joblib, pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import f1_score, accuracy_score

DATA = "data/toy.csv"
OUTDIR = "models/latest"
os.makedirs(OUTDIR, exist_ok=True)

df = pd.read_csv(DATA)
y = df.pop(df.columns[-1])
X_train, X_test, y_train, y_test = train_test_split(df, y, test_size=0.2, random_state=42, stratify=y)
clf = LogisticRegression(max_iter=500).fit(X_train, y_train)
pred = clf.predict(X_test)

metrics = {
    "accuracy": float(accuracy_score(y_test, pred)),
    "f1": float(f1_score(y_test, pred, average="macro"))
}
joblib.dump(clf, f"{OUTDIR}/model.pkl")

with open(f"{OUTDIR}/metrics.json", "w") as f: json.dump(metrics, f, indent=2)
with open(f"{OUTDIR}/model_card.md", "w") as f:
    f.write(f"# MVD LogisticRegression\n\nMetrics: {metrics}\nData: toy.csv\nSeed: 42\n")
print(json.dumps(metrics))

```

**src/evaluate.py**

```python
import json, sys
with open("models/latest/metrics.json") as f: m = json.load(f)
ACC_THR, F1_THR = 0.7, 0.7
ok = (m["accuracy"] >= ACC_THR) and (m["f1"] >= F1_THR)
print("metrics:", m, "thresholds:", {"acc":ACC_THR,"f1":F1_THR})
sys.exit(0 if ok else 1)

```

**src/infer.py**

```python
import sys, joblib, pandas as pd
clf = joblib.load("models/latest/model.pkl")
X = pd.read_json(sys.stdin)     # echo '{"feat1":..., "feat2":...}' | python src/infer.py
print(clf.predict(pd.DataFrame([X]))[0])

```

**src/serve.py**

```python
import joblib, pandas as pd
from fastapi import FastAPI
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from fastapi.responses import Response

clf = joblib.load("models/latest/model.pkl")
app = FastAPI(title="MVD Inference")
REQS = Counter("requests_total", "Total requests")
LAT  = Histogram("request_latency_seconds", "Latency")

@app.post("/predict")
def predict(payload: dict):
    REQS.inc()
    with LAT.time():
        X = pd.DataFrame([payload])
        y = int(clf.predict(X)[0])
        return {"prediction": y}

@app.get("/metrics")
def metrics():
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)

```

**Dockerfile**

```docker
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "src.serve:app", "--host", "0.0.0.0", "--port", "8000"]

```

**tests/test_infer.py**

```python
import json, subprocess
def test_infer_smoke():
    proc = subprocess.run(["python","src/infer.py"], input=json.dumps({"f1":1,"f2":0}), text=True, capture_output=True)
    assert proc.returncode == 0

```

---

# CI/CD (GitLab) — `.gitlab-ci.yml`

```yaml
stages: [lint, unit, train, evaluate, package, deploy, smoke]

variables:
  IMAGE_NAME: "$CI_REGISTRY_IMAGE/mvd:${CI_COMMIT_SHORT_SHA}"

lint:
  stage: lint
  image: python:3.11
  script:
    - pip install ruff==0.6.8
    - ruff check src
  rules: [ { when: on_success } ]

unit:
  stage: unit
  image: python:3.11
  script:
    - pip install -r requirements.txt pytest
    - pytest -q
  artifacts:
    when: always
    reports: { junit: junit.xml }
  rules: [ { when: on_success } ]

train:
  stage: train
  image: python:3.11
  script:
    - pip install -r requirements.txt
    - python src/train.py
    - cat models/latest/metrics.json
  artifacts:
    paths: [models/latest/]
    expire_in: 1 week
  rules: [ { when: on_success } ]

evaluate:
  stage: evaluate
  image: python:3.11
  needs: ["train"]
  script:
    - pip install -r requirements.txt
    - python src/evaluate.py   # fails pipeline if below threshold
  rules: [ { when: on_success } ]

package:
  stage: package
  image: docker:26
  services: [ docker:26-dind ]
  needs: ["evaluate"]
  script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" "$CI_REGISTRY" --password-stdin
    - docker build -t "$IMAGE_NAME" .
    - docker push "$IMAGE_NAME"
  artifacts:
    paths: [models/latest/model_card.md]
  rules: [ { when: on_success } ]

deploy:
  stage: deploy
  image: alpine:3.20
  needs: ["package"]
  script:
    - echo "Deploy step placeholder (Helm/K8s/VM). For MVD, we only echo."
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: http://example.invalid   # replace when our wire K8s ingress
  rules: [ { when: manual } ]     # manual for MVD

smoke:
  stage: smoke
  image: curlimages/curl:8.9.1
  needs: ["deploy"]
  script:
    - echo "Smoke test placeholder: call /predict once when URL is available."
  rules: [ { when: manual } ]

```

> For now deploy and smoke are placeholders. When we are ready, swap in a Helm job that applies a Deployment + Service and then curl POST /predict.
> 

---

# Tiny dataset to start (put in `data/toy.csv`)

```
f1,f2,label
1,0,1
0,1,0
1,1,1
0,0,0
1,0,1
0,1,0

```

---

# How to run locally

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python src/train.py && python src/evaluate.py
uvicorn src.serve:app --reload
# POST: curl -X POST localhost:8000/predict -H 'content-type: application/json' -d '{"f1":1,"f2":0}'
# Metrics: curl localhost:8000/metrics

```
