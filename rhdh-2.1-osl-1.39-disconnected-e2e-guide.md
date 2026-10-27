# Disconnected RHDH 2.1 + OSL 1.39 E2E guide for CRC

This runbook installs and tests Red Hat Developer Hub (RHDH) 2.1 with
Orchestrator on a local OpenShift Local/CRC cluster without Internet access at
deployment and test time. It uses a registry running on the CRC host as the
cluster's mirror.

The tested component versions are:

| Component | Version or artifact |
| --- | --- |
| RHDH | 2.1.0 image, Helm chart `redhat-developer-hub-3.4.13.tgz` |
| Orchestrator infrastructure | `redhat-developer-hub-orchestrator-infra-0.7.3.tgz` |
| OpenShift Serverless Logic (OSL) | released 1.39.0, `stable` channel |
| OpenShift Serverless | released `stable` channel |
| OpenShift/CRC | 4.22 in this lab |
| Local registry | `host.crc.testing:5000` |

Do not use the OSL 1.39.0.CR3 engineering-registry images for this procedure.
Those images require access to Red Hat engineering registries, so an
`unauthorized` response cannot be fixed by retrying `oc-mirror`. Use released
OSL 1.39.0 content instead.

## What the test proves

After the disconnect point, the procedure proves that the following work
without Internet access:

1. OpenShift pulls the RHDH, PostgreSQL, Serverless, and Logic images from the
   local registry.
2. The RHDH dynamic-plugin init container obtains its plugin catalog and
   plugins from the local registry.
3. RHDH can issue a Guest token and the Orchestrator UI can call its backend.
4. RHDH reaches SonataFlow Data Index and Jobs Service in the cluster.

The RHDH chart creates the `SonataFlowPlatform`, Data Index, and Jobs Service.
It does **not** supply an application workflow. A `SonataFlow` YAML is required
only when the test must list and run an actual workflow definition.

## Prerequisites

### Required host tools

Install these before disconnecting the workstation.

| Tool | Why it is needed |
| --- | --- |
| `crc` | Runs the local OpenShift cluster. |
| `oc` | Administers OpenShift and applies generated mirror resources. |
| `oc-mirror` v2 | Creates and imports the OSL/operator/RHDH image archive. |
| `helm` 3 | Installs Orchestrator infrastructure and RHDH. |
| `podman` | Runs the local registry and inspects local images. |
| `skopeo`, `jq`, GNU `tar` | Export and import RHDH dynamic-plugin OCI artifacts. |
| `curl`, `grep`, `sed` | Performs lightweight health and configuration checks. |
| `git` | Optional; obtains the RHDH chart source when chart archives do not already exist. |

The host must have enough free disk space for CRC, the `oc-mirror` archive,
unpacked cache, dynamic plugins, and the registry storage. Assign the CRC VM
at least 8 vCPUs, 20 GiB of memory, and a 60 GiB disk. Reserve additional host
disk space for the archive, cache, plugins, and registry storage. That space is
not part of the 60 GiB CRC disk.

### Configure CRC before starting it

If CRC was set up before, stop it, then remove its existing cluster state.

**CAUTION:** `crc cleanup` deletes the cluster state. After it completes, redo
all cluster-side steps in this guide.

```bash
crc stop
crc cleanup
```

For a new or reset instance, set the configuration before you run `crc setup`.
The setting lets the cluster reach the local registry at
`host.crc.testing:5000`.

```bash
crc config set host-network-access true
crc config set cpus 8
crc config set memory 20480
crc config set disk-size 60
crc config view
crc setup
```

Then continue with step 8 to start CRC and log in.

### Required access and identities

* A connected-machine entitlement that can pull `registry.redhat.io` operator
  content and the RHDH images/plugins from Quay.
* A CRC cluster-admin identity (`kubeadmin`) for image configuration,
  ImageDigestMirrorSets, catalog sources, and operator installation.
* A project-capable identity for the `rhdh` namespace. `developer` is suitable
  only if it has the required namespace permissions.
* No OpenShift-registry password is needed for this guide: it uses the
  unauthenticated local registry on port 5000. Do not reuse the OpenShift
  `developer` password for `podman login`.

### Source repository

Clone the source repository on the connected workstation, then enter it:

