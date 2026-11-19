# W&B Models — Practical Q&A

Answers to four recurring questions, verified against the official docs on
[docs.coreweave.com](https://docs.coreweave.com/products/wandb) (tip: append `.md` to any
docs URL for raw markdown).

---

## 1. Plotting multiple metrics on one chart, with styling, from the client

**Question:** How do I plot metrics like training and validation loss on the *same* chart, and
control line thickness and other visualisation settings via the wandb client?

**Answer:** Logging `run.log({"train/loss": ..., "val/loss": ...})` always creates one panel per
metric — prefixes like `train/` only group panels into sections. To merge metrics into one panel
*and* style it from Python, use the **`wandb-workspaces`** library, which edits the same saved
workspace/report the UI edits:

```bash
pip install wandb wandb-workspaces
```

```python
import wandb
import wandb_workspaces.workspaces as ws
import wandb_workspaces.reports.v2 as wr   # panel classes live here

# line_colors/line_widths are keyed "{runId}:{metricName}" — grab run IDs first
api = wandb.Api()
run_id = next(iter(api.runs("my-team/my-project"))).id

loss_panel = wr.LinePlot(
    title="Training vs Validation Loss",
    x="Step",
    y=["train/loss", "val/loss"],            # multiple metrics, ONE panel
    smoothing_type="gaussian",               # exponentialTimeWeighted|exponential|gaussian|average|none
    smoothing_factor=0.8,
    smoothing_show_original=True,
    log_y=True,
    legend_position="north",
    line_colors={f"{run_id}:train/loss": "#1f77b4", f"{run_id}:val/loss": "#d62728"},
    line_widths={f"{run_id}:train/loss": 1.5,  f"{run_id}:val/loss": 3.0},   # line thickness
    line_marks ={f"{run_id}:train/loss": "solid", f"{run_id}:val/loss": "dashed"},
    line_titles={f"{run_id}:train/loss": "train", f"{run_id}:val/loss": "validation"},
)

workspace = ws.Workspace(
    entity="my-team", project="my-project", name="Loss comparison",
    sections=[ws.Section(name="Losses", panels=[loss_panel], is_open=True)],
    settings=ws.WorkspaceSettings(x_axis="Step", ignore_outliers=False),
    runset_settings=ws.RunsetSettings(
        run_settings={run_id: ws.RunSettings(color="#e377c2")},   # run colors, programmatically
    ),
)
workspace.save()          # or .save_as_new_view(); load existing: ws.Workspace.from_url("...?nw=...")
```

The same `LinePlot` object drops into a Report via `wr.PanelGrid(panels=[...], runsets=[...])`.
Other `LinePlot` controls: `groupby`/`groupby_aggfunc` (mean/min/max/…), `range_x/range_y`,
`log_x`, `custom_expressions`, `metric_regex`, `plot_type` (line/stacked-area/pct-area),
`font_size`, `max_runs_to_show`.

**Lighter alternatives:**

- `wandb.plot.line_series(xs, ys, keys, title, xname)` — combines several series into one custom
  chart at log time. No styling parameters at all (no color/width/smoothing), backed by a Table
  (10,000-row cap). Good for one-off comparisons.
- Custom Vega charts (`wandb.plot_table`) — full mark control including `strokeWidth`, at the cost
  of authoring a Vega spec in the UI editor first.
- `run.define_metric("val/*", step_metric="epoch")` — aligns metrics on a shared x-axis but does
  **not** merge panels.

**Honest limits:** line thickness is settable only *per series* via `line_widths` (you must
enumerate run IDs) — there is no global/default width setting in the workspace API. The UI quick
panel builder (regex like `(?:train|val)/loss`) is point-and-click only, though `metric_regex`
covers it programmatically. The workspaces API is Public Preview.

**References:**

