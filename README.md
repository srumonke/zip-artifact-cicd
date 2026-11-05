# Build an artifact once, deploy that exact artifact

A minimal, working Harness pipeline that builds a `.zip`, pushes it to an artifact registry,
and then deploys **that same zip** — with **no output variables** wired between the build and
deploy stages.

## The question this answers

> Our pipeline builds a `.zip`, pushes it to an artifact registry, then deploys it. The
> approach we found uses **output variables** on the push step, which a later pull/deploy step
> consumes. It feels overly complex. Is there an easier way, or is this correct?

There is an easier way, and it is also the recommended one. **You do not need to pass anything
from the push step to the deploy step.**

## The pattern

The CI stage and the CD stage never communicate directly. They both address the **registry**,
using the **same deterministic coordinates**.

```
                        ┌──────────────────────────────┐
   CI stage ─── push ──▶│   registry                   │
   (HarUpload step)     │   package: demo-app          │◀── resolve ─── CD stage
                        │   version: 1.0.<build no.>   │                (service's
                        └──────────────────────────────┘                 artifact source)

                 no output variables, no step-to-step wiring
```

1. **CI** derives the version from the pipeline's own build number — `<+pipeline.sequenceId>` —
   and pushes the zip to those coordinates.
2. **CD** gets the artifact from the **service's artifact source**, configured with those same
   coordinates. Harness resolves it *before* the first deploy step runs and exposes it as
   `<+artifact.*>`.
3. The deploy step reads `<+artifact.version>` — never `<+steps.push_zip.output.*>`.

A useful signal: the native `HarUpload` step publishes **no output variables at all**. That is
deliberate.

### Where the handoff lives

[`.harness/service.yaml`](.harness/service.yaml), in `artifacts.primary.sources[0].spec`:

```yaml
- identifier: app_zip
  type: Har
  spec:
    registryRef: <YOUR_REGISTRY>
    type: generic
    spec:
      artifact: demo-app
      version: 1.0.<+pipeline.sequenceId>          # same coordinates as the push
      fileName: demo-app-1.0.<+pipeline.sequenceId>.zip
```

That block plus the `HarUpload` step in the pipeline are the entire contract between build and
deploy. It's also where you put `<+input>` if you want a human to choose the version at runtime.

## Why this beats passing output variables

Output variables aren't *wrong* — they're narrower. They bind your deployment to one specific CI
execution. Once the version exists only as a step output, all of these stop working:

| | Output variables | Service artifact source |
|---|---|---|
| Re-run **only** the Deploy stage | ✗ CI outputs are gone | ✓ artifact re-resolves |
| Roll back to version N-1 | ✗ no CI run in this pipeline produced it | ✓ pick any version |
| Deploy from a separate CD-only pipeline | ✗ not possible | ✓ same service, works as-is |
| Let a human pick the version at runtime | ✗ not possible | ✓ `version: <+input>` |
| Deployed version in the Services dashboard | ✗ Harness doesn't know | ✓ tracked per instance |
| Promote test → prod | ✗ must re-thread the variables | ✓ add a stage |

### When output variables *are* right

When the value genuinely cannot be derived ahead of time:

- a digest the registry assigns at push time (e.g. an OCI `sha256:...` you want to pin by digest)
- a registry-generated build number you don't control

For a *dynamically computed* version (from `git describe`, a bumped tag, a version file), the
cleaner fix is to compute it **once** into a pipeline variable both stages read — which is what
this demo does for `packageName` and `registryName`.

## Proof it deploys the identical artifact

The deploy step re-hashes the zip after downloading and asserts that a `version.txt` baked in at
build time matches `<+artifact.version>`. From a verified run:

| | sha256 of the zip |
|---|---|
| CI `Build ZIP` step | `eaf816c87b1a8e78d5926a470ed81f0506b60256c9851ca8a985452951591823` |
| CD `Deploy ZIP` step, after download | `eaf816c87b1a8e78d5926a470ed81f0506b60256c9851ca8a985452951591823` |

Byte-for-byte identical, with zero variables wired between the two stages.

---

# Setup

## Prerequisites

- A Harness project with **CI** and **CD** enabled
- A **Linux delegate** connected (the demo deploys *to* the delegate, so you need no other infra)
- An environment with identifier `test` (or edit `environmentRef` in two files)

## 1. Create a GENERIC Artifact Registry

**Artifact Registry → + New Artifact Registry → Generic.** Note its identifier.

No connector and no credentials are needed — it's Harness-native on both the push and pull side.

## 2. Create an API key secret

The deploy step `curl`s the zip back out of the registry, which needs an API key.

1. Generate a token: **Artifact Registry → your registry → Setup Client** (or your user
   profile → **API Keys**).
2. Store it: **Project Settings → Secrets → + New Secret → Text**, identifier
   **`harness_api_key`**.

## 3. Fill in the placeholders