```bash
git clone https://github.com/rostalan/rhdh-disconnected-lab.git \
  "$HOME/redhat/rhdh-disconnected-lab"
cd "$HOME/redhat/rhdh-disconnected-lab"
```

### Tracked configuration

Phase 1 reads the three version-controlled YAML files in
`disconnected-lab-assets/config`. They set the release pins and lab options:

```text
disconnected-lab-assets/
├── config/
│   ├── imageset-rhdh-2.1-osl-1.39.0.yaml
│   ├── values-rhdh-2.1-osl-1.39.0.yaml
│   └── values-orchestrator-infra-osl-1.39.0.yaml
```

Steps 1–6 create all other artifacts while the workstation is connected.
The chart archives, image mirror, plugin helper and export, and registry image
are build outputs. Do not add them to the repository.

From the repository root (`$HOME/redhat/rhdh-disconnected-lab`), set these
variables once per shell:

```bash
export REPO_ROOT="$PWD"
export WORK="$REPO_ROOT/disconnected-lab-assets"
export CONFIG="$WORK/config"
export SCRIPT_DIR="$WORK/scripts"
export OCMIRROR="$(command -v oc-mirror)"
test -x "$OCMIRROR"
export MIRROR='host.crc.testing:5000'
export RHDH_NS='rhdh'
export OSL_DIR="$WORK/osl-1.39.0-ga"
export IMAGESET_CONFIG="$CONFIG/imageset-rhdh-2.1-osl-1.39.0.yaml"
export RHDH_VALUES="$CONFIG/values-rhdh-2.1-osl-1.39.0.yaml"
export INFRA_VALUES="$CONFIG/values-orchestrator-infra-osl-1.39.0.yaml"
```

## Phase 1: prepare the disconnected bundle while connected

Keep the workstation connected for this phase.

### 1. Create and verify the chart archives

Create the archives while connected. Both charts come from the same pinned
`release-2.1` source revision. The RHDH chart needs its dependencies before
Helm packages it.

```bash
mkdir -p "$WORK/charts"

export SOURCE_COMMIT='bc8ff1ff6b8345934d67fa3638e78750d37fd5cd'
export CHART_SOURCE="$HOME/redhat/rhdh-chart-release-2.1"

git clone --depth 1 --branch release-2.1 \
  https://github.com/redhat-developer/rhdh-chart.git "$CHART_SOURCE"
git -C "$CHART_SOURCE" checkout "$SOURCE_COMMIT"
```

#### Create the RHDH chart archive

Build the RHDH chart dependencies. Package the chart after the build finishes.

```bash
helm dependency build "$CHART_SOURCE/charts/rhdh"
helm package "$CHART_SOURCE/charts/rhdh" --destination "$WORK/charts"

helm show chart "$WORK/charts/redhat-developer-hub-3.4.13.tgz"
```

This creates `redhat-developer-hub-3.4.13.tgz` in `$WORK/charts`.

#### Create the Orchestrator Infrastructure chart archive

Package the Orchestrator Infrastructure chart from the same source checkout.

```bash
helm package "$CHART_SOURCE/charts/orchestrator-infra" \
  --destination "$WORK/charts"

helm show chart \
  "$WORK/charts/redhat-developer-hub-orchestrator-infra-0.7.3.tgz"
```

This creates `redhat-developer-hub-orchestrator-infra-0.7.3.tgz` in
`$WORK/charts`.

### 2. Authenticate to the source registries

Use credentials that are allowed to pull the released content. Do not put
passwords in shell history or this document.

```bash
podman login registry.redhat.io
podman login quay.io
```

### 3. Create the OSL/operator/RHDH image archive

The image-set configuration selects the released Serverless and Logic
operators from the Red Hat 4.22 catalog and adds the RHDH and PostgreSQL image
digests.

```bash
"$OCMIRROR" --v2 \
  --config "$IMAGESET_CONFIG" \
  "file://$OSL_DIR"
```

Expected result: `$OSL_DIR/mirror_000001.tar` and the generated
`$OSL_DIR/working-dir/cluster-resources/` directory.

### 4. Download the RHDH 2.1 plugin-mirroring script

Download the script from the pinned RHDH Operator commit. Verify its SHA-256
sum before you run it.

