# Runbook — build, run, measure, submit

Exact commands for the parts that need a running cluster, plus the 4-day plan and commit schedule.
Capture every command's output into `results/` (e.g. `... | tee results/<name>.txt`) — that is the
evidence the report and marking need.

---

## Status

The repository includes saved evidence for the main project tasks:

- Task 1: 3,000 commands processed and 2,861 retained (`results/clean_counts.json`).
- Task 2: student v1/v2 evaluation and training loss are saved under `results/`; the trained
  model's quality gate passes.
- Task 3: local inference and endpoint checks are saved, along with a 1.74 GB container image
  measurement (`results/service_local_inference.txt`, `results/cluster_image_size.txt`).
- Task 4: cluster evidence covers deployment, recovery, cold start, load balancing, and HPA behavior.
  The saved rolling-update test had 2 non-200 responses out of 39 (5.13%), above the 0.5% limit.
  That check is not passed and must be rerun successfully if the limit is required.
- Task 5: subnet plan and NACL simulator results are included; the sample check passes 6/6.
- Task 6: the trained-model CI run and deliberately failing base-model quality-gate run are recorded
  in the report.
- Task 7: load-test and cost measurements are saved under `results/`.

The reflection must be written by Shobhit, scanned, and saved as `report/230101046.pdf`. The existing
PDF in that path is only a typed draft; rewrite it by hand before submitting. The editable report
sources also need to be rendered to `report/report.pdf` after any changes.

---

## 4-day plan & commit schedule

Build is already done in the repo; spread the **commits** (and the heavy runs) across four days so
the history reflects genuine incremental work. Make each day's commit after you have actually run
and checked that day's part.

| Day | Do | Commit (suggested message) |
|-----|----|----|
| **1** | Repo scaffold, Task 5 networking (verify `nacl_sim` 6/6, subnet calc), `make up` brings cluster + SeaweedFS up | `feat: repo scaffold, k3d+seaweedfs, task5 networking (nacl 6/6)` |
| **2** | Task 1 labelling (`ollama pull phi3.5`, `make label` or local), `make clean-data`; commit counts | `feat: task1 teacher labelling workers + clean.py (counts in results/)` |
| **3** | Task 2 training (`make train`, `make train-v2`, `make eval`), publish Release assets | `feat: task2 train pythia-70m student + eval vs teacher/base` |
| **4** | Task 3 container, Task 4 deploy+reliability, Task 6 CI, Task 7 load+cost, report, **tag final** | `feat: task3-7 service, k8s, ci, load test, cost, report` |

```bash
git add -A && git commit -m "<message for the day>"
git push                     # after you create the GitHub repo (see below)
# on day 4, after everything is in:
git tag final && git push --tags
```

### Create the GitHub repo — DONE
Public repo `shobhit2294/230101046_assignment_mini_project_cloud_computing` already created and pushed.

### Publish the model as a Release — DONE
`model-v1` Release already holds `v1.tar.gz` + `v2.tar.gz` (checksums verified against
`models/<v>/model.sha256`). To re-publish after retraining:
```bash
gh release upload model-v1 models/v1.tar.gz models/v2.tar.gz --clobber
```

---

## Task 1 — labelling evidence
```bash
make label
kubectl -n pipeline get pods -o wide | tee results/task1_workers.txt   # two pods, different index
make clean-data                                                        # writes results/clean_counts.json
```

## Task 4 — reliability tests (keep a load test running in another terminal where noted)

**Rolling update + rollback** (step 5):
```bash
# terminal A: load
locust -f tools/locustfile.py --headless --users 4 --spawn-rate 4 --run-time 4m \
  --host http://localhost:8080 --csv results/rollout | tee results/rollout.txt
# terminal B: while load runs
kubectl -n app set env deployment/command-parser MODEL_VERSION=v2 MODEL_URI=s3://models/v2
kubectl -n app rollout status deployment/command-parser
kubectl -n app rollout undo deployment/command-parser
kubectl -n app rollout status deployment/command-parser
```
Check `results/rollout_headers.json` for `X-Model-Version`: it should show v2 during, v1 after.
Failed requests must stay ≤ 0.5%.

**Recovery** (step 6) — delete a Pod under load:
```bash
kubectl -n app delete pod "$(kubectl -n app get pod -l app=command-parser -o name | head -1)"
kubectl -n app get pods -w | tee results/recovery.txt   # replica count returns to 2
```

**Cold start** (step 7) — three runs, report the median; show the model download in the logs:
```bash
for i in 1 2 3; do
  kubectl -n app delete pod -l app=command-parser
  start=$(date +%s)
  kubectl -n app wait --for=condition=ready pod -l app=command-parser --timeout=600s
  echo "run $i cold start: $(( $(date +%s) - start ))s"
done | tee results/coldstart.txt
kubectl -n app logs -l app=command-parser --tail=20 | tee results/coldstart_logs.txt   # shows the S3 download
```

**Load balancing** (step 8) — exactly two Ready replicas, read per-replica counts:
```bash
kubectl -n app scale deployment/command-parser --replicas=2
make load                                               # writes results/loadtest_headers.json
cat results/loadtest_headers.json                        # each replica should serve >= 30%
```

## Task 7 — performance, resources, cost
```bash
make load                                                # results/loadtest_stats.csv (p50, p95, rps, failures)
kubectl -n app top pods | tee results/top_pods.txt       # CPU + memory per pod during load
python scripts/cost.py --replicas 2 --cpu-limit 1 --mem-gib 1 --rps <rps_from_csv> | tee results/cost.txt
```

## Container size + limits (Task 3)
```bash
docker images cloud-mini/service:v1 --format '{{.Size}}' | tee results/image_size.txt   # <= 2 GB
docker run --rm --cpus 1 --memory 1g ... cloud-mini/service:v1   # send 50 requests, watch for OOM
```