Three placeholders appear across the four YAML files:

| Placeholder | Replace with | Appears in |
|---|---|---|
| `<YOUR_PROJECT>` | your project identifier | all four files |
| `<YOUR_REGISTRY>` | registry identifier from step 1 | `pipeline.yaml`, `service.yaml` |
| `<YOUR_ACCOUNT_ID>` | your Harness account ID (in any Harness URL, after `/account/`) | `pipeline.yaml` |

```bash
cd .harness
sed -i '' 's/<YOUR_PROJECT>/my_project/g; s/<YOUR_REGISTRY>/my_registry/g; s/<YOUR_ACCOUNT_ID>/abc123XYZ/g' *.yaml
```

If `orgIdentifier` isn't `default` in your account, change that too.

## 4. Apply the four entities, in this order

They reference each other, so order matters:

```
1. deployment-template.yaml   (zip_deploy_target v1)
        │  referenced by both of the next two
        ├──▶ 2. service.yaml            (zip_demo)
        └──▶ 3. infrastructure.yaml     (zip_demo_target, under env `test`)
                     │
                     └──▶ 4. pipeline.yaml   (zip_build_deploy)
```

For each: go to the relevant page, choose **+ New**, switch to the **YAML** editor, paste the file.

| File | Where in the UI |
|---|---|
| `deployment-template.yaml` | Project Settings → Templates → + New Template → Deployment Template |
| `service.yaml` | Project Settings → Services → + New Service |
| `infrastructure.yaml` | Project Settings → Environments → `test` → Infrastructure Definitions |
| `pipeline.yaml` | Pipelines → + Create a Pipeline |

The pipeline names the other three by identifier and won't save if any are missing.

## 5. Run it

**Pipelines → `zip_build_deploy` → Run.** No runtime inputs. Takes ~90 seconds, most of it CI
machine provisioning.

Watch for these three lines in the **Deploy ZIP** step — they're the whole point:

```
   version    : 1.0.3
   version.txt inside the zip : 1.0.3
   MATCH - deployed the exact artifact CI pushed
```

---

## What's in `.harness/`

| File | Entity | Identifier |
|---|---|---|
| [`deployment-template.yaml`](.harness/deployment-template.yaml) | Deployment Template (CustomDeployment) | `zip_deploy_target` v1 |
| [`service.yaml`](.harness/service.yaml) | Service | `zip_demo` |
| [`infrastructure.yaml`](.harness/infrastructure.yaml) | Infrastructure Definition (env `test`) | `zip_demo_target` |
| [`pipeline.yaml`](.harness/pipeline.yaml) | Pipeline | `zip_build_deploy` |

Two deliberate simplifications, so the demo shows the pattern rather than the plumbing:

- **Harness Artifact Registry rather than an external registry.** The pattern is identical with
  JFrog, Nexus, or ECR — only the artifact source `type` and the upload step change. HAR just
  needs no credentials, so there's nothing to configure before it runs.
- **Custom Deployment as the target.** No VM, no Kubernetes cluster, no host list, no SSH or
  WinRM credential — the delegate itself is the deploy target. Your real deployments keep
  whatever target they already use; the pattern is unaffected.

## Adapting it to a real deployment

Swap the Deploy stage's `deploymentType` and infrastructure for your actual target (SSH, WinRM,
Kubernetes, ECS…). **The artifact-resolution part does not change** — `<+artifact.version>` works
the same way for every deployment type.

If you move to a different registry, you change two things: the service's artifact source `type`,
and the CI upload step. The deploy steps don't change at all — which is the strongest practical
argument for this decoupling.

## Practical notes

1. **`<+artifact.metadata.fileName>` is empty for HAR generic artifacts**, even though the
   service step logs display the filename. Derive the filename from `<+artifact.version>`
   instead. Unresolvable `metadata` sub-paths render as the literal string `null` rather than
   failing the step — so echo anything you depend on, or a typo fails silently.

2. **The Linux delegate image has no `unzip` and no `python3`** (only `java` and `tar`). The
   demo installs `unzip` on the fly as a fallback — fine for a demo, not something you want in a
   production deploy step. Either bake `unzip` into a custom delegate image, or ship a `.tar.gz`.

3. **HAR generic download URL** is
   `https://pkg.harness.io/pkg/<ACCOUNT_ID>/<REGISTRY>/generic/<PACKAGE>/<VERSION>?filename=<FILE>`
   with an `x-api-key` header. Not `/tools/generic/` and not `/tools/files/`.

4. **Repository format:** the **Raw** HAR format is supported as a CD artifact source only for
   **WinRM** deployment types. Use **GENERIC**, which works broadly — including the Custom
   Deployment type used here.

5. The upload step logs a cosmetic `status 405 — Failed to publish metadata` warning. The upload
   itself succeeds. This is plausibly why `artifact.metadata` comes back empty.