```bash
export MIRROR_PLUGINS_COMMIT='e5d5373d3182b87d92657e5e1fcf1eaa44dee6bd'

mkdir -p "$SCRIPT_DIR"
curl --fail --location \
  --output "$SCRIPT_DIR/mirror-plugins.sh" \
  "https://raw.githubusercontent.com/redhat-developer/rhdh-operator/${MIRROR_PLUGINS_COMMIT}/.rhdh/scripts/mirror-plugins.sh"
chmod +x "$SCRIPT_DIR/mirror-plugins.sh"

printf '%s  %s\n' \
  '5656ba0fcc8d2bfebc85d77658c767bdf7fd95da81f76d45b3b02ef25f799135' \
  "$SCRIPT_DIR/mirror-plugins.sh" | sha256sum --check -
```

The final command must report `OK`.

### 5. Export the dynamic-plugin OCI artifacts

Dynamic plugins are not covered by the `oc-mirror` image set. Export the RHDH
2.1 plugin catalog and its referenced artifacts separately.

```bash
"$SCRIPT_DIR/mirror-plugins.sh" \
  --plugin-index 'oci://quay.io/rhdh/plugin-catalog-index@sha256:95756855e99641a69be1066ba91beb997d577b2f0a0d27583b9a5c96f769f1da' \
  --to-dir "$WORK/plugins"
```

Expected result: `$WORK/plugins/rhdh-plugin-mirroring-summary.txt`. Keep this
file: it records the source-to-local-registry mapping and is used during
plugin import.

### 6. Obtain the local registry image before disconnecting

The registry image itself must be present locally before the host goes offline.

```bash
podman pull docker.io/library/registry:2
```

### 7. Transfer artifacts, if the target is a separate host

Copy the entire `disconnected-lab-assets` directory, including the archive, charts,
plugin export, and generated `working-dir`. Do not transfer only the tarball:
the generated cluster-resource YAML and plugin export are also required.

## Phase 2: prepare CRC and the local registry

This phase can be completed while connected for the most conservative setup.
If CRC and its base images are already cached, it can also be completed offline.

### 8. Start CRC and log in as a cluster administrator

Run `crc setup` only for a new or deliberately reset CRC installation.
`crc cleanup` removes the cluster state; after using it, reapply every
cluster-side step in this guide.

```bash
crc start

# Obtain the temporary local credentials from CRC, then log in with kubeadmin.
crc console --credentials
oc login -u kubeadmin https://api.crc.testing:6443

oc whoami
oc whoami --show-server
```

### 9. Start the registry on the CRC host

The `host.crc.testing` address resolves to the CRC host from inside the VM.
The registry is deliberately plain HTTP, so the cluster and the dynamic-plugin
installer must be told it is insecure.

```bash
mkdir -p "$HOME/redhat/registry-data"

podman run -d --replace --name rhdh-mirror \
  --network host \
  -v "$HOME/redhat/registry-data:/var/lib/registry:Z" \
  docker.io/library/registry:2

curl -fsS http://127.0.0.1:5000/v2/
```

If the host firewall blocks the CRC VM, allow TCP 5000 in the active firewall
zone before disconnecting. For example, on a host using the
`FedoraWorkstation` zone:

```bash
sudo firewall-cmd --zone=FedoraWorkstation --add-port=5000/tcp --permanent
sudo firewall-cmd --reload
```

Use the correct zone for the host. This port must not be exposed outside the
local test environment.

### 10. Mark the local registry as insecure to OpenShift

For a stock CRC installation, add `host.crc.testing:5000` under
`spec.registrySources.insecureRegistries`. If the list already contains other
registries, preserve them rather than replacing the list.

```bash
oc edit image.config.openshift.io/cluster
```

Ensure the edited resource contains:

```yaml
spec:
  registrySources:
    insecureRegistries:
      - host.crc.testing:5000
```

Wait for the Machine Config Operator to settle, then verify the value:

```bash
oc get image.config.openshift.io/cluster \
  -o jsonpath='{.spec.registrySources.insecureRegistries}{"\n"}'
oc get clusteroperator machine-config
```

## Phase 3: import content and configure the disconnected cluster

At the end of this phase the host registry contains every deployment image and
plugin artifact. It is the preferred point to disconnect the workstation.

### 11. Import the `oc-mirror` archive to the local registry