1. `wandb_workspaces` API reference — full `LinePlot`/`Workspace`/`RunsetSettings` parameter lists, including `line_widths`, `line_colors`, `line_marks` — [workspaces reference](https://docs.coreweave.com/products/wandb/ref/wandb_workspaces/workspaces)
2. Line plot panel — what the UI exposes (smoothing, outliers, legends) and the regex multi-metric builder — [line plots](https://docs.coreweave.com/products/wandb/app/features/panels/line-plot)
3. `wandb.plot.line_series` and other preset charts logged at train time — [log plots](https://docs.coreweave.com/products/wandb/track/log/plots)
4. Custom Vega charts — `wandb.plot_table`, authoring specs with full mark control (`strokeWidth` etc.) — [custom charts](https://docs.coreweave.com/products/wandb/app/features/custom-charts)
5. `define_metric` — custom x-axes for aligning train/val steps — [customize logging axes](https://docs.coreweave.com/products/wandb/track/log/customize-logging-axes)

---

## 2. Registry over data in CoreWeave buckets — without copying it

**Question:** Our data lives in CoreWeave cloud buckets. Can I have an artifact that just points
at those files, link it in the Registry, and use it like a normal artifact — without duplicating
the data?

**Answer: yes — this is exactly what reference artifacts are for**, and the docs name CoreWeave
AI Object Storage as a supported provider: W&B stores only per-object metadata (ETag/MD5
checksum, size, version ID if bucket versioning is on); *"the files themselves never leave your
system."* Reference artifacts link into Registry collections like any artifact (a link is itself
just a pointer), and *"the APIs for downloading artifacts are the same for both reference and
non-reference artifacts."*

### Walkthrough: bucket → reference artifact → Registry → consumer

**Step 1 — create the registry (one-time, org-level).** Every org already has the core
**Models** and **Datasets** registries (target paths `wandb-registry-model` /
`wandb-registry-dataset`), so this step is optional — create a custom registry when you want a
separate access-control boundary or alias namespace for this data:

```python
import wandb

api = wandb.Api()
api.create_registry(
    name="cw-datasets",            # target path prefix becomes: wandb-registry-cw-datasets
    visibility="organization",     # or "restricted" (invite-only; creator becomes registry admin)
)
```

(UI equivalent: Registry app → Create registry. Collections do **not** need pre-creating —
they're created automatically on first link.)

**Step 2 — point the SDK at CoreWeave AI Object Storage.** wandb uses boto3 under the hood, so
the standard S3 env vars work:

```python
import os

os.environ["AWS_S3_ENDPOINT_URL"]   = "https://cwobject.com"  # inside CW clusters: http://cwlota.com (LOTA cache)
os.environ["AWS_ACCESS_KEY_ID"]     = "<cw-access-key-id>"
os.environ["AWS_SECRET_ACCESS_KEY"] = "<cw-secret-key>"
os.environ["AWS_REGION"]            = "US-EAST-04A"           # your CW availability zone
```

**Step 3 — create the reference artifact and log it.** `add_reference` walks the prefix and
records one pointer per object (checksum/ETag, size, version ID) — no bytes are copied:

```python
with wandb.init(entity="my-team", project="data-publishing") as run:  # team entity — personal entities can't link to Registry
    art = wandb.Artifact(name="training-corpus", type="dataset")
    art.add_reference(uri="s3://my-cw-bucket/datasets/corpus-v3/")    # pointers only, no copy
    # for very large prefixes: add_reference(..., max_objects=500_000)
    run.log_artifact(art)
    art.wait()                                                        # ensure the version is committed before linking
```

**Step 4 — link it into the registry.** The collection (`training-corpus` here) is auto-created
on first link; the linked version is a pointer with its own version number inside the collection:

```python
    art.link(
        target_path="wandb-registry-cw-datasets/training-corpus",
        aliases=["production"],    # the movable pointer downstream consumers pin to
    )
```

Optionally document the collection with a card so consumers know what they're pulling:

```python
coll = wandb.Api().artifact_collection(
    type_name="dataset", name="wandb-registry-cw-datasets/training-corpus")
coll.description = "HiLift corpus v3 — referenced from s3://my-cw-bucket; data never copied."
coll.save()
```

**Step 5 — consume it like any artifact.** The download API is identical to a normal artifact;
bytes come straight from the bucket using the *consumer's own* credentials (W&B never proxies or
stores them), and `use_artifact` records the lineage edge as usual:

```python
import os, wandb

os.environ["AWS_S3_ENDPOINT_URL"]   = "https://cwobject.com"
os.environ["AWS_ACCESS_KEY_ID"]     = "<consumer-cw-key>"      # consumer's own CW credentials
os.environ["AWS_SECRET_ACCESS_KEY"] = "<consumer-cw-secret>"

with wandb.init(entity="my-team", project="training") as run:
    data_dir = run.use_artifact("wandb-registry-cw-datasets/training-corpus:production").download()
```

**Caveats to plan around:**

- **Enable bucket versioning.** If objects are overwritten, old artifact versions only stay
  retrievable with S3-style versioning on. CoreWeave supports it post-creation:
  `aws s3api put-bucket-versioning --bucket <b> --versioning-configuration Status=Enabled`.
- **Virtual-hosted addressing.** CoreWeave doesn't support path-style S3 requests; if downloads
  fail, set `addressing_style = virtual` in `~/.aws/config`.
- **Every consumer needs bucket credentials** (humans, CI, Launch jobs) — that's the design.
- `add_reference` caps objects per prefix (set `max_objects=` explicitly for big prefixes);
  `checksum=False` speeds up creation but loses per-object change detection.
- Linking to a Registry **permanently disables the artifact's TTL policy** (unlinking doesn't restore it).
- Personal-entity artifacts can't be linked to Registry; **Restricted Viewer** registry role can't
  download or `use_artifact`.
- Referenced media won't render in the W&B UI unless the bucket's CORS policy allows the app domain.

**Alternative — team-level BYOB (Secure Storage Connector):** if you *also* want the artifacts
W&B itself writes (checkpoints, run files) to live in your CoreWeave bucket, Multi-tenant team-level
BYOB supports CoreWeave AI Object Storage (Enterprise; immutable — must be set at team creation;
CW buckets currently need a Support ticket to verify). BYOB = W&B writes into your bucket;
reference artifacts = data never moves. They compose — a BYOB team can still log reference artifacts.

**References:**

1. Reference artifacts — tracking files in external S3-compatible storage (CoreWeave AI Object Storage named explicitly), `AWS_S3_ENDPOINT_URL`, credentials, `max_objects`, CORS, bucket versioning — [track external files](https://docs.coreweave.com/products/wandb/artifacts/track-external-files)
2. Registry overview — registries → collections → linked versions, `create_registry`, collection auto-creation on first link — [Registry](https://docs.coreweave.com/products/registry)
3. Linking a version into a collection — target paths, team-entity requirement, link = pointer not copy — [link version](https://docs.coreweave.com/guides/registry/link_version)
4. Downloading/consuming from a registry — path format, roles required — [download & use](https://docs.coreweave.com/guides/registry/download_use_artifact)
5. Artifact TTL — linking to a registry permanently disables TTL — [TTL policies](https://docs.coreweave.com/products/wandb/artifacts/ttl)
6. Secure Storage Connector (BYOB) — team-level buckets incl. CoreWeave AI Object Storage; contrast with references — [secure storage connector](https://docs.coreweave.com/products/wandb/platform/hosting/data-security/secure-storage-connector)
7. CoreWeave Object Storage endpoints — `cwobject.com` vs in-cluster `cwlota.com` (LOTA), virtual-hosted addressing requirement — [configure endpoints](https://docs.coreweave.com/products/storage/object-storage/using-object-storage/configure-endpoints)

---

## 3. Managing incomparable run cohorts in one project

**Question:** Runs r₁…r_k used model/dataset state sᵢ; later runs used state sⱼ and can't be
compared with the earlier ones. Manually selecting comparable runs for a view is unwieldy. How do
we handle this?

**Answer:** W&B can't detect that runs have become incomparable, so you label them and let
filters do the selecting. When a run starts, record which model/dataset state it belongs to as a
key in its `config` (e.g. `model_state: "s2"`). Then create one **saved view** per state: a saved
view is a workspace whose run list is pinned to a filter (here `model_state = "s2"`), so opening
the view shows only the comparable runs — and every future run that stamps itself `s2` appears
there automatically, with no manual selection ever again. Each view has its own shareable URL and
sits in the workspace menu next to your personal workspace. For the runs that already exist from
before this convention, you can't edit their config, but you can bulk-add a **tag** (select them
in the runs table → Tag, or via the Public API — including finding them by which dataset artifact
they consumed) and filter the view on the tag instead. Views can be created by hand in the UI
("Save as a new view") or scripted with the `wandb-workspaces` library if you have many cohorts.
Two boundaries to know: `group=` is the wrong tool (it aggregates grouped runs into mean curves,
changing chart semantics), and once a project accumulates on the order of 10,000 runs the
documented advice is to move retired cohorts to an archive project (a UI-only operation).

**References:**

1. Filtering runs — how run filters work: operators, AND/OR groups, filtering on config keys — [filter runs](https://docs.coreweave.com/products/wandb/runs/filter-runs)
2. Workspaces & saved views — creating, naming, and sharing saved views ("Save as a new view") — [workspaces](https://docs.coreweave.com/products/wandb/track/workspaces)
3. Tags — bulk-tagging existing runs from the runs table or the Public API — [tags](https://docs.coreweave.com/products/wandb/runs/tags)
4. `wandb_workspaces` reference — scripting saved views with runset filters, for many cohorts — [workspaces API](https://docs.coreweave.com/products/wandb/ref/wandb_workspaces/workspaces)
5. Artifact graph traversal — `used_by()` to find every run that consumed a given dataset version — [explore artifact graphs](https://docs.coreweave.com/products/wandb/artifacts/explore-and-traverse-an-artifact-graph)
6. Limits & performance — the ~10k runs/project guidance and archive-project advice — [experiments limits](https://docs.coreweave.com/products/wandb/track/limits)
7. Managing runs — moving runs between projects (UI-only; artifacts don't move with the run) — [manage runs](https://docs.coreweave.com/products/wandb/runs/manage-runs)

---

## 4. Resuming runs vs asynchronous evals — which and when

**Question:** What is best practice for resuming runs vs asynchronous evals?

**Answer:** They solve different problems, and the distinction is about **who is writing to the
run**:

- **Resume** continues the *same logical process* — one writer, picking up where it crashed or
  was preempted, appending to the same metric stream.
- **Async evals** are *different processes adding information about* a training run — they must
  never write into it directly; each eval gets its own run, tied back by group/config/lineage.

The anti-pattern to avoid is using resume for evals: reopening the training run from an eval
worker (`wandb.init(id=train_run_id, resume="allow")`) *looks* convenient but breaks the run's
data model with concurrent writers (details below).

### Resuming: crash/preemption recovery

```python
# The documented recommendation: resume="allow" + an explicit run ID.
with wandb.init(
    entity="my-team", project="my-project",
    id=run_id,              # the run to continue (store it with your checkpoints)
    resume="allow",         # resume if it exists, create it if it doesn't
) as run:
    # W&B resumes the metric history from the last recorded step.
    # Restoring model/optimizer state and your epoch counter from a
    # checkpoint is YOUR job — otherwise new logs land on wrong steps.
    ...
```

| `resume=` | Run ID exists | Run ID doesn't exist | Use when |
|---|---|---|---|
| `"must"` | resumes from last step | **error** | the run must already exist |
| `"allow"` | resumes from last step | creates it | **recommended default** for restartable jobs |
| `"never"` | **error** | creates it | guarantee a fresh run |
| `"auto"` | resumes (recovers ID from the local run dir) | creates | same-machine, same-directory restarts only |

Caveats: never let two live processes share one run ID ("unexpected results will occur");
`"auto"` depends on the local filesystem, so pass `WANDB_RUN_ID` explicitly on clusters; for
preemptible sweeps use `mark_preempting()` and re-`init` within the 60-minute lease. Related but
different tools: `fork_from` branches a *new* run off a checkpoint's history (original untouched);
`resume_from` (rewind) resets the *same* run's history from a step — both cloud-preview features.

### Asynchronous evals: the one-job-one-run pattern (notebook 6)

Why not just resume the training run from eval workers? Notebook 6 §6 names four structural
failure modes, none of which show up in a single-kernel demo but all of which bite in production:
**config overwrites** (eval config keys silently merge over training's, no conflict detection),
**step interleaving** (two writers → non-monotonic steps → meaningless charts), **`finish()`
races** (first finisher closes the run for everyone), and **summary last-write-wins**. The safe
pattern is *one job = one run*, tied together server-side:

```python
# Each eval worker: its OWN run — never the training run's ID.
with wandb.init(
    entity=ENTITY, project=PROJECT,
    name=f"eval-{train_name}-ep{ep:03d}",
    group=training_run_id,        # UI: groups the eval fleet with its training run
    job_type="eval",              # distinguishes role within the group
    resume="never",               # structurally cannot touch the training run
    config={"source_training_run_id": training_run_id,   # server-side discovery key
            "source_checkpoint_artifact": ref},
) as eval_run:
    ckpt = eval_run.use_artifact(ref, type="model")      # lineage: eval consumed this checkpoint
    ...                                                  # evaluate, then:
    eval_run.log({**metrics, "eval/checkpoint_epoch": ep})
```

Workers don't coordinate and don't know about each other — checkpoint discovery is a *pull* on
the training run's logged artifacts, and triggering is whatever you have (Automations, CI, Slurm).
A separate aggregation pass then collects results and writes them back — **summary-only, into a
namespace the training script never writes**:

```python
api = wandb.Api()
# find the eval fleet by the config key every worker stamped
eval_runs = api.runs(f"{ENTITY}/{PROJECT}", filters={
    "config.source_training_run_id": training_run_id, "config.job_type": "eval"})

agg = aggregate(eval_runs)                               # e.g. eval/best/*, eval/latest/*
training_run = api.run(f"{ENTITY}/{PROJECT}/{training_run_id}")
training_run.summary.update(agg)                         # summary keys only — NEVER run.log()
training_run.update()                                    # into another run, never training-owned keys
```

This is safe at any time for disjoint `eval/*` keys, and unconditionally safe once training has
called `run.finish()`. Trade-off: it's eventually consistent — the training run's summary shows
eval results only after the aggregator pass runs.

### Decision rule

**Resume** when the new process *is* the old process continued: same code, same config, same
metric stream, exactly one writer — crash recovery, preemption, pausing. **Async evals** when a
different job adds information *about* a run: post-hoc checkpoint evals, benchmarks, profiling —
especially in parallel or while training is still going. If you're tempted to resume a run from a
second process while the first might still write, you want the async pattern; if a "new run per
restart" would fracture one experiment's history across run IDs, you want resume.

**References:**

1. Resume a run — the `resume` mode table, run-ID requirement, `auto` filesystem caveat, preemptible-sweep lease, and the `resume="allow"` + explicit ID recommendation — [resuming](https://docs.coreweave.com/products/wandb/runs/resuming)
2. Fork a run — branch a new run from a source run's step without touching it — [forking](https://docs.coreweave.com/products/wandb/runs/forking)
3. Rewind a run — reset the same run's history from a step; logs reset, system metrics not backfilled — [rewind](https://docs.coreweave.com/products/wandb/runs/rewind)
4. Notebook 6 in this repo — the full async-eval implementation: per-checkpoint eval runs, `group`/`config` linkage, lineage via `use_artifact`, public-API aggregation, and the §6 "why not resume?" failure-mode analysis — `notebooks/6_advanced_asynchronous_evals.ipynb`
