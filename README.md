# nodejs-demo-app

A minimal Express app whose only job is exercising
[platform-cicd](https://github.com/jfillman/platform-cicd)'s Phase 1 pipeline end to
end: `build -> test -> deploy-to-dev`, with unified tracing and the Grafana dashboards.
Nothing here is meant to be a real product - keep it boring on purpose.

Tenant: `platform-cicd-demo`. Registry: `ghcr.io/jfillman/nodejs-demo-app`.

## What's in here, and why

| File | Platform stage | Purpose |
|---|---|---|
| `server.js` | - | Express app - `GET /` and `GET /healthz` |
| `test/server.test.js` | build (unit test) | Node's built-in test runner (`node --test`), no extra deps |
| `test.sh` | build (unit test) | What `catalog/tasks/run-tests.yaml` invokes - runs `npm ci && npm test` |
| `build.sh` | build (build script) | What `catalog/tasks/build-image.yaml`'s `run-build-script` step invokes, inside the resolved build-agent image, before Dockerfile packages the result - runs `npm ci --omit=dev` |
| `Dockerfile` | build (image) | Thin packaging step - does NOT run npm itself, see note below |
| `integration-test.sh` | test | What `catalog/tasks/run-integration-tests.yaml` invokes - starts the app and curls `/healthz` (see the file's own header for why this tests the code, not the built image) |
| `cicd.yaml` | all | The only file you'd normally hand-edit day to day - see platform-cicd's `docs/cicd-yaml-reference.md` |
| `.tekton/push.yaml`, `.tekton/pull-request.yaml` | trigger | Generated boilerplate - never hand-edited, see their own headers |
| `k8s/dev-deployment.yaml` | deploy | One-time manual apply during onboarding - see the file's own header |

## Architecture change this app drove

`cicd.yaml`'s `build.script` field is required by `schemas/cicd.schema.json`, but an
earlier version of `catalog/tasks/build-image.yaml` never actually invoked it - the
build stage jumped straight to kaniko building `Dockerfile` as-is. That's now fixed in
platform-cicd: `build-image.yaml` runs `build.script` inside the resolved build-agent
image *before* kaniko packages the result. The corresponding split in this repo:
`build.sh` does the actual build (`npm ci`), `Dockerfile` just copies what's already
been built. Don't reintroduce a second `npm ci` inside the Dockerfile - that would just
redo `build.sh`'s work.

## Onboarding status

App, `cicd.yaml`, and the generated `.tekton/` boilerplate are ready. Still needed
before a push actually flows end to end - see platform-cicd's `docs/onboarding.md`:

1. Apply `platform-cicd/platform/broker/manifests/tenant-triggers-template.yaml` for
   tenant `platform-cicd-demo` (namespace, RBAC, the build->test->deploy `Trigger` CRs).
2. `kubectl apply -f k8s/dev-deployment.yaml`.
3. Create the `registry-credentials` Secret (ghcr.io push auth) in the
   `platform-cicd-demo` namespace - **pending**: needs a GitHub token/App with
   `write:packages` scope, not yet provided.
4. If the GHCR package ends up private, also create a pull secret in
   `platform-cicd-demo-dev` (see `k8s/dev-deployment.yaml`'s commented
   `imagePullSecrets`) - or just make the package public and skip this.
5. Push this repo to `github.com/jfillman/nodejs-demo-app`, install the platform's
   GitHub App on it, create the PaC `Repository` CR.