This operation reads the local archive; it does not need Internet access.

```bash
"$OCMIRROR" --v2 \
  --config "$IMAGESET_CONFIG" \
  --from "file://$OSL_DIR" \
  "docker://$MIRROR" \
  --dest-tls-verify=false
```

### 12. Import the dynamic plugins to the local registry

```bash
"$SCRIPT_DIR/mirror-plugins.sh" \
  --from-dir "$WORK/plugins" \
  --to-registry "$MIRROR"
```

The expected mapping includes both the plugin catalog index and
`backstage-plugin-auth-backend-module-guest-provider`.

```bash
grep 'backstage-plugin-auth-backend-module-guest-provider' \
  "$WORK/plugins/rhdh-plugin-mirroring-summary.txt"
```

### 13. Apply mirror routing and the catalog source

Apply only the ImageDigestMirrorSet and the generated `CatalogSource`:

```bash
export RESOURCES="$OSL_DIR/working-dir/cluster-resources"
export CATALOG='cs-redhat-operator-index-v4-22'

oc apply -f "$RESOURCES/idms-oc-mirror.yaml"
oc apply -f "$RESOURCES/cs-redhat-operator-index-v4-22.yaml"
```

Do **not** apply `cc-redhat-operator-index-v4-22.yaml` on this CRC setup. It
uses the OLM v1 `ClusterCatalog` API, which is not installed here and fails
with `no matches for kind "ClusterCatalog"`.

Wait until the catalog pod is ready:

```bash
oc get catalogsource -n openshift-marketplace "$CATALOG"
oc get pods -n openshift-marketplace -w | grep "$CATALOG"
```

The pod must become `1/1 Running`, not `ImagePullBackOff`.

### 14. Create the RHDH namespace and plugin mirror configuration

```bash
oc create namespace "$RHDH_NS" --dry-run=client -o yaml | oc apply -f -

cat <<EOF | oc apply -n "$RHDH_NS" -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: rhdh-plugin-mirror-conf
data:
  rhdh-registries.conf: |
    [[registry]]
    prefix = "registry.access.redhat.com/rhdh"
    location = "$MIRROR/rhdh"
    insecure = true

    [[registry]]
    prefix = "quay.io/rhdh"
    location = "$MIRROR/rhdh"
    insecure = true
EOF
```

### 15. Permit RHDH to reach the host registry

The RHDH chart's network policies otherwise prevent its
`install-dynamic-plugins` init container from reaching TCP 5000. In the usual
CRC network, the host gateway is `192.168.127.254`. Verify or replace that
address when CRC uses a different network.

```bash
export CRC_HOST_GATEWAY='192.168.127.254'

cat <<EOF | oc apply -n "$RHDH_NS" -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-rhdh-plugin-mirror
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: backstage
      app.kubernetes.io/instance: rhdh
      app.kubernetes.io/name: developer-hub
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: ${CRC_HOST_GATEWAY}/32
      ports:
        - protocol: TCP
          port: 5000
EOF
```

## Disconnect point

You may now disconnect the workstation from all external networks. Leave the
CRC host-only network and loopback networking enabled: the cluster still needs
to reach `host.crc.testing:5000` on the same machine.

For the strongest offline test, start CRC and populate the local registry
before disconnecting, then perform the Helm deployments and all validation
offline. Starting CRC while already disconnected is also valid only when its
bundle and base images are cached locally.

## Phase 4: install the operators and RHDH while disconnected

### 16. Install Serverless and Logic operator subscriptions

The generated catalog source name is not `redhat-operators`; point both
subscriptions to the generated catalog source explicitly.

```bash
helm upgrade --install orchestrator-infra \
  "$WORK/charts/redhat-developer-hub-orchestrator-infra-0.7.3.tgz" \
  --namespace "$RHDH_NS" \
  --values "$INFRA_VALUES" \
  --set serverlessOperator.subscription.spec.source="$CATALOG" \
  --set serverlessLogicOperator.subscription.spec.source="$CATALOG"
```

The values file requests manual InstallPlan approval. Inspect and approve the
two plans in their operator namespaces:

