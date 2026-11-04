# Beacon release review

## Pipeline audit
Gap / risk / repaired dependency:
The `production` job ran in parallel with `validate` and did not declare `needs: validate`, so a failed validation did not block the production
review job (`check_release.py pipeline` reported `pipeline: FAIL`, exit 1). Risk: a broken or untested change could reach the production
gate stage even when tests/validation fail. Fix: added `needs: validate` to the `production` job in `.github/workflows/release.yml` so
production only runs after `validate` succeeds (`check_release.py pipeline` now reports `pipeline: PASS`, exit 0). No other job behavior,
environment, or secret wiring was changed.
Production reviewer control to configure (not demonstrated locally):
The workflow references `environment: production`, but an environment name alone does not enforce approval. Separately, in the GitHub repo
settings (Settings -> Environments -> production -> Environment protection rules), "Required reviewers" must be enabled and at least one
(or the agreed set of) reviewer(s) added, plus optionally a wait timer / branch restriction. Until that setting is configured in GitHub,
any run with merge rights can enter the production environment without a human approval. This was not done or demonstrated here; it is a
repository/environment configuration step outside the workflow file.
Secret handling and scope:
`REGISTRY_TOKEN` is referenced only as `${{ secrets.REGISTRY_TOKEN }}` in the production job's step env and is never echoed, printed, or
written into logs; the step only prints a fixed "review only" string. The secret should be stored as a production-environment secret
(Settings -> Environments -> production -> Secrets), not as a repo-wide or organization-wide secret, so it is injected only into jobs that
declare `environment: production`. Least privilege: scope it to the registry (read/pull or the minimum push permission actually needed),
rotate it on a schedule and on suspected exposure, and restrict the production environment with required reviewers and branch/tag
deployment rules so only approved, protected-branch refs can use it. Values were not read, printed, or committed; do not paste them into
this file, issues, PR comments, or CI logs.

## Traceability
Why PATCH is appropriate:
release-contract.json lists candidate v2.3.1 over known-good v2.3.0 with release_kind "backward-compatible bug fix". Under semver, a
backward-compatible fix with no new features and no breaking changes is a PATCH increment (2.3.0 -> 2.3.1). Minor/major would imply new
backwards-compatible functionality or a breaking API/behavior change, which this release does not claim.
Candidate image and annotated Git tag:
Candidate image: beacon-service:v2.3.1 (as pinned in k8s/deployment.json, changed from the floating `latest` tag; workload name,
selector/pod labels `app: beacon`, and containerPort 8080 are unchanged). The annotated Git tag for this release is v2.3.1, to be created
only after all repairs and this readiness file are complete. It must be verified to resolve to the final code commit with
`git rev-parse v2.3.1` (tag object) and `git rev-parse v2.3.1^{commit}` (commit it points to) — record both values. Image-tag does not
equal commit proof; the tag only identifies which source commit the release notes describe.
Image-tag mutability limitation:
A tag like v2.3.1 is a mutable, human-assigned pointer: it can be re-pushed, overwritten, or deleted in a registry, and `latest` can point
at anything at any time. A verified digest (`beacon-service@sha256:<hash>`) is content-addressed and immutable — it identifies the exact
image bytes regardless of what the tag says. Even so, a digest only proves image content, not provenance: neither tag nor digest proves the
image was built from this commit. Provenance requires registry-side verification (immutable tags, digest pinning after build, signatures/
attestations such as cosign/SLSA, or a verification step in CI), which was not performed or verified here. No image build or registry
check was run; the tag reference satisfies the static release contract only.

## Incident recovery
Evidence and rollback vs forward decision:
All incident data below is a supplied training fixture (incident-fixture.md), not observed runtime evidence — nothing was deployed, observed,
or rolled back here. Fixture signals: 10:00 candidate v2.3.1 promoted (v2.3.0 previously healthy); 10:03 ~35% of requests failing across all
users and rising; two of three readiness checks failing; root cause not yet understood; v2.3.0 available and verified healthy in the
scenario; no database/schema migration and rollback compatibility confirmed in the fixture. Decision: ROLLBACK, not forward-fix. Reason:
widespread user impact with rising error rate, unknown root cause, no schema/migration coupling to undo, and a known-good image confirmed
compatible — rolling back restores service fastest while the cause is investigated. A forward fix would ship unverified code into an actively
degrading incident with an undiagnosed root cause.
Known-good target:
beacon-service:v2.3.0 (release-contract.json known_good_image; rollback.json target_image).
Proposed kubectl command (not executed):
kubectl set image deployment/beacon beacon=beacon-service:v2.3.0
(Not run against any cluster. After registry digest verification this should pin the digest, e.g. beacon=beacon-service@sha256:<known-good>.)
Recovery checks after action:
kubectl rollout status deployment/beacon; kubectl get pods -l app=beacon; readiness probe status (expect 3/3 passing); error rate and 5xx
ratio back to baseline; logs free of the new failure signature; confirm image in spec is v2.3.0; watch via metrics for a sustained window,
and only then root-cause the v2.3.1 regression before any re-promotion.

## Release note and decision
Change scope:
Release-safety repairs only: (1) .github/workflows/release.yml — production job now declares `needs: validate`; (2) k8s/deployment.json —
image pinned from floating `latest` to beacon-service:v2.3.1 (names, labels, ports unchanged); (3) rollback.json — target_image set to
known-good beacon-service:v2.3.0; (4) this readiness.md. No app code, tests, ports, labels, or workload names changed.
Local validation proof:
`py check_release.py pipeline` -> PASS (exit 0); `py check_release.py image` -> PASS (exit 0); `py check_release.py rollback` -> PASS (exit 0)
(`python3` is not on PATH, so `py` was used). `py -m unittest -v` -> OK (1 test). No cluster, registry, approval, or deployment was
verified; these are static contract checks and the unit test only.
Open risk / owner / mitigation:
Open risk: the release tag/image reference (v2.3.1) is mutable and unverified — nothing proves the image bytes match this commit. Owner:
release engineer / service owner. Mitigation: build and push in CI, then pin the verified sha256 digest (and enable signatures/attestation,
e.g. cosign) in the deployment before any promotion; keep immutable tags in the registry. Separately, production approval requires the
GitHub "Required reviewers" environment protection rule, which is still NOT configured here.
Current production decision and blocker:
NO-GO. approval_status is NOT REQUESTED and deployment_status is NOT DEPLOYED in release-contract.json. Passing local static checks does
not establish production readiness: no reviewer approval, no registry/digest verification, no cluster rollout or smoke test, and the incident
is an unexercised hypothetical. Blocker: production Environment required-reviewer protection and explicit approval are not in place; the
candidate has not been deployed or validated at runtime.
Terraform vs Kubernetes responsibilities:
Terraform provisions and versions the underlying infrastructure (clusters, node pools, IAM, environment scaffolding) as declarative,
state-managed changes, while Kubernetes operates the workload itself — scheduling pods, rolling out the new image, readiness/liveness
health, and rollbacks within that provisioned cluster.
