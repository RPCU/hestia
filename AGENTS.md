# Hestia Project Guide for AI Agents

## ⚠️ CRITICAL INSTRUCTIONS FOR AI AGENTS

1. **Commit policy — do NOT commit unless explicitly asked.** Preview changes,
   show `git diff`, list files, and draft a commit message for approval first.
2. **Documentation policy — ALWAYS update this file when you change the
   project.** Add/update the relevant Section 1 entry, versions, directory
   structure, and the "Last Updated" line, in the same request you ask to commit.
3. **Never inline a real secret.** Everything sensitive comes from Vault via
   External Secrets Operator (`vault-backend` ClusterSecretStore). Section 8
   lists the upstream demo credentials that are still literal — do not add more.

## Project Overview

**Hestia** is RPCU's GitOps repo for the **platform layer**: an
[OpenChoreo](https://openchoreo.dev) internal developer platform reconciled by
Flux CD onto the `platform` cluster. It ships the OpenChoreo control plane
(API + Backstage portal + cluster-gateway), the workflow plane (Argo Workflows
build pipelines), the observability plane (OpenSearch logs/traces + Prometheus
metrics + observer), ThunderID as the OIDC provider, CloudNativePG for the
Backstage database, and the tenant-facing OpenChoreo abstractions
(ComponentTypes, Traits, Workflows, Environments, Projects).

**Hestia is a consumer, not a provider, of cluster infrastructure.** The
substrate — Kubernetes itself, Flux, cert-manager + the `vault-issuer`
ClusterIssuer, External Secrets + `vault-backend`, the `rpcu-ca-trust` CA
bundle, kgateway/Gateway API, StorageClasses — is owned by
[argus](https://github.com/RPCU/argus) (`../argus`). If something you need does
not exist on the cluster and is not in this repo, it belongs in argus.

**Division of labour vs argus:** argus builds and operates *clusters*; hestia
builds and operates the *platform that runs on top of one*. The two repos meet
at three seams: the `vault-backend` ClusterSecretStore + `vault-issuer`
ClusterIssuer + `rpcu-ca-trust` ConfigMap (argus → hestia), the
`kgateway-system` external gateway (hestia's ClusterProjectType punches a
NetworkPolicy hole for it), and the `chihiro` component, whose config template
drives argus's Sveltos add-on labels.

---

## 1. Directory Structure

### Root

`infrastructure/` platform components (Flux Kustomizations + HelmReleases) ·
`openchoreo/` OpenChoreo tenant/platform CRs · `README.md`.

**There is no tooling in this repo** — no `devenv.nix`, no `.pre-commit-config`,
no `renovate.json5`, no `.yamllint`/`.prettierrc`, no CI. Chart versions are
bumped by hand (Section 5). If you introduce tooling, mirror argus's setup and
document it here.

### infrastructure/ — reconciliation root

`infrastructure/kustomization.yaml` is the ordered resource list applied by the
root `hestia` Flux Kustomization. Entries are a mix of **Flux Kustomization
pointers** (to a subdirectory) and **inline HelmReleases/CRs**:

- **fluxcd.yaml** → Kustomization `fluxcd` (`./infrastructure/fluxcd`, 1m, prune,
  5m timeout).
  - `fluxcd/gitrepository.yaml` — GitRepository `hestia`
    (`https://github.com/RPCU/hestia.git`, branch `main`, interval 1m).
  - `fluxcd/flux-kustomization.yaml` — the root Kustomization `hestia`
    (`path: ./infrastructure/`, interval 1m, prune). **Self-referential**: the
    root Kustomization re-declares itself and its own GitRepository, so once
    bootstrapped Flux owns its own sync config. Both objects carry
    `kustomize.toolkit.fluxcd.io/prune: disabled` so a bad reconcile can never
    garbage-collect the sync loop itself. Do NOT remove that annotation.
- **cnpg.yaml** → Kustomization `cnpg` (`wait: true`, 5m) → `cnpg/`:
  HelmRepository `cloudnative-pg` (https) + HelmRelease `cloudnative-pg` v0.29.0
  in ns `cnpg-system`, `install.crds: Create` / `upgrade.crds: CreateReplace`,
  50m/200Mi. `wait: true` here is deliberate — the Backstage `Cluster` CR in
  openchoreo-requirements needs the CNPG CRDs.
- **openchoreo-requirements.yaml** → Kustomization `openchoreo-requirements`
  (10m, 5m timeout). The directory has **no `kustomization.yaml`** — Flux
  auto-generates one from every YAML in the tree. Adding a file is enough; there
  is no index to update.
  - `backstage-cnpg.yaml` — CNPG `Cluster/backstage` (ns
    openchoreo-control-plane): 1 instance, `enablePDB: false`, 6Gi storage,
    256Mi/50m, initdb db+owner `backstage` from Secret `backstage`.
  - `backstage-secret.yaml` — ExternalSecret `backstage`: `postgres-user`/
    `postgres-password` from Vault `backstage/db`, plus `dataFrom.extract` of
    `backstage/db` **and** `backstage/bootstrap` (the latter carries the OAuth
    client secret Backstage presents to ThunderID).
  - `certificate.yaml` — Certificate `wildcard-tls` (ns
    openchoreo-control-plane), `*.platform.rpcu.lan`, ClusterIssuer
    `vault-issuer`. Referenced by the control-plane chart's gateway listener.
  - `observability.yaml` — ns `openchoreo-observability-plane` +
    ExternalSecrets `opensearch-admin-credentials` (Vault `opensearch`
    username/password), `observer-secret`
    (`UID_RESOLVER_OAUTH_CLIENT_SECRET` ← `observer/client-secret`),
    `opensearch-telemetry-writer` (← `opensearch/writer-password`) + the
    namespace's own `wildcard-tls` Certificate + **the OpenSearch internal PKI**:
    `Issuer/opensearch-selfsigned-bootstrap` (selfSigned) →
    `Certificate/opensearch-ca` (isCA, RSA-2048, 87600h) →
    `Issuer/openchoreo-observability-plane-ca-issuer`. That last name is
    **hardcoded in the `observability-logs-opensearch` chart**, which issues its
    http/transport/admin certs from it but does NOT create it. All three certs
    must chain to ONE CA or transport mTLS and admin auth break — hence a real
    CA, not three selfSigned issuers.
  - `thunderid-bootstrap.yaml` — ns `thunderid` + ExternalSecret
    `thunderid-bootstrap` (`backstage-client-secret`, `admin-password`,
    `zitadel-client-secret`, `workflows-client-secret`, `observer-client-secret`).
- **thunderid.yaml** → Kustomization `thunderid`
  (`dependsOn: openchoreo-requirements`, 10m interval/timeout) → `thunderid/`:
  OCI HelmRepository `thunder-id` (`oci://ghcr.io/thunder-id/helm-charts`) +
  HelmRelease `thunderid` v1.0.0 (`targetNamespace: thunderid`,
  `upgrade.remediation.retries: 3`). Values are injected through a
  `configMapGenerator`-built ConfigMap `thunderid-values`
  (`disableNameSuffixHash: true`) rather than inline `values:` — edit
  `thunderid/values.yaml`.
  - Chart migrated from the pre-release `asgardeo/thunder` 0.36.0 (whose gate
    frontend had diverged from the generated `config.js` → blank sign-in page).
  - HTTPRoute on `gateway-default` (ns openchoreo-control-plane),
    `thunderid.platform.rpcu.lan`; chart Ingress off.
  - **SQLite for all FOUR databases** (`config`, `runtime_transient`, `entity`,
    `runtime_persistent` — 1.0.0 added the fourth). Therefore
    `deployment.replicaCount: 1` and `hpa.enabled: false` — a single writer.
  - **Three PVCs, all mandatory**: `persistence` 2Gi (the SQLite DBs),
    `certs.persistence` 16Mi (per-install TLS/JWT/AES key material generated by
    the setup job — losing it invalidates every issued token and all previously
    encrypted data), `secrets.persistence` 16Mi (Direct-Auth-Secret).
  - `server.httpOnly: true` — Thunder serves plain HTTP on :8090; TLS is
    terminated at the gateway. `server.publicUrl` is the canonical issuer.
  - `consoleClient.resourceIdentifier: https://localhost:8090/mcp` — pinned on
    purpose (Section 8).
  - `log.level: debug` — marked temporary in-file; a candidate for removal.
  - `setup.secretEnv` exposes the four Vault-backed client secrets to the
    bootstrap importer's Go-template context as `{{ .BACKSTAGE_CLIENT_SECRET }}`,
    `{{ .ZITADEL_CLIENT_SECRET }}`, `{{ .WORKFLOWS_CLIENT_SECRET }}`,
    `{{ .OBSERVER_CLIENT_SECRET }}`.
  - `bootstrap.scripts` — declarative resource documents (`resource_type: ...`,
    NOT shell scripts) mounted **alongside** the chart's shipped defaults:
    - `70` user_type `openchoreo-user`; `71` four users
      (`admin|developer|platform-engineer|sre@openchoreo.dev`); `72` four groups
      (`admins`, `developers`, `platform-engineers`, `sres`).
    - `73` OIDC `connection` **zitadel** → `https://rpcu-gabeck.eu1.zitadel.cloud`
      (clientId `387085074090190709`), scopes incl.
      `urn:zitadel:iam:org:project:roles`, account linking on `email`.
    - `74` `flow` `openchoreo-federated-auth`: START → OIDCAuthExecutor
      (`allowAuthenticationWithoutLocalUser: true`) → ProvisioningExecutor gated
      on `{{ctx(groups)}} == rpcu-admin` → `admins`, else `developers` →
      AuthorizationExecutor → AuthAssertExecutor → END. **This is where Zitadel
      group → OpenChoreo group mapping happens.**
    - `80` Backstage app (authorization_code + client_credentials + refresh_token,
      redirect `https://console.platform.rpcu.lan/api/auth/openchoreo-auth/handler/frame`);
      `87` Workload Publisher (m2m, used by the build pipeline's
      generate-workload step); `88` Observer resource reader (m2m, used by the
      observability plane's uid-resolver); `81`–`86`, `89`, `90` upstream demo
      apps (customer-portal, rca-agent, CLI, system, user/service MCP, finops,
      mcp-e2e-subject).
    - Those three RPCU-critical apps set `id:` to the **clientId slug**, not a
      UUID like every demo app. That is load-bearing — see Section 8.
- **openchoreo-control-plane.yaml** — OCI HelmRepository `openchoreo`
  (`oci://ghcr.io/openchoreo/helm-charts`, **declared here and reused by the five
  other OpenChoreo HelmReleases** — workflow plane, observability plane, and the
  three observability modules; do not delete this file's first document) +
  HelmRelease `openchoreo-control-plane` v1.2.2.
  - `kubernetesClusterDomain: platform.local` — the cluster domain is
    **not** `cluster.local`. Every in-cluster FQDN in this repo must use
    `.svc.platform.local` or the short `.svc` form.
  - `features.secretManagement.enabled: true` — turns on the
    `SecretReference`/ExternalSecret plumbing the ComponentTypes and Workflows
    depend on.
  - cluster-gateway: served at `https://cluster-gateway...svc:8444` in-cluster
    with `tls.insecure: true` for the openchoreoApi and controllerManager hops,
    because `vault-issuer` only signs `*.platform.rpcu.lan` so the server cert
    carries just the external SAN `cluster-gateway.platform.rpcu.lan`. Client
    mTLS still authenticates the callers. A TLSRoute exposes the external name.
  - `security.oidc` split — **browser-facing endpoints external HTTPS, machine
    hops in-cluster plain HTTP**: `issuer` + `authorizationUrl` →
    `https://thunderid.platform.rpcu.lan`; `jwksUrl` + `tokenUrl` →
    `http://thunderid-service.thunderid.svc.platform.local:8090`.
  - Backstage: `secretName: backstage`, postgres, base URL + hostname
    `console.platform.rpcu.lan`, API over `http://openchoreo-api:8080`.
  - `gateway.tls` → `*.platform.rpcu.lan` / `certificateRefs: [wildcard-tls]`.
- **openchoreo-workflow-plane.yaml** — HelmRelease v1.2.2 (ns
  openchoreo-workflow-plane, `dependsOn: openchoreo-control-plane` because the
  agent-facing cluster-gateway must exist first).
  `clusterAgent.planeID: workflow` (**must match**
  `ClusterWorkflowPlane.spec.planeID`), `serverUrl:
wss://cluster-gateway.platform.rpcu.lan/ws` — the agent uses the external
  hostname (the cert has no internal SAN) and verifies it against
  `rpcu-ca-trust`. `generateCerts: true` → the agent mints its own
  `cluster-agent-tls` CA, whose `ca.crt` the registration CR references. Argo
  controller 100m/128Mi → 500m/512Mi.
- **openchoreo-observability-plane.yaml** — HelmRelease v1.2.2 (ns
  openchoreo-observability-plane). Same cluster-agent pattern with
  `planeID: observability`. Observer: OpenSearch admin creds + `observer-secret`,
  OAuth client `openchoreo-observer-resource-reader-client`, control-plane API at
  `...svc.platform.local:8080`, hostname `observer.platform.rpcu.lan`,
  `AUTHZ_TIMEOUT=30s`, CORS for the console. Gateway adds **TLS passthrough** on
  `opensearch.platform.rpcu.lan:9443` for remote collectors.
- **opensearch-operator.yaml** — HelmRepository (https) + HelmRelease
  `opensearch-operator` v2.8.0 (ns openchoreo-observability-plane,
  `dependsOn: openchoreo-observability-plane`, CRDs CreateReplace).
  `manager.dnsBase: platform.local`. `kubeRbacProxy.image` pinned to
  `quay.io/brancz/kube-rbac-proxy:v0.15.0` — the chart's `gcr.io/kubebuilder`
  default is frequently unavailable.
- **observability-opensearch-users.yaml** — plain `OpensearchRole` +
  `OpensearchUser` CRs (**not** a Flux Kustomization; applied directly by the
  root `hestia` Kustomization). `telemetry-writer` is the least-privilege
  identity handed to the remote data planes' collectors: `cluster_composite_ops`
  + `cluster_monitor`, and on `container-logs-*` / `k8s-events-*` /
  `otel-traces-*` only `create_index` + `write` — no read, delete or manage. A
  compromised data plane therefore cannot read other tenants' telemetry;
  per-tenant read isolation is enforced separately by the observer's OIDC authz.
- **openchoreo-observability-modules.yaml** — three HelmReleases:
  - `observability-logs-opensearch` v0.5.3 (`dependsOn`
    observability-plane + opensearch-operator). `openSearch.enabled: false`,
    `openSearchCluster.enabled: true`. **master `replicas: 3`** — the operator
    seeds a temporary bootstrap node then deletes it; with a single master the
    voting-config handover loses quorum → `cluster_manager_not_discovered`
    deadlock. Data `replicas: 2`. Memory 1Gi req / **2Gi limit** on both pools:
    the chart defaults (900Mi/1Gi) OOMKill OpenSearch 3.3 (~25 bundled plugins;
    auto heap ≈50% of the container + Netty direct buffers + Lucene mmap).
    `fluent-bit: enabled: false` (collectors live on the remote data planes).
  - `observability-metrics-prometheus` v0.6.1 —
    `global.installationMode: multiClusterReceiver`, hostname
    `prometheus.platform.rpcu.lan`, prometheus-operator 20m/64Mi → 256Mi (the
    module default caps it at 60Mi and it OOMs).
  - `observability-tracing-opensearch` v0.6.0 (`dependsOn`
    observability-logs-opensearch) — reuses the logs module's OpenSearch
    (`openSearch.enabled: false`), OTLP receiver on `otel.platform.rpcu.lan`.
- **openchoreo-resources.yaml** → Kustomization `openchoreo-resources`
  (`path: ./openchoreo/fluxcd`, 1m, prune, 5m). The hand-off into the
  OpenChoreo CR tree.

### openchoreo/ — OpenChoreo resources

Three-level Flux fan-out: `openchoreo-resources` → `openchoreo/fluxcd/*` →
per-namespace Kustomizations.

- **fluxcd/namespaces-kustomization.yaml** — Kustomization
  `openchoreo-namespaces` (`./openchoreo/namespaces`, 5m, **`prune: false`**).
  Creates the tenant namespaces.
- **fluxcd/platform-shared-kustomization.yaml** — Kustomization
  `openchoreo-platform-shared` (`./openchoreo/platform-shared`, 5m, prune).
  Cluster-scoped CRs only; deliberately **no `targetNamespace`**.
- **namespaces/rpcu/** — the only tenant namespace today.
  `namespace.yaml` labels it `openchoreo.dev/control-plane: "true"`.
  `flux-kustomization.yaml` declares two more Kustomizations, both
  `targetNamespace: rpcu`, prune, 5m: `openchoreo-platform-rpcu`
  (`.../rpcu/platform`) and `openchoreo-projects-rpcu` (`.../rpcu/projects`).
  Neither `platform/` nor `projects/` has a `kustomization.yaml` — Flux
  auto-generates.
  - **platform/component-types/service.yaml** — `ComponentType/service`
    (workloadType deployment). `allowedWorkflows`: dockerfile / gcp-buildpacks /
    paketo-buildpacks / ballerina-buildpack. `allowedTraits`: api-configuration,
    dragonfly, rbac. Validation: ≥1 endpoint. Renders Deployment + ClusterIP
    Service + external/internal HTTPRoutes + env/file ConfigMaps + ExternalSecret
    env/file mounts via `${dataplane.secretStore}`.
    **Routing shape: path-prefix** — hostname
    `<environmentName>-<componentNamespace>.<gatewayHost>`, path
    `/<componentName>-<endpoint>` with a `ReplacePrefixMatch` URLRewrite to the
    endpoint's `basePath`.
  - **platform/component-types/webapp.yaml** — `ComponentType/web-application`.
    Same resource set, but `allowedWorkflows` additionally includes
    **`nix-builder`**, validation requires an `HTTP` endpoint, and
    **routing shape is subdomain**:
    `oc_dns_label(endpoint, componentName, environmentName, componentNamespace)`
    prefixed onto the gateway host, root path, no rewrite. Pick `service` for
    path-based APIs and `web-application` for anything that needs its own host.
  - Both ComponentTypes embed a **PE-locked** `observability-alert-rule`
    ClusterTrait instance `default-error-rate` (log query `ERROR`, 1m window/
    interval, `gte 1`, severity critical) whose enable/channels/incident knobs
    are wired to a per-env `environmentConfigs.alerting` block on the
    ReleaseBinding. Developers can tune delivery, not the rule.
  - **platform/infra/environments/** — `development`, `staging`, `production`
    (only production `isProduction: true`). **All three** reference
    `dataPlaneRef: {kind: DataPlane, name: test}`.
  - **platform/infra/deployment-pipelines/standard.yaml** — `DeploymentPipeline
standard`: development → staging → production.
  - **platform/traits/dragonfly.yaml** — namespaced `Trait/dragonfly`. Injects a
    **Redis sidecar container** into the Deployment (`redis:7-alpine` default,
    port 6379, 15m/256Mi → 500m/512Mi) plus a ClusterIP Service
    `<name>-<instanceName>`. Despite the name it is not the DragonflyDB operator
    and it is not a separate workload.
  - **platform/traits/rbac.yaml** — namespaced `Trait/rbac`. Creates a dedicated
    ServiceAccount `<name>-<instance>`, a ClusterRole + ClusterRoleBinding
    `<ns>-<name>-<instance>` from `parameters.rules`, and patches
    `spec.template.spec.serviceAccountName`. A dedicated SA (not `default`) so
    the grant is not inherited by every other pod in the cell namespace.
  - **platform/workflows/.gitkeep** — placeholder for namespace-scoped
    `Workflow` CRs; all builders are currently cluster-scoped.
  - **projects/testing/project.yaml** — `Project/testing` (ClusterProjectType
    `standard`, pipeline `standard`) + three `ProjectReleaseBinding`s, one per
    environment.
  - **projects/testing/chihiro.yaml** — the `chihiro` Component
    (`deployment/web-application`) + its `Workload`. Built by `nix-builder`
    from `github.com/RPCU/chihiro` (`./nix/oci.nix`, branch main). Traits:
    `dragonfly/session-store` (the Redis sidecar backing
    `CHIHIRO_REDIS_ADDR=localhost:6379`) and `rbac/capi-viewer` (namespaces +
    secrets read, CAPI cluster/machine* read, cluster create/update/patch/delete,
    infrastructure/controlplane/bootstrap provider read). The Workload mounts a
    large `/config.yaml` describing chihiro's cluster-creation form — **this is
    the UI contract for argus**: `cluster.template` emits a CAPI `Cluster` in ns
    `mgmt` with the `sveltos.argus.rpcu.io/*` add-on labels and the
    `capo-version` annotation. Changing a toggle here without the matching
    ClusterProfile in argus produces a label nothing consumes.
- **platform-shared/** — cluster-scoped OpenChoreo + Argo CRs.
  - `cluster-project-types/default.yaml` — `ClusterProjectType/standard`:
    the per-environment cell Namespace (labels/annotations from
    `ProjectReleaseBinding.spec.environmentConfigs`, guarded with `has()`
    because OpenChoreo does **not** inject openAPIV3Schema defaults into the CEL
    context) **plus** the `allow-kgateway-system-ingress` NetworkPolicy.
    OpenChoreo's built-in per-app NetworkPolicy only admits gateways labelled
    `openchoreo.dev/system-component`; the argus-managed `kgateway-system/https`
    gateway carries no such label, so its Envoy proxies were dropped. Policies
    are additive, so this widens the allow-list instead of patching the built-in.
  - `cluster-traits/observability-alert-rule.yaml` — `ClusterTrait`. Parameters
    (description/severity/source{type,query,metric}/condition{window,interval,
    operator,threshold}) are PE-owned; `environmentConfigs`
    (enabled/notifications.channels/incident{enabled,triggerAiRca,
    triggerAiCostAnalysis}) are per-env. Four validations enforce: a channel is
    mandatory (env default counts), AI RCA and AI cost analysis both require
    `incident.enabled`, and cost analysis only applies to `budget` sources.
    Emits an `ObservabilityAlertRule` on `targetPlane: observabilityplane`,
    `includeWhen: ${has(dataplane.observabilityPlaneRef)}` — so components on a
    data plane without an observability plane simply skip it.
  - `cluster-workflow-templates/argo/` — the Argo `ClusterWorkflowTemplate`
    building blocks. All build steps write `/mnt/vol/app-image.tar` and read
    `/mnt/vol/source`, so they are interchangeable.
    - `checkout-source.yaml` — `checkout`, `alpine/git:v2.52.0`. Optional git
      Secret supports `ssh-privatekey` (incl. CodeCommit key IDs) or
      basic-auth (URL-encoded into an HTTPS remote). Shallow clone by commit or
      branch; outputs an 8-char `git-revision` and a `git-tag` (the git tag
      when the commit is tagged, empty otherwise).
    - `workflow-templates.yaml` — **auto-generated upstream by
      `make workflow-templates-gen`, DO NOT EDIT.** Holds
      `paketo-buildpacks-build`, `gcp-buildpacks-build`,
      `ballerina-buildpack-build`, `containerfile-build`. All run
      `ghcr.io/openchoreo/podman-runner:v1.2` **privileged** with
      `hostUsers: false` and a 10Gi emptyDir at `/storage`. Builders and run
      images are pinned by digest.
    - `nix-build.yaml` — RPCU-authored `nix-build`. Drop-in replacement for
      `containerfile-build`, but **unprivileged**: `nixos/nix:2.24.9`,
      `sandbox = false`, classic `nix-build` (experimental features off) against
      the pinned `nixos-26.05` channel. Handles both `streamLayeredImage` (an
      executable that streams the tar to stdout) and `buildLayeredImage` (a tar
      path), then re-tags with `skopeo copy docker-archive:...` — no daemon, no
      privileged container. Extra `--argstr` pairs come from a `nix-args` JSON
      array.
    - `publish-image.yaml` — `publish-image`: `podman load`, discovers the
      loaded image tag dynamically, retags to
      **`zot.rpcu.io/public/<image-name>:<tag>`** — the tag is the git tag when
      the commit is tagged (e.g. `v1.2.3`), otherwise the commit SHA; push with
      `--tls-verify=true` using the `registry-push-secret` dockerconfigjson
      (a zot htpasswd user from Vault key `registry-push-secret`). zot creates
      repos on push and the `public` path allows anonymous pull, so there is no
      per-package rights management and **no data-plane pull secret**. Outputs
      the full image ref.
    - `generate-workload.yaml` — `generate-workload-cr`. An initContainer copies
      `occ` out of `ghcr.io/openchoreo/openchoreo-cli:latest-dev`; the main
      container runs `occ workload create` (from the repo's `workload.yaml`
      descriptor if present, else a default), fetches a `client_credentials`
      token from `https://thunderid.platform.rpcu.lan/oauth2/token` as
      `openchoreo-workload-publisher-client`, then POSTs (or PUTs on 409) to
      `https://api.platform.rpcu.lan/api/v1/namespaces/<ns>/workloads` and
      annotates the WorkflowRun with `openchoreo.dev/workload`. **RPCU
      customization**: `CLIENT_SECRET` comes from a per-run Vault-backed Secret
      (`{{workflow.parameters.workload-publisher-secret}}`), avoiding a standing
      per-tenant secret or pre-created `workflows-<org>` namespaces. On 409 a
      source-defined workload is fully replaced; an auto-generated one only has
      its container image patched.
  - `workflows/` — the five `ClusterWorkflow` CRs developers pick from. All
    share: `workflowPlaneRef: ClusterWorkflowPlane/default`,
    `ttlAfterCompletion: 1d`, `serviceAccountName: workflow-sa`, a four-step
    pipeline (checkout-source → build-image → publish-image →
    generate-workload-cr), an RWO `workspace` volumeClaimTemplate, and three
    generated ExternalSecrets via `${workflowplane.secretStore}`:
    `<run>-git-secret` (only when `repository.secretRef` is set; shape copied
    from the referenced `SecretReference`), `<run>-registry-push-secret` (Vault
    `registry-push-secret`.`value` → dockerconfigjson) and
    `<run>-workload-publisher-secret` (Vault `thunderid`.`workflows-client-secret`).
    - **RPCU-customized**: `dockerfile-builder` (2Gi workspace) and
      `nix-builder` (**4Gi** workspace) expose an `imageName` parameter that
      defaults to the component name → `zot.rpcu.io/public/<component>`.
    - **Upstream-shaped**: `gcp-buildpacks-builder`,
      `paketo-buildpacks-builder`, `ballerina-buildpack-builder` (2Gi) have no
      `imageName` and hardcode
      `image-name = <namespace>-<project>-<component>`. Adding `imageName` to
      them is the obvious consistency fix.
    - `nix-builder` accepts `buildEnv` purely for interface compatibility —
      Nix builds are hermetic and never see it.
  - `infra/workflow-planes/default.yaml` — `ClusterWorkflowPlane/default`. The
    name `default` matches the built-in `ClusterWorkflowPlaneRef` default so
    Workflows resolve without an explicit ref; `planeID: workflow` must match the
    HelmRelease's `clusterAgent.planeID`. `clientCA` via `secretKeyRef`
    (`cluster-agent-tls`/openchoreo-workflow-plane/`ca.crt`),
    `observabilityPlaneRef` → the observability plane (so build logs ship there),
    `secretStoreRef: vault-backend`.
  - `infra/observability-planes/default.yaml` —
    `ClusterObservabilityPlane/default`, `planeID: observability`,
    `observerURL: https://observer.platform.rpcu.lan`. **`clientCA` is an inlined
    PEM**, not a `secretKeyRef` — see Section 8.
  - `authz/role-bindings/.gitkeep` — placeholder. `ClusterAuthzRoleBinding`s
    lived here; the workload-publisher binding was removed in `67a8f01` once it
    matched the chart default. New non-default bindings belong here.

---

## 2. Technologies & Dependencies

- **GitOps**: Flux CD v2.x (source/kustomize/helm controllers), Kustomize, Helm.
  Bootstrap is external; this repo self-manages its GitRepository + root
  Kustomization.
- **IDP**: OpenChoreo v1.2.2 (control plane, workflow plane, observability
  plane) + Backstage (bundled in the control-plane chart).
- **Identity**: ThunderID 1.0.0 (OIDC issuer, SQLite), federated to RPCU's
  hosted Zitadel (`rpcu-gabeck.eu1.zitadel.cloud`).
- **Build**: Argo Workflows (bundled in the workflow-plane chart), Podman
  (privileged) for Dockerfile/buildpack builds, Nix + skopeo (unprivileged) for
  the RPCU `nix-builder`.
- **Registry**: self-hosted zot at `zot.rpcu.io/public` (anonymous pull, push
  via htpasswd creds from Vault).
- **Data**: CloudNativePG 0.29.0 (Backstage Postgres), OpenSearch (via
  opensearch-k8s-operator 2.8.0) for logs + traces, Prometheus for metrics.
- **Secrets**: External Secrets Operator with the `vault-backend`
  ClusterSecretStore (**provided by argus**).
- **Certs / ingress**: cert-manager `vault-issuer` ClusterIssuer and
  kgateway + Gateway API (**provided by argus**); OpenChoreo charts render
  HTTPRoute/TLSRoute.

### Helm Chart Versions

| Component                          | Version | Repository                                     |
| ---------------------------------- | ------- | ---------------------------------------------- |
| openchoreo-control-plane           | 1.2.2   | oci://ghcr.io/openchoreo/helm-charts           |
| openchoreo-workflow-plane          | 1.2.2   | oci://ghcr.io/openchoreo/helm-charts           |
| openchoreo-observability-plane     | 1.2.2   | oci://ghcr.io/openchoreo/helm-charts           |
| observability-logs-opensearch      | 0.5.3   | oci://ghcr.io/openchoreo/helm-charts           |
| observability-metrics-prometheus   | 0.6.1   | oci://ghcr.io/openchoreo/helm-charts           |
| observability-tracing-opensearch   | 0.6.0   | oci://ghcr.io/openchoreo/helm-charts           |
| thunderid                          | 1.0.0   | oci://ghcr.io/thunder-id/helm-charts           |
| cloudnative-pg                     | 0.29.0  | cloudnative-pg.github.io/charts                |
| opensearch-operator                | 2.8.0   | opensearch-project.github.io/opensearch-k8s-operator |

Pinned images: `ghcr.io/openchoreo/podman-runner:v1.2`, `nixos/nix:2.24.9`,
`alpine/git:v2.52.0`, `quay.io/brancz/kube-rbac-proxy:v0.15.0`,
`ghcr.io/openchoreo/openchoreo-cli:latest-dev` (**floating tag**), buildpack
builder/run images pinned by sha256 digest.

Sync intervals: GitRepository + root/fluxcd/cnpg/openchoreo-resources
Kustomizations 1m; openchoreo-requirements/thunderid 10m; `openchoreo/*`
Kustomizations 5m; HelmReleases 5m (cnpg + its HelmRepository 1h).

---

## 3. Key Configuration Details

- **Git**: `git@github.com:RPCU/hestia.git` (the in-cluster GitRepository uses
  the HTTPS URL), branch `main`. Single author history, 60 commits, no CI.
- **Cluster domain**: `platform.local`, **not** `cluster.local`. Set on the
  control plane (`kubernetesClusterDomain`), observability plane and the
  OpenSearch operator (`manager.dnsBase`).
- **DNS**: everything platform-facing is `*.platform.rpcu.lan` —
  `api` (OpenChoreo API), `console` (Backstage), `thunderid`, `observer`,
  `prometheus`, `otel`, `opensearch` (TLS passthrough :9443),
  `cluster-gateway` (agent WSS + TLSRoute). Tenant workloads get
  `<env>-<ns>.<gatewayHost>` (service) or an `oc_dns_label` subdomain (webapp).
- **Namespaces**: `flux-system` (all HelmReleases and Kustomizations live here),
  `cnpg-system`, `thunderid`, `openchoreo-control-plane`,
  `openchoreo-workflow-plane`, `openchoreo-observability-plane`, `rpcu` (tenant
  control-plane namespace).
- **Vault keys consumed** (must exist before Flux converges):
  `backstage/db` (username, password), `backstage/bootstrap` (client-secret),
  `opensearch` (username, password, writer-password), `observer` (client-secret),
  `thunderid` (admin-password, zitadel-client-secret, workflows-client-secret),
  `registry-push-secret` (value = a dockerconfigjson).
- **Formatting**: no enforced formatter. The tracked files use double-quoted
  YAML scalars; the in-flight ComponentType edits use single quotes. Match the
  file you are editing.

---

## 4. Deployment & Sync Process

### Apply order

`infrastructure/kustomization.yaml` list order is not itself an ordering
guarantee — only `dependsOn` and `wait` are. The effective graph is:

1. `fluxcd` (GitRepository + root Kustomization; self-managing).
2. `cnpg` (`wait: true` → CNPG CRDs ready before anything uses them).
3. `openchoreo-requirements` (namespaces, ExternalSecrets, certs, OpenSearch PKI,
   Backstage DB).
4. `thunderid` (`dependsOn: openchoreo-requirements`).
5. `openchoreo-control-plane` (HelmRelease, `createNamespace`).
6. `openchoreo-workflow-plane` (`dependsOn: openchoreo-control-plane`).
7. `openchoreo-observability-plane` (**no `dependsOn`** — see Section 8).
8. `opensearch-operator` (`dependsOn: openchoreo-observability-plane`).
9. `observability-opensearch-users` (bare CRs, **no ordering** — see Section 8).
10. `observability-logs-opensearch` (`dependsOn` observability-plane +
    opensearch-operator) → `observability-tracing-opensearch`
    (`dependsOn` logs); `observability-metrics-prometheus` (`dependsOn`
    observability-plane).
11. `openchoreo-resources` → `openchoreo-namespaces` (prune off) +
    `openchoreo-platform-shared` → `openchoreo-platform-rpcu` +
    `openchoreo-projects-rpcu`.

### Health checks

None. No Kustomization in this repo declares `healthChecks`; only `cnpg` uses
`wait: true`. Everything else converges by retry.

### Local validation

```
kustomize build infrastructure
kustomize build infrastructure/{cnpg,fluxcd,thunderid}
kustomize build openchoreo/namespaces openchoreo/namespaces/rpcu
```

The Flux-only directories (`infrastructure/openchoreo-requirements`,
`openchoreo/fluxcd`, `openchoreo/platform-shared`,
`openchoreo/namespaces/rpcu/{platform,projects}`) have **no
`kustomization.yaml`** and cannot be `kustomize build`-ed; validate them with
`kubectl apply --dry-run=client -f` or `flux build kustomization`.

---

## 5. Making Changes

### Common tasks

- **Bump a chart**: edit the `spec.chart.spec.version` in the relevant
  `infrastructure/*.yaml` (or `infrastructure/<c>/helmrelease.yaml`), update the
  Section 2 table, then `flux reconcile helmrelease <name> -n flux-system
--with-source`.
- **Add a platform prerequisite** (Certificate, ExternalSecret, namespace): drop
  a YAML into `infrastructure/openchoreo-requirements/` — no index to update.
- **Add a builder**: add the Argo `ClusterWorkflowTemplate` under
  `openchoreo/platform-shared/cluster-workflow-templates/argo/` (never inside
  the generated `workflow-templates.yaml`), add the `ClusterWorkflow` under
  `openchoreo/platform-shared/workflows/`, then list it in the relevant
  ComponentType's `allowedWorkflows`. It must consume `/mnt/vol/source` and
  produce `/mnt/vol/app-image.tar`.
- **Add a trait**: namespaced under
  `openchoreo/namespaces/<ns>/platform/traits/` (tenant-specific), or
  cluster-scoped under `openchoreo/platform-shared/cluster-traits/` (fleet-wide);
  then add it to the ComponentTypes' `allowedTraits`. `allowedTraits` entries are
  bare names — a missing trait fails at render time, not at apply time.
- **Onboard a tenant namespace**: create
  `openchoreo/namespaces/<ns>/{namespace.yaml,kustomization.yaml,flux-kustomization.yaml}`
  mirroring `rpcu/`, and add `<ns>/` to
  `openchoreo/namespaces/kustomization.yaml`.
- **Add an OpenChoreo app/client**: add a `bootstrap.scripts` entry in
  `infrastructure/thunderid/values.yaml` (numbered prefix = apply order) and, if
  it needs a real secret, add the key to `thunderid-bootstrap.yaml` + `setup.secretEnv`.

### Dev workflow

feature branch → YAML edits → `kustomize build` the affected targets →
`flux diff kustomization <name> --path ./…` against a live cluster if available →
commit (`feat ✨ (scope): …` / `fix 🐛 (scope): …` / `chore 🧹 (scope): …`,
matching the existing history) → push → PR.

### Dependency updates

Manual. There is no Renovate config here (unlike argus). When bumping
OpenChoreo, bump the control plane, workflow plane and observability plane
together — they share the `openchoreo` HelmRepository and the cluster-agent
protocol.

---

## 6. Git Hooks & Code Quality

None configured. No pre-commit, no linter, no formatter, no GitHub Actions.
Before committing, at minimum run the `kustomize build` targets in Section 4 and
eyeball the CEL expressions in ComponentTypes/Traits — they are only validated
at render time by the OpenChoreo controller.

---

## 7. Documentation & Resources

Docs: <https://docs.rpcu.io/gitops/>. Upstream: openchoreo.dev,
fluxcd.io, argo-workflows.readthedocs.io, cloudnative-pg.io,
opensearch.org/docs, external-secrets.io, gateway-api.sigs.k8s.io.
Sibling repo: `../argus` (cluster infrastructure) — read its `AGENTS.md` before
assuming anything about cert-manager, Vault, kgateway or Sveltos.

---

## 8. Important Notes for AI Agents (traps & incidents)

**Commit policy: DO NOT commit unless explicitly asked** (preview, diff, list
files, draft message). **File safety:** never remove the
`kustomize.toolkit.fluxcd.io/prune: disabled` annotations in
`infrastructure/fluxcd/`; never delete the `openchoreo` HelmRepository document
at the top of `openchoreo-control-plane.yaml` (five other HelmReleases reference
it); never commit secrets. **Cluster safety:** `openchoreo-namespaces` is
`prune: false` on purpose — deleting a tenant namespace is a manual, deliberate
act. Deleting a `ComponentType` or `Trait` that live Components reference breaks
their next render.

### Ordering & dependency gaps

- **`observability-opensearch-users.yaml` has no ordering.** It contains bare
  `OpensearchRole`/`OpensearchUser` CRs applied by the root `hestia`
  Kustomization, while the CRDs arrive with the `opensearch-operator`
  HelmRelease. On a cold bootstrap the root Kustomization fails with
  `no matches for kind "OpensearchRole"` until the operator lands. It converges
  by retry, but the whole root Kustomization is `Ready=False` meanwhile — do not
  chase that as a separate bug. Moving these CRs into their own Kustomization
  with `dependsOn: opensearch-operator` is the clean fix.
- **`openchoreo-observability-plane` has no `dependsOn: openchoreo-control-plane`**
  even though it registers a cluster agent against the control plane's
  cluster-gateway and reuses the `openchoreo` HelmRepository. The workflow plane
  does declare it. Same convergence-by-retry situation.
- **`openchoreo-namespaces` and `openchoreo-platform-shared` are unordered**
  despite the in-file comment claiming namespaces sync "before platform/projects".
  There is no `dependsOn`. Namespaced CRs in `rpcu` can transiently fail until
  the namespace exists.
- **`openchoreo-platform-rpcu` and `openchoreo-projects-rpcu` are unordered.**
  Projects referencing a `ComponentType`/`DeploymentPipeline` from `platform/`
  can render-fail on first apply.

### Missing / external references

- **`DataPlane/test` is not in this repo.** All three Environments point at it.
  It must be registered out-of-band in the `rpcu` namespace (agent CA, gateway
  hostnames, `secretStore`, `observabilityPlaneRef`). Without it, every
  ReleaseBinding fails to resolve, and `${dataplane.secretStore}` in the
  ComponentTypes renders empty.
- **`api-configuration` trait is not in this repo** but is listed in both
  ComponentTypes' `allowedTraits` — it comes from the OpenChoreo charts.
- **`chihiro-secrets`**: `chihiro.yaml`'s env block reads
  `clientId`/`clientSecret`/`sessionKey` from a Secret named `chihiro-secrets`
  via a raw `valueFrom.secretKeyRef`, and an in-file comment attributes it to a
  `secrets` trait. **No such trait exists here** (only `dragonfly` and `rbac` are
  declared, and only those two are attached). Either add the trait or move the
  values to the ComponentType's `secret-env-external` path
  (`configurations.toSecretEnvsByContainer()`), which is the supported
  ExternalSecret mechanism.
- **Registry host**: `publish-image` pushes to `zot.rpcu.io/public`.
  The `chihiro` Workload defaults to `zot.rpcu.io/public/chihiro:latest`; the
  build pipeline's `generate-workload` step overwrites it with the
  `<git-revision>` tag (full commit SHA) or git tag (e.g. `v1.2.3`) on every
  build.

### Certificates & TLS

- **The `ClusterObservabilityPlane` clientCA is an inlined PEM with an expiry.**
  `openchoreo/platform-shared/infra/observability-planes/default.yaml` inlines
  `cluster-agent-tls`'s CA because of
  [openchoreo#4579](https://github.com/openchoreo/openchoreo/pull/4579): the
  resolver only reads `clientCA.value`, so a `secretKeyRef` resolves to empty and
  the control-plane finalizers fail with `agent mode must be enabled`. The
  pinned cert is valid **2026-08-25 → 2026-11-23**. When
  `clusterAgent.tls.generateCerts` rotates the CA (or the agent Secret is
  recreated), this value must be refreshed by hand or the plane silently stops
  authenticating. The workflow plane uses `secretKeyRef` and needs no such
  workaround — do not "harmonise" them without re-testing.
- **`tls.insecure: true` on the in-cluster cluster-gateway hops is deliberate.**
  `vault-issuer` (pki-int) only signs `*.platform.rpcu.lan`, so the gateway's
  server cert has no `.svc` SAN and in-cluster clients cannot hostname-verify it.
  Client mTLS still authenticates the caller and the hop never leaves the
  cluster. Fixing it properly means adding the internal SAN to the issuer's
  allowed domains in argus.
- **OpenSearch needs ONE CA for three certs.** The chart issues
  `opensearch-http`, `opensearch-transport` and `opensearch-admin` from a
  namespaced Issuer named `openchoreo-observability-plane-ca-issuer` that it does
  not create. Replacing the dedicated `opensearch-ca` with per-cert selfSigned
  issuers breaks transport mTLS between nodes and admin authentication.
- `generate-workload.yaml` and `publish-image` use `curl -sk` / accept the
  platform CA implicitly. The `-k` is a known shortcut, not a requirement.

### OpenSearch sizing

- **3 masters is a hard floor, not redundancy theatre.** The operator seeds a
  temporary bootstrap node then deletes it as soon as the StatefulSet pods
  exist, relying on voting-config auto-reconfiguration. With one master the
  handover loses quorum (bootstrap + master = 2 voters; removing the bootstrap
  leaves a lone node that cannot elect itself) → permanent
  `cluster_manager_not_discovered`.
- **2Gi memory limits, not the chart's 900Mi/1Gi.** OpenSearch 3.3 ships ~25
  plugins; auto heap ≈50% of the container plus off-heap (Netty direct buffers,
  Lucene mmap, thread stacks) overflows the default → OOMKilled (137) crashloop.
- The in-file comment says "Keep a single data node to stay light (yellow
  health)" while `data.replicas: 2`. The comment is stale; the value is what
  runs.
- `observability-metrics-prometheus` caps prometheus-operator at 60Mi by default
  and OOMs; the override to 64Mi/256Mi is required.

### ThunderID

- **SQLite ⇒ exactly one replica.** `deployment.replicaCount: 1` and
  `hpa.enabled: false` are correctness constraints, not cost-saving. Scaling up
  corrupts the databases.
- **All three PVCs are load-bearing.** `certs` holds per-install TLS/JWT/AES key
  material generated by the setup job: losing it invalidates every issued token
  and everything previously encrypted. `persistence` holds the four SQLite DBs.
  `secrets` holds the Direct-Auth-Secret.
- **`consoleClient.resourceIdentifier` is pinned to
  `https://localhost:8090/mcp`** on purpose. The Console sends an RFC 8707
  resource indicator that must match a registered resource server; the chart's
  shipped "System" resource server hardcodes that identifier (it does *not*
  template `publicUrl`, despite the chart comment), while the console otherwise
  derives `<publicUrl>/mcp` — the mismatch returns `invalid_target` on
  `/console`. Do not "fix" it to the public URL.
- **Upstream demo credentials are still literal** in `values.yaml`:
  `Dev@123` / `PE@123` / `SRE@123` for the developer/platform-engineer/sre
  users, and hardcoded client secrets for `customer-portal-client`
  (`supersecret`), `openchoreo-rca-agent`, `openchoreo-system-app`,
  `service_mcp_client`, `openchoreo-finops-agent`, `mcp-e2e-subject-client`.
  Only `admin`, Backstage, Zitadel, workload-publisher and observer are
  Vault-backed. Treat the rest as untrusted demo data; do not add more.
- **`log.level: debug`** is marked temporary in-file (it was added to capture the
  federated `groups` claim). Verbose auth logging in a production issuer is a
  standing risk.
- **Zitadel group mapping lives in the `74-federated-flow` bootstrap script**,
  not in RBAC: `{{ctx(groups)}} == "rpcu-admin"` → `admins`, everything else →
  `developers`. Changing group names in Zitadel without editing this flow
  silently downgrades admins to developers.
- **An m2m token's `sub` is the app's bootstrap `id`, not its `clientId`.**
  This bit RPCU once: `87-workload-publisher-app` originally used a UUID `id`, so
  the token `sub` was that UUID, the chart-shipped `workload-publisher-binding`
  (which matches `sub == openchoreo-workload-publisher-client`) never fired, and
  `generate-workload` got HTTP 403. The fix in `67a8f01` was to set `id:` equal
  to the clientId slug and delete the custom `ClusterAuthzRoleBinding`. The same
  applies to `80-backstage-app` and `88-observer-app`. **Never "tidy" those three
  `id:` values into UUIDs**, and give any new app that needs a shipped
  ClusterAuthzRole a slug `id` too. Changing an existing `id` requires a
  ThunderID reinstall (the importer keys on it).

### OpenChoreo rendering

- **openAPIV3Schema defaults are NOT injected into the CEL context.** The
  ClusterProjectType comment documents this the hard way: guard every optional
  `environmentConfigs.*` key with `has()` or the render fails with "no such key".
  Both ComponentTypes and the alert trait already do this — copy the pattern.
- **`allowedTraits` uses bare names**, `traits[]` uses `{kind, name}`. A trait
  referenced but not installed fails at render time, when a ReleaseBinding is
  reconciled — not at `kubectl apply`.
- **`service` vs `web-application` differ only in routing and allowed
  workflows.** Everything else (Deployment/Service/ConfigMaps/ExternalSecrets) is
  duplicated verbatim across the two files. Any change to one almost certainly
  belongs in the other; the two drift easily.
- **The embedded `default-error-rate` alert fires on any `ERROR` log line
  within 1m** on every service and web-application component. It requires a
  notification channel (validation), so a data plane whose Environment has no
  `defaultNotificationChannel` and a component with no
  `environmentConfigs.alerting.channels` will fail validation, not silently skip.
- **Cell namespaces need the kgateway NetworkPolicy hole.** OpenChoreo's built-in
  per-app policy only admits gateways labelled
  `openchoreo.dev/system-component`; the argus-managed gateway has no such label.
  Any new ClusterProjectType must carry `allow-kgateway-system-ingress` (or an
  equivalent) or external traffic times out at connect.

### Build pipeline

- **`workflow-templates.yaml` is generated upstream** (`make
workflow-templates-gen`). Local edits are lost on the next regeneration — put
  RPCU changes in a sibling file (as `nix-build.yaml` does).
- **Podman-based builders run privileged**; `nix-build` does not. Prefer
  `nix-builder` where the app ships a Nix expression.
- **`nix-build` disables the Nix sandbox** (`sandbox = false`) and fetches the
  `nixos-26.05` channel at build time, so builds are reproducible-ish but not
  network-isolated. Bumping the channel changes every image.
- **Image naming is inconsistent across builders** (see §1
  `platform-shared/workflows/`): dockerfile/nix use `<component>`, the three
  buildpack builders use `<namespace>-<project>-<component>`. Two components with
  the same name in different projects collide under the dockerfile/nix builders.
- **`openchoreo-cli:latest-dev` is a floating tag** in `generate-workload.yaml`.
  A CLI regression upstream breaks every build with no repo change.
- Build workspaces are RWO PVCs (2Gi, 4Gi for Nix) and the podman builders add a
  10Gi emptyDir at `/storage`. Large images fail on the emptyDir, not the PVC.

### chihiro ↔ argus coupling

- `chihiro.yaml`'s `cluster.template` writes `sveltos.argus.rpcu.io/*` labels and
  the `sveltos.argus.rpcu.io/capo-version` **annotation**. Each toggle must have a
  matching Sveltos ClusterProfile in argus; each `parameters.*` entry drives a
  form field. `capoversion` and `imagename` are `select` fields on purpose —
  chihiro hard-errors on an empty `{{ chihiro.* }}` create-form placeholder.
- `imagename` options carry `constrain: {version: [...]}` bindings to the
  Kubernetes versions in `CHIHIRO_AVAILABLE_VERSIONS` (`v1.36.1,v1.35.4`).
  Adding a version means updating both, plus the image names.
- The `rbac/capi-viewer` trait grants **cluster create/update/patch/delete** on
  `cluster.x-k8s.io` — that is a genuine cluster-lifecycle privilege on the
  management cluster, scoped to chihiro's own ServiceAccount. Do not widen it.

---

## 9. Summary

GitOps repo for RPCU's OpenChoreo-based internal developer platform: control
plane + Backstage, Argo-based build pipelines (Dockerfile, buildpacks, Nix),
OpenSearch/Prometheus observability, ThunderID OIDC federated to Zitadel, and
the tenant abstractions (ComponentTypes, Traits, Workflows, Environments,
Projects) developers actually consume. Reconciled by Flux at 1–10m intervals
onto a cluster whose infrastructure is owned by argus.

**Repository**: <https://github.com/RPCU/hestia.git> · **Branch**: main ·
**Cluster domain**: `platform.local` · **Public suffix**: `*.platform.rpcu.lan`.

---

**Last Updated**: September 2026 — Initial AGENTS.md. Documents the full
`infrastructure/` reconciliation chain (fluxcd self-management → cnpg →
openchoreo-requirements → thunderid → control/workflow/observability planes →
opensearch-operator → observability modules → openchoreo-resources) and the
`openchoreo/` CR tree (`fluxcd/` fan-out, the `rpcu` tenant namespace with its
`service`/`web-application` ComponentTypes, `dragonfly`/`rbac` Traits,
environments + standard pipeline, and the `testing` project with the `chihiro`
component; `platform-shared/` with the `standard` ClusterProjectType, the
`observability-alert-rule` ClusterTrait, the Argo ClusterWorkflowTemplates
including the RPCU-authored unprivileged `nix-build`, the five ClusterWorkflows,
and the workflow/observability plane registrations). Section 8 records the
current traps: the unordered `observability-opensearch-users` /
`openchoreo-observability-plane` / namespaces-vs-platform-shared Kustomizations,
the missing `DataPlane/test` and `chihiro-secrets`/`secrets`-trait references,
the expiring inlined
`ClusterObservabilityPlane` clientCA (valid to 2026-11-23), the deliberate
`tls.insecure` cluster-gateway hops, the OpenSearch 3-master/2Gi sizing floors,
ThunderID's single-replica SQLite + three load-bearing PVCs + pinned
`consoleClient.resourceIdentifier` + remaining upstream demo credentials, the
CEL `has()` requirement, the kgateway NetworkPolicy hole, and the
generated-file / floating-tag / image-naming hazards in the build pipeline.
At time of writing the working tree also carries uncommitted work: the
`dragonfly`/`rbac` traits, the `nix-build` template + `nix-builder` workflow, the
`chihiro` component, and the `allowedTraits` additions to both ComponentTypes.
