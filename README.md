# release-pipeline-gitops

Deployment side of the release pipeline. Git is the source of truth: whatever tag is written
here is what Argo CD runs in the cluster. Images are built in
[release-pipeline-app](https://github.com/abhishek-patwa-devops/release-pipeline-app).

## Layout

| Path | What it is |
|---|---|
| `chart/` | One Helm chart for every environment: API Deployment + Service, Postgres StatefulSet, DB Secret, migrate Job |
| `envs/<dev\|staging\|prod>/values.yaml` | Per-environment values. Today each one only sets `image.tag` |
| `argocd/api-<env>-env.yaml` | One Argo CD Application per environment: same chart, different values file and namespace |
| `.github/workflows/promote.yaml` | Moves a tag dev → staging → prod |

Same image, different config: environments differ only by their values file, so a promoted
image is byte-for-byte the one that was tested in the environment before it.

## Environments and version visibility

| Env | Namespace | Tag set by | Gate |
|---|---|---|---|
| dev | `dev` | `main.yaml` in the app repo on every merge | Required PR checks |
| staging | `staging` | `promote.yaml`, target `staging` | Manual trigger |
| prod | `prod` | `promote.yaml`, target `prod` | GitHub Environment `production` with a required reviewer |

What runs where can be read in three places: the tag in `envs/<env>/values.yaml`, the image on
the pods (`kubectl get deploy api -n <env> -o jsonpath='{..image}'`), and `GET /version` from
the API itself. Every tag is `sha-<commit>`, so it points straight back to the source code.

## Deploy order inside a sync

Argo CD applies the chart in waves:

| Wave | What | Why |
|---|---|---|
| 0 | Secret, Services, Postgres StatefulSet | The database has to exist first |
| 1 | `migrate` Job (Sync hook, Flyway) | Schema is updated before new code starts |
| 2 | API Deployment | Starts only after migrations succeed |

The migrate Job has `activeDeadlineSeconds: 120`. An image that can't be pulled never counts
against `backoffLimit`, so without a deadline a bad tag would leave the Job hanging forever.
With it, each attempt is stopped after 120 seconds and wave 2 never starts, so the old API
pods keep serving.

## Promotion (`promote.yaml`)

Run it from the Actions tab with `target` = `staging` or `prod`.

1. Works out the source: staging takes from dev, prod takes from staging. You can't skip a stage.
2. Reads `image.tag` from the source env and stops if it is still `none`.
3. Writes that tag into the target env with `yq` and commits as `release-pipeline-bot`
   (`<env>: promote sha-xxx from <source>`). If the target already has the tag, nothing is committed.
4. Prod runs in the `production` GitHub Environment, so the job waits for a reviewer to approve.
   The approval is recorded on the workflow run.

Always checks out `main`, uses the built-in `GITHUB_TOKEN` (`contents: write`) and
`concurrency: promote`, so two promotions can't race each other.

## Failure detection and rollback

**Detection**
- Bad image tag → the migrate Job (wave 1) can't pull it and hits its deadline. Wave 2 never
  runs, so the old API pods keep serving. Argo CD's automated sync then retries the failed
  operation, so the app stays in a retrying sync instead of settling on **Failed**.
- Image pulls but the app is broken → the new pod never passes `/readyz`. The rolling update
  doesn't remove old pods until new ones are Ready, so users stay on the old version, and
  Argo CD shows the app as **Degraded / Progressing**.

**Rollback** is a Git operation, like every other change:

```bash
git log --oneline -- envs/prod/values.yaml   # find the bad bump/promote commit
git revert <commit>                          # new commit that puts the previous tag back
git push
```

Argo CD syncs the previous image. Nothing is rebuilt, because that image is already in GHCR,
and the revert itself stays in the history.

I tested this on dev by pushing a tag that doesn't exist (`sha-0000000`) and reverting it.
While Argo CD was still retrying the failed sync, it did not pick up the revert commit. I had
to **Terminate** the running sync in the UI and **Sync** again; after that, dev was back on the
previous version within a minute. Users were never affected, because the old pods kept serving.
This manual step is why alerts on sync failures and Argo Rollouts are next on the list below.

## Bootstrap

```bash
kubectl apply -n argocd -f argocd/
```

The Applications watch `chart/` and their own `envs/` file. They don't watch `argocd/`, so a
new Application is added with `kubectl apply` once.

## Assumptions

- Local single-node cluster with Argo CD installed and the repo public (no repo credentials).
- Postgres runs in-cluster per environment; in a real setup it would be a managed database.
- Secrets are plain placeholders in the chart, for the local prototype only.

## Known limitations and what I would do next

- **Branch protection on this repo.** Today anyone with write access could edit `envs/prod`
  directly. I would protect `main`, add CODEOWNERS for `envs/prod/`, and let the promote bot
  push through a GitHub App with a bypass. Doing it now would also block the bot.
- **Automatic rollback.** Today rollback is a manual `git revert`. Next would be Argo Rollouts
  with an analysis step that rolls back on failed health checks, plus a `tag` input on
  `promote.yaml` to redeploy a specific known-good version. Argo CD notifications on sync
  failures, so a stuck sync is noticed without someone watching the UI.
- **Smoke tests before promotion**, so staging/prod can only receive a tag that passed in the env before.
- **App-of-apps**, so new Applications are created from Git instead of `kubectl apply`.
- **Secrets** from a secret manager (External Secrets / Sealed Secrets) instead of the chart.
- PR checks on this repo (`helm lint`, `helm template`, schema validation).

## Use of AI

I used an AI assistant (Claude) during this project, as the brief allows: to explain Argo CD
and Helm concepts, review my YAML, help debug, and draft parts of the work, including the
Postgres manifest, parts of the workflows and these READMEs. I ran and tested everything myself
and can walk through every file.