```bash
oc get installplan -n openshift-serverless
oc get installplan -n openshift-serverless-logic

# Replace each placeholder with the pending InstallPlan name shown above.
oc patch installplan <serverless-installplan> \
  -n openshift-serverless \
  --type=merge -p '{"spec":{"approved":true}}'

oc patch installplan <logic-installplan> \
  -n openshift-serverless-logic \
  --type=merge -p '{"spec":{"approved":true}}'
```

Wait for both CSVs and the SonataFlow CRDs:

```bash
oc get csv -n openshift-serverless
oc get csv -n openshift-serverless-logic
oc get crd sonataflows.sonataflow.org sonataflowplatforms.sonataflow.org
```

Both CSVs must show `Succeeded` before RHDH is installed.

### 17. Enable token-capable Guest authentication in the RHDH values

The `auth` configuration alone enables the Guest button, but RHDH 2.1 needs
the Guest backend dynamic plugin to issue an identity token. Without it, the
browser can fall back to a legacy Guest identity and Orchestrator fails with
`AuthenticationError: Illegal token`.

Add this `plugins` entry under the existing top-level `dynamicPlugins:` block
in `$RHDH_VALUES`; preserve the existing `initContainer` configuration beneath
that same block.

```yaml
  plugins:
    - package: oci://quay.io/rhdh/backstage-plugin-auth-backend-module-guest-provider@sha256:550b73577df28add47ee9cdf45c38224c868bb7304587a68467bb98aace40fe4
      enabled: true
      pluginConfig:
        auth:
          providers:
            guest:
              dangerouslyAllowOutsideDevelopment: true
```

The values file must also retain these keys:

```yaml
appConfig:
  signInPage: guest
  auth:
    environment: development
    providers:
      guest:
        dangerouslyAllowOutsideDevelopment: true
```

Replace `apps.example.com` in `openshift.clusterRouterBase` with the CRC route
domain before installing. Obtain it from the cluster:

```bash
oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}{"\n"}'
```

### 18. Install RHDH and its SonataFlowPlatform

```bash
helm upgrade --install rhdh \
  "$WORK/charts/redhat-developer-hub-3.4.13.tgz" \
  --namespace "$RHDH_NS" \
  --values "$RHDH_VALUES"

oc rollout status deployment/rhdh-developer-hub \
  -n "$RHDH_NS" --timeout=15m
```

The RHDH init container must finish successfully. Check it on the current pod,
not on the completed database-creation job.

```bash
POD=$(oc get pods -n "$RHDH_NS" \
  -l app.kubernetes.io/name=developer-hub \
  -o jsonpath='{.items[0].metadata.name}')

oc logs -n "$RHDH_NS" "$POD" -c install-dynamic-plugins
oc get pods -n "$RHDH_NS"
oc get sonataflowplatform -n "$RHDH_NS"
```

Expected healthy workload set:

* `rhdh-developer-hub` is `1/1 Running`.
* `rhdh-postgresql-0` is `1/1 Running`.
* `sonataflow-platform-data-index-service` is `1/1 Running`.
* `sonataflow-platform-jobs-service` is `1/1 Running`.
* `sonataflow-platform` is present as a `SonataFlowPlatform` resource.

## Phase 5: end-to-end validation

### 19. Verify the Guest backend issues tokens

```bash
POD=$(oc get pods -n "$RHDH_NS" \
  -l app.kubernetes.io/name=developer-hub \
  -o jsonpath='{.items[0].metadata.name}')

oc logs -n "$RHDH_NS" "$POD" -c backstage-backend | \
  grep 'auth-backend-module-guest-provider'

oc exec -n "$RHDH_NS" "$POD" -c backstage-backend -- \
  curl -fsS 'http://127.0.0.1:7007/api/auth/guest/refresh?env=development'
```

The refresh endpoint must return identity/token JSON. It must not return
`Unknown auth provider 'guest'`.

### 20. Sign in and test the Orchestrator UI

Get the RHDH route:

```bash
oc get route -n "$RHDH_NS"
```

Open the reported `https://rhdh-developer-hub-rhdh.apps-crc.testing/` route.

1. Sign in as Guest.
2. Open the Orchestrator page.
3. Confirm that the page loads workflow overviews and workflow instances
   without `AuthenticationError` or `Illegal token`.
4. Confirm the browser needs no external network connection.

