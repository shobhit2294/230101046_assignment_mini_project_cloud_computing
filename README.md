# Cloud Computing Mini Project: Command Parser

This project turns short English commands into structured JSON. A larger language model,
Phi-3.5, provides training examples; a fine-tuned Pythia 70M model handles predictions in
the deployed service. The service runs on a local Kubernetes cluster, with SeaweedFS used
for object storage.

- Student: Shobhit Mishra
- Roll number: 230101046
- Repository: https://github.com/shobhit2294/230101046_assignment_mini_project_cloud_computing

## What the service does

Send a command to `POST /parse`. The service returns a JSON object with four fields:

    Request:  {"text": "set an alarm for seven thirty am tomorrow"}
    Response: {"intent": "alarm_set", "date": "tomorrow", "time": "seven thirty am", "place": null}

## What you need

Install these tools before starting:

| Tool | Purpose |
| --- | --- |
| Docker | Build images and run the local Kubernetes cluster |
| [k3d](https://k3d.io) and kubectl | Create and manage the Kubernetes cluster |
| [Ollama](https://ollama.com) | Run the Phi-3.5 teacher model for labeling |
| Python 3.11 | Run the training scripts, utilities, and tests |
| make | Run the project commands |
| [Locust](https://locust.io) | Run the load test |
| [GitHub CLI](https://cli.github.com) | Manage GitHub releases |

On Windows, run the shell scripts from Git Bash. You can install `make` with Chocolatey
using `choco install make`. Each `make` target runs a script or command from the repository.

## Project layout

- `cluster/` contains the k3d cluster configuration.
- `prompts/` and `worker/` contain the teacher prompt and data-labeling workers.
- `training/` contains the student training and prediction scripts.
- `service/` contains the FastAPI application and its Dockerfile.
- `k8s/` contains the Kubernetes manifests for the application and storage.
- `network/` contains the subnet plan and NACL configuration.
- `scripts/` contains setup, deployment, evaluation, and cost utilities.
- `tools/` contains the scoring, NACL simulation, and load-testing tools.
- `tests/` contains the unit tests.
- `data/` contains the command pool, validation data, and project configuration.
- `results/` stores evaluation and run evidence.
- `report/` contains the project report.

## Getting started

Copy the example environment file and edit the development secret before bringing up the
cluster. Run these commands from the repository root:

    cp .env.example .env
    make up

`make up` creates the k3d cluster if needed, builds and imports the service and worker images,
then deploys SeaweedFS and the service. It also creates the `datasets` and `models` buckets,
uploads the command pool, and seeds the model from the GitHub Release when one is available.
The service is exposed at `http://localhost:8080/parse`.

To create the training data and train the model, start Ollama and run the labeling and data
cleaning steps before training:

    ollama pull phi3.5
    ollama serve

In another terminal:

    make label
    make clean-data
    make train

The full training workflow and the other project tasks are described below.

## Task 1: label and clean the commands

Ollama runs the Phi-3.5 teacher locally. The labeling job splits the work across two
Kubernetes workers. If you prefer to run the workers without Kubernetes, use the local
helper script.

    ollama pull phi3.5
    ollama serve
    make up
    make label
    make clean-data

    # To run the workers locally instead:
    bash scripts/run_workers_local.sh

The cleaned training data is written to `datasets/v1/train.jsonl`, along with a checksum.
The cleaning script also prints summary counts.

## Task 2: train and evaluate the student

Install the CPU version of PyTorch and the remaining training dependencies:

    pip install torch==2.4.1 --index-url https://download.pytorch.org/whl/cpu
    pip install -r training/requirements.txt

Use a small subset for a quick smoke test, or train the full model. The second version uses
a different seed and is used in the deployment exercises.

    make train SUBSET=50
    make train
    make train-v2
    make eval

Training writes the models under `models/v1/` and `models/v2/`. The evaluation compares the
teacher, the untrained base model, and the trained student, then saves the results under
`results/`.

Publish `models/v1.tar.gz` and `models/v2.tar.gz` as assets on the `model-v1` GitHub Release.
Check that each uploaded archive matches its checksum. Do not commit model weights to the
repository.

## Task 3: build and run the service

Build the service image and check its size:

    docker build -f service/Dockerfile -t cloud-mini/service:v1 .
    docker images cloud-mini/service:v1

For a local test, start the container with access to the model in SeaweedFS. Replace the
credential placeholders with the values from your local environment:

    docker run --rm -p 8000:8000 -e MODEL_URI=s3://models/v1 -e MODEL_VERSION=v1 \
      -e S3_ENDPOINT=http://host.docker.internal:8333 -e MAX_NEW_TOKENS=64 \
      -e AWS_ACCESS_KEY_ID=YOUR_ACCESS_KEY -e AWS_SECRET_ACCESS_KEY=YOUR_SECRET_KEY \
      cloud-mini/service:v1

In another terminal, send a sample request:

    curl -s localhost:8000/parse -H 'content-type: application/json' \
      -d '{"text":"weather in london"}'

The service also supports a `FAULT_INJECT` test hook. Set `FAULT_INJECT=1` when testing
the error response.

## Task 4: deploy and check reliability

Deploy the service and inspect the Kubernetes resources:

    make deploy
    kubectl -n app get deploy,svc,ingress,hpa,pods
    curl -i -s localhost:8080/parse -H 'content-type: application/json' \
      -d '{"text":"weather in london"}'

The deployment exercises include rolling updates and rollback, recovery after a pod is
deleted, cold-start timing, and load balancing. Follow the commands in
[`docs/RUNBOOK.md`](docs/RUNBOOK.md) to run those checks and save the evidence.

## Task 5: check the network configuration

These utilities do not require a running cluster:

    python tools/nacl_sim.py network/nacl.json network/packets_sample.jsonl --check
    python scripts/subnet_calc.py

The sample NACL check should report six passing cases.

## Task 6: run tests and CI checks

Run the local lint and test commands:

    make lint
    make test

The GitHub Actions workflow runs linting, tests, model evaluation, a quality gate using
`score.py --min-validity 95`, and Gitleaks. To see the quality gate fail on the untrained
base model, run the `ci` workflow with `model_source` set to `base`.

## Task 7: measure performance and cost

Run the load test against the deployed service, then use its measured request rate when
estimating cost:

    make load
    python scripts/cost.py --replicas 2 --cpu-limit 1 --mem-gib 1 --rps <measured-rps>

The load test writes its output under `results/`. See the runbook for the additional
measurements to collect for the report.

## Tear down

When you are finished, remove the local k3d cluster:

    make down