If this browser previously accepted the legacy Guest fallback, sign out and
hard-refresh. If it still uses the stale identity, run this once in the browser
developer console, then reload and sign in again:

```js
localStorage.removeItem('enableLegacyGuestToken'); location.reload();
```

### 21. Optional: validate a real workflow execution

To execute a workflow, apply a project-owned `SonataFlow` YAML after confirming
that every dependency is available locally:

* The workflow's `spec.podTemplate.container.image` is mirrored or is an OSL
  operator-managed image already included in the released operator content.
* Every referenced ConfigMap, Secret, schema, and OpenAPI specification is
  applied in the namespace.
* Any external API called by the workflow is replaced with a local endpoint or
  is intentionally excluded from the offline test.

Do not apply the example manifests that reference
`quay.io/orchestrator/demo-*` until those images are mirrored. They otherwise
turn a valid offline test into an `ImagePullBackOff`.

For a verified workflow manifest named `example-workflow.yaml`:

```bash
oc apply -n "$RHDH_NS" -f example-workflow.yaml
oc get sonataflow -n "$RHDH_NS"
oc get pods -n "$RHDH_NS" -w
```

Once its workflow pod is ready, refresh the Orchestrator page, start the
workflow, and verify the resulting instance reaches a terminal state in the
UI. That is the full RHDH → Orchestrator backend → Data Index/Jobs Service →
SonataFlow execution path.

## Troubleshooting

| Symptom | Cause | Correct action |
| --- | --- | --- |
| `manifest unknown` for `registry.access.redhat.com/rhdh/plugin-catalog-index:2.1` | That tag is not the RHDH 2.1 plugin catalog reference used here. | Use the pinned `quay.io/rhdh/plugin-catalog-index@sha256:...` reference from this guide. |
| `unauthorized` for `registry-proxy.engineering.redhat.com/rh-osbs/...` | The CR candidate images require engineering-registry authorization. | Use released OSL 1.39.0 GA and the released Red Hat operator catalog. |
| `ClusterCatalog` API not found | CRC does not have OLM v1 `ClusterCatalog`. | Apply `idms-oc-mirror.yaml` and `cs-redhat-operator-index-v4-22.yaml`; skip `cc-...yaml`. |
| CatalogSource pod is `ImagePullBackOff` | The registry is empty, unreachable, or not marked insecure. | Check the host registry, port 5000, `insecureRegistries`, and re-run archive import. |
| RHDH init container cannot fetch plugins | Default RHDH egress policy blocks the host registry. | Verify `rhdh-plugin-mirror-conf` and `allow-rhdh-plugin-mirror`. |
| RHDH reports no authentication providers | The dynamic App Auth plugin needs an explicit sign-in page. | Keep `appConfig.signInPage: guest` and the Guest provider settings. |
| Orchestrator reports `Illegal token` | Browser used the legacy Guest fallback, which has no backend token. | Enable the Guest backend dynamic plugin, sign out, and remove `enableLegacyGuestToken` if necessary. |
| `crc cleanup` was run | Cluster-side configuration, mirrors, catalog sources, subscriptions, and Helm releases were removed. | Re-run Phases 2 through 5. The host registry storage may remain, but verify it before relying on it. |

For a focused log capture:

```bash
POD=$(oc get pods -n "$RHDH_NS" \
  -l app.kubernetes.io/name=developer-hub \
  -o jsonpath='{.items[0].metadata.name}')

oc logs -n "$RHDH_NS" "$POD" -c backstage-backend --since=15m | \
  grep -Ei -C 4 'orchestrator|auth|token|error'

oc get events -n "$RHDH_NS" --sort-by=.lastTimestamp
```

## Final acceptance checklist

- [ ] The host has no Internet connectivity, while CRC host-only networking
  remains available.
- [ ] The local registry responds at `host.crc.testing:5000` from the cluster.
- [ ] The catalog-source pod, Serverless CSV, and Logic CSV are healthy.
- [ ] RHDH, PostgreSQL, Data Index, and Jobs Service are all ready.
- [ ] The RHDH dynamic-plugin init container completed successfully.
- [ ] `/api/auth/guest/refresh` returns a token-bearing Guest identity.
- [ ] The Orchestrator page loads without an authentication error.
- [ ] If executing a workflow, all workflow-specific images and resources are
  local and the instance completes successfully.
