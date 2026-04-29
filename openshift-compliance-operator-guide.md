# OpenShift Compliance Operator — Complete Guide
## Installation, Scanning, Debugging, Reporting, and Remediation

---

## Table of Contents
1. [Verify Operator Installation](#1-verify-operator-installation)
2. [Understanding the Required YAMLs](#2-understanding-the-required-yamls)
3. [Apply ScanSettings and ScanSettingBinding](#3-apply-scansettings-and-scansettingbinding)
4. [Debugging Pending or Not-Ready Pods](#4-debugging-pending-or-not-ready-pods)
5. [Monitor the Scan](#5-monitor-the-scan)
6. [Generate a Compliance Report](#6-generate-a-compliance-report)
7. [Remediation](#7-remediation)
8. [Common Errors and Fixes](#8-common-errors-and-fixes)

---

## 1. Verify Operator Installation

Before applying any YAML files, confirm the operator and its CRDs are healthy.

```bash
# Check the operator namespace (usually openshift-compliance)
oc get pods -n openshift-compliance
```

Expected output — you should see these pods Running:
```
NAME                                                 READY   STATUS    RESTARTS
compliance-operator-<hash>                           1/1     Running   0
ocp4-openshift-compliance-pp-<hash>                 1/1     Running   0
rhcos4-openshift-compliance-pp-<hash>               1/1     Running   0
```

If any pod shows `Pending`, `CrashLoopBackOff`, or `Error`, fix those first (see Section 4).

```bash
# Confirm all CRDs are registered
oc get crd | grep compliance
```

You should see CRDs including:
- `compliancescans.compliance.openshift.io`
- `compliancesuites.compliance.openshift.io`
- `scansettings.compliance.openshift.io`
- `scansettingbindings.compliance.openshift.io`
- `complianceremediations.compliance.openshift.io`
- `profilebundles.compliance.openshift.io`

If CRDs are missing, the operator install is incomplete — reinstall via OperatorHub.

---

## 2. Understanding the Required YAMLs

The compliance operator scan requires **two YAML files**:

### File 1: `scansettings.yaml`
Defines **how** the scan runs — schedule, storage, node roles to scan.

```yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSetting
metadata:
  name: my-scan-setting
  namespace: openshift-compliance
rawResultStorage:
  size: "1Gi"
  rotation: 3
roles:
  - worker
  - master
scanTolerations:
  - operator: Exists
schedule: "0 1 * * *"   # cron: daily at 1am. Use "0 0 31 2 *" to not auto-repeat.
```

### File 2: `scansettingbinding.yaml`
Defines **what** to scan — binds a profile (e.g., CIS, NIST) to the ScanSetting.

```yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSettingBinding
metadata:
  name: my-scan-binding
  namespace: openshift-compliance
profiles:
  - apiGroup: compliance.openshift.io/v1alpha1
    kind: Profile
    name: ocp4-cis          # or rhcos4-moderate, ocp4-moderate, etc.
  - apiGroup: compliance.openshift.io/v1alpha1
    kind: Profile
    name: ocp4-cis-node
settingsRef:
  apiGroup: compliance.openshift.io/v1alpha1
  kind: ScanSetting
  name: my-scan-setting     # must match metadata.name in scansettings.yaml
```

### List available profiles
```bash
oc get profiles.compliance -n openshift-compliance
```

Common profiles:
| Profile Name           | Description                              |
|------------------------|------------------------------------------|
| `ocp4-cis`             | CIS Benchmark for OpenShift platform     |
| `ocp4-cis-node`        | CIS Benchmark for OpenShift nodes        |
| `ocp4-moderate`        | NIST 800-53 Moderate for platform        |
| `rhcos4-moderate`      | NIST 800-53 Moderate for RHCOS nodes     |
| `ocp4-pci-dss`         | PCI-DSS for platform                     |

---

## 3. Apply ScanSettings and ScanSettingBinding

```bash
# Apply in this order — settings first, then binding
oc apply -f scansettings.yaml
oc apply -f scansettingbinding.yaml
```

Verify they were created:
```bash
oc get scansettings -n openshift-compliance
oc get scansettingbindings -n openshift-compliance
```

### What happens next (automatically)
After applying the binding, the operator creates a `ComplianceSuite`, which in turn
creates individual `ComplianceScan` objects — one per profile per node role. Each scan
spawns scanner pods on every node.

```bash
# Watch the ComplianceSuite appear
oc get compliancesuites -n openshift-compliance -w

# Watch the ComplianceScans
oc get compliancescans -n openshift-compliance -w

# Watch scanner pods spin up (there will be one per node)
oc get pods -n openshift-compliance -w
```

---

## 4. Debugging Pending or Not-Ready Pods

This is the most common failure point. Work through each step.

---

### Step 4.1 — Identify which pods are stuck

```bash
oc get pods -n openshift-compliance
```

Look for `Pending`, `Init:Error`, `CrashLoopBackOff`, `Error`, or `0/1 Running`.

---

### Step 4.2 — Describe the stuck pod

```bash
oc describe pod <pod-name> -n openshift-compliance
```

Scroll to the **Events** section at the bottom. Common messages and their meaning:

| Event Message                              | Likely Cause                              |
|--------------------------------------------|-------------------------------------------|
| `0/N nodes are available: N Insufficient memory` | Node resource pressure                |
| `did not have node affinity/taints`        | Scanner pod can't tolerate master taint   |
| `persistentvolumeclaim ... pending`        | PVC not bound (storage issue)             |
| `ImagePullBackOff`                         | Can't pull scanner image (network/registry) |
| `Init containers not started`              | Init container is failing                 |

---

### Step 4.3 — Fix: Scanner pods stuck on tainted master nodes

Master nodes are tainted by default. Add tolerations to your `scansettings.yaml`:

```yaml
scanTolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
    operator: Exists
  - operator: Exists   # catch-all for any other taints
```

Re-apply:
```bash
oc apply -f scansettings.yaml
```

Delete the stuck ComplianceSuite to force re-creation:
```bash
oc delete compliancesuite <suite-name> -n openshift-compliance
oc apply -f scansettingbinding.yaml
```

---

### Step 4.4 — Fix: PVC stuck in Pending (storage not provisioning)

```bash
# Check PVCs
oc get pvc -n openshift-compliance

# Describe the stuck PVC
oc describe pvc <pvc-name> -n openshift-compliance
```

If no StorageClass is set or none exists:

```bash
# List available storage classes
oc get storageclass

# Check which one is default (marked with "(default)")
```

If there is no default StorageClass, patch the ScanSetting to specify one:

```yaml
rawResultStorage:
  size: "1Gi"
  rotation: 3
  storageClassName: <your-storage-class-name>
```

If you are on a bare-metal or minimal cluster with no dynamic provisioner, you may need
to create a PV manually:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: compliance-pv-1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/compliance-results
```

```bash
oc apply -f compliance-pv.yaml
```

---

### Step 4.5 — Check pod logs

```bash
# For scanner pods
oc logs <scanner-pod-name> -n openshift-compliance

# For init containers (if pod is in Init state)
oc logs <pod-name> -n openshift-compliance -c <init-container-name>

# For the main operator pod
oc logs deployment/compliance-operator -n openshift-compliance
```

---

### Step 4.6 — Check node resource pressure

```bash
oc describe nodes | grep -A5 "Conditions:"
oc top nodes   # requires metrics-server
```

If nodes show `MemoryPressure` or `DiskPressure`, the scanner pods will not schedule.
Free up resources or add more nodes before re-running.

---

### Step 4.7 — Check ComplianceScan status for operator-level errors

```bash
oc describe compliancescan <scan-name> -n openshift-compliance
```

Look at the `Status.Conditions` field — the operator writes error messages there.

```bash
# Also check events in the namespace
oc get events -n openshift-compliance --sort-by='.lastTimestamp'
```

---

### Step 4.8 — Nuclear reset: delete and re-trigger the suite

If everything looks correct but scans are still stuck:

```bash
# Delete the binding (this cascades to suite and scans)
oc delete scansettingbinding my-scan-binding -n openshift-compliance

# Wait for cleanup
oc get pods -n openshift-compliance -w

# Re-apply both files
oc apply -f scansettings.yaml
oc apply -f scansettingbinding.yaml
```

---

## 5. Monitor the Scan

Scanner pods run, then complete. Each pod mounts a PVC and writes raw ARF/XML results.

```bash
# Watch scan phase transition
# Phases: Pending -> Launching -> Running -> Aggregating -> Done
oc get compliancescans -n openshift-compliance -w

# Check suite phase
oc get compliancesuite -n openshift-compliance

# Count results as they populate
oc get compliancecheckresults -n openshift-compliance | wc -l
```

A finished suite shows `DONE` in the PHASE column. This can take 10–30 minutes depending
on cluster size and profile complexity.

---

## 6. Generate a Compliance Report

### Option A: CLI summary

```bash
# List all check results with PASS/FAIL/MANUAL status
oc get compliancecheckresults -n openshift-compliance

# Filter for failures only
oc get compliancecheckresults -n openshift-compliance \
  -l compliance.openshift.io/check-status=FAIL

# Filter by scan
oc get compliancecheckresults -n openshift-compliance \
  -l compliance.openshift.io/suite=my-scan-binding
```

### Option B: Extract the raw ARF/XCCDF XML report

The operator stores raw results in a PVC. Use a pod to extract them:

```bash
# Find the result PVC
oc get pvc -n openshift-compliance

# Spin up a pod to access the PVC
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: result-extractor
  namespace: openshift-compliance
spec:
  containers:
  - name: extractor
    image: registry.access.redhat.com/ubi8/ubi
    command: ["sleep", "3600"]
    volumeMounts:
    - name: results
      mountPath: /results
  volumes:
  - name: results
    persistentVolumeClaim:
      claimName: <your-result-pvc-name>   # replace this
  restartPolicy: Never
EOF

# Wait for the pod to be Running
oc wait --for=condition=Ready pod/result-extractor -n openshift-compliance --timeout=60s

# List result files
oc exec result-extractor -n openshift-compliance -- ls /results

# Copy results to your local machine
oc cp openshift-compliance/result-extractor:/results ./compliance-results/

# Clean up
oc delete pod result-extractor -n openshift-compliance
```

### Option C: Generate an HTML Report

HTML reports are human-readable, shareable with auditors, and show pass/fail details with
rule descriptions and remediation guidance inline.

#### Step C.1 — Fetch the raw ARF results

Use the `oc-compliance` plugin to pull the result files from the PVCs onto your local machine:

```bash
# Install the plugin if not already present
# Download from: https://github.com/openshift/oc-compliance/releases
# Place the binary in your PATH, e.g.:
#   mv oc-compliance-linux-amd64 /usr/local/bin/oc-compliance
#   chmod +x /usr/local/bin/oc-compliance

# Fetch all raw ARF result files for a ScanSettingBinding
oc compliance fetch-raw scansettingbindings my-scan-binding \
  -o ./my-compliance-results \
  -n openshift-compliance

# Results land at: ./my-compliance-results/<scan-name>/<node>.xml
ls ./my-compliance-results/
```

Each `.xml` file is an ARF (Assessment Results Format) document — the input for `oscap`.

#### Step C.2 — Generate HTML from each ARF file

Requires `openscap` installed locally (`dnf install openscap-scanner` on RHEL/Fedora,
`apt install libopenscap8` on Debian/Ubuntu):

```bash
# Generate an HTML report from a single ARF file
oscap xccdf generate report \
  ./my-compliance-results/<scan-name>/<node>.xml \
  > ./report-<node>.html

# Batch — generate one report per result file
for arf in ./my-compliance-results/**/*.xml; do
  node=$(basename "$arf" .xml)
  oscap xccdf generate report "$arf" > "./report-${node}.html"
  echo "Generated report-${node}.html"
done
```

The HTML file is self-contained (no external assets) and can be emailed or attached to a ticket.

#### Step C.3 — View the report

Open directly in a browser:

```bash
# Linux
xdg-open report-<node>.html

# macOS
open report-<node>.html
```

Or serve over HTTP to share with the team without sending files:

```bash
# Python 3 — serves current directory on port 8080
python3 -m http.server 8080 --directory .
# Open http://localhost:8080/report-<node>.html in a browser
```

#### What the HTML report contains

- **Score** — overall percentage of passing rules
- **Rule results** — each check listed as PASS / FAIL / NOTSELECTED with severity badge
- **Rule descriptions** — what the check validates and why it matters
- **Fix text** — the remediation instructions embedded from the SCAP content
- **Identifiers** — CCE, CVE, and STIG IDs cross-referenced per rule

#### Alternative: SCAP Workbench (GUI)

If you prefer a graphical viewer, SCAP Workbench can open ARF files directly:

```bash
# Install
dnf install scap-workbench   # RHEL/Fedora

# Open an ARF file
scap-workbench ./my-compliance-results/<scan-name>/<node>.xml
```

---

## 7. Remediation

The operator can automatically apply or stage fixes for failed checks.

### Step 7.1 — List available remediations

```bash
oc get complianceremediations -n openshift-compliance

# See which ones are applied vs unapplied
oc get complianceremediations -n openshift-compliance \
  -o custom-columns=NAME:.metadata.name,APPLY:.spec.apply,TYPE:.spec.type
```

### Step 7.2 — Review a specific remediation before applying

```bash
oc describe complianceremediation <remediation-name> -n openshift-compliance
```

The `spec.current` field shows the exact MachineConfig or Kubernetes object that will
be created/modified. **Always review this before applying.**

### Step 7.3 — Apply a single remediation manually

```bash
oc patch complianceremediation <remediation-name> \
  -n openshift-compliance \
  --type merge \
  -p '{"spec":{"apply":true}}'
```

### Step 7.4 — Apply all remediations for a scan (bulk)

```bash
# Apply all remediations for a specific scan
oc get complianceremediations -n openshift-compliance \
  -l compliance.openshift.io/scan-name=<scan-name> \
  -o name | xargs -I{} oc patch {} \
  -n openshift-compliance \
  --type merge \
  -p '{"spec":{"apply":true}}'
```

### Step 7.5 — Auto-remediation via ScanSetting

To have the operator apply remediations automatically after each scan, set `autoApplyRemediations: true` in your ScanSetting:

```yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSetting
metadata:
  name: my-scan-setting
  namespace: openshift-compliance
autoApplyRemediations: true   # <-- add this
rawResultStorage:
  size: "1Gi"
  rotation: 3
roles:
  - worker
  - master
scanTolerations:
  - operator: Exists
schedule: "0 1 * * *"
```

> **Warning:** `autoApplyRemediations: true` will trigger node reboots via MachineConfig.
> Do NOT use this in production without understanding each remediation first.
> Node reboots happen in a rolling fashion via the MachineConfigOperator.

### Step 7.6 — Monitor MachineConfig rollout (after remediation)

Remediations that modify OS-level settings produce MachineConfig objects, which cause
rolling node reboots:

```bash
# Check MachineConfigPool status
oc get mcp

# Watch the rollout
oc get mcp -w

# A node is updating when UPDATING=True and READYMACHINECOUNT decreases
```

Wait for `UPDATED=True` and `DEGRADED=False` before re-running a scan.

### Step 7.7 — Re-run the scan to verify fixes

```bash
# Trigger a manual rescan
oc annotate compliancescan <scan-name> \
  -n openshift-compliance \
  compliance.openshift.io/rescan=
```

Check whether previously-failing checks now pass:

```bash
oc get compliancecheckresults -n openshift-compliance \
  -l compliance.openshift.io/check-status=FAIL
```

---

### Step 7.8 — Understand Check Result Statuses Before Remediating

Every `ComplianceCheckResult` has one of four statuses. Knowing which you have determines your remediation path:

| Status | Meaning | Action |
|---|---|---|
| `PASS` | Check satisfied | None |
| `FAIL` | Check failed — a `ComplianceRemediation` *may* exist | Apply remediation or fix manually (see 7.9–7.11) |
| `MANUAL` | No remediation object will ever be created | Human review required (see 7.12) |
| `INFO` / `NOT-APPLICABLE` | Informational or scoped out for this platform | No action |

```bash
# Count results by status
oc get compliancecheckresults -n openshift-compliance \
  -o jsonpath='{range .items[*]}{.metadata.labels.compliance\.openshift\.io/check-status}{"\n"}{end}' \
  | sort | uniq -c

# List MANUAL checks (require human action — no remediation object will be created)
oc get compliancecheckresults -n openshift-compliance \
  -l compliance.openshift.io/check-status=MANUAL \
  -o custom-columns=NAME:.metadata.name,SEVERITY:.metadata.labels.compliance\.openshift\.io/check-severity

# Find FAILs that have NO remediation object (need a manual fix)
comm -23 \
  <(oc get compliancecheckresults -n openshift-compliance \
      -l compliance.openshift.io/check-status=FAIL \
      -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | sort) \
  <(oc get complianceremediations -n openshift-compliance \
      -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | sort)
```

---

### Step 7.9 — Common FAIL: Kernel / sysctl Parameter Checks

Sysctl failures are among the most frequent CIS/STIG findings. The operator creates a `MachineConfig` when a remediation object exists; if none exists, create one manually.

**Example failing checks:**
- `ocp4-cis-node-worker-sysctl-net-ipv4-conf-all-accept-redirects`
- `ocp4-cis-node-worker-sysctl-net-ipv4-conf-default-send-redirects`
- `ocp4-stig-worker-sysctl-kernel-dmesg-restrict`

```bash
# Step 1 — check if a remediation object exists
oc get complianceremediation -n openshift-compliance | grep sysctl

# Step 2 — apply it (this creates a MachineConfig and triggers a rolling reboot)
oc patch complianceremediation <remediation-name> \
  -n openshift-compliance \
  --type merge -p '{"spec":{"apply":true}}'

oc get mcp -w   # wait for UPDATED=True, DEGRADED=False
```

If no remediation object exists, create the MachineConfig directly:

```yaml
# sysctl-hardening.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 75-sysctl-hardening
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - path: /etc/sysctl.d/75-compliance.conf
          mode: 0644
          contents:
            source: "data:,net.ipv4.conf.all.accept_redirects%3D0%0Anet.ipv4.conf.default.send_redirects%3D0%0Akernel.dmesg_restrict%3D1%0A"
```

```bash
oc apply -f sysctl-hardening.yaml
oc get mcp -w
```

---

### Step 7.10 — Common FAIL: File Permission / Ownership Checks

Many CIS checks fail because kubeconfig, audit log, or kubelet config files have overly permissive modes.

**Example failing checks:**
- `ocp4-cis-node-worker-file-permissions-kubeconfig`
- `ocp4-cis-node-master-file-permissions-etcd-data-dir`
- `ocp4-stig-worker-file-owner-kube-apiserver`

```bash
# Identify the exact path and expected permissions from the check description
oc describe compliancecheckresult <check-name> -n openshift-compliance \
  | grep -A5 "Description"

# Inspect current permissions on the node (investigation only — not persistent)
oc debug node/<node-name> -- chroot /host stat /etc/kubernetes/kubeconfig
```

Fix persistently via MachineConfig (survives reboots and re-provisions):

```yaml
# file-perms-fix.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 75-file-perms-fix
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
        - path: /etc/kubernetes/kubeconfig
          mode: 0600
          user:
            name: root
          group:
            name: root
```

```bash
oc apply -f file-perms-fix.yaml
oc get mcp -w
```

---

### Step 7.11 — Common FAIL: Kubelet Configuration Checks

Kubelet checks require a `KubeletConfig` object; do **not** edit kubelet flags directly or they will be overwritten by MCO.

**Example failing checks:**
- `ocp4-cis-node-worker-kubelet-anonymous-auth`
- `ocp4-cis-node-worker-kubelet-authorization-mode-webhook`
- `ocp4-cis-node-worker-kubelet-protect-kernel-defaults`
- `ocp4-stig-worker-kubelet-enable-streaming-connections-timeout`

```bash
# Check if a remediation object exists and apply it
oc get complianceremediation -n openshift-compliance | grep kubelet

oc patch complianceremediation <remediation-name> \
  -n openshift-compliance \
  --type merge -p '{"spec":{"apply":true}}'

oc get mcp -w
```

If no remediation object exists, create a `KubeletConfig`:

```yaml
# kubelet-hardening.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: compliance-kubelet-hardening
spec:
  machineConfigPoolSelector:
    matchLabels:
      pools.operator.machineconfiguration.openshift.io/worker: ""
  kubeletConfig:
    anonymousAuth: false
    authorization:
      mode: Webhook
    protectKernelDefaults: true
    streamingConnectionIdleTimeout: "5m"
```

```bash
oc apply -f kubelet-hardening.yaml
oc get mcp -w   # triggers a rolling node reboot
```

Verify the settings landed on a node after rollout:

```bash
oc debug node/<node-name> -- chroot /host \
  cat /etc/kubernetes/kubelet.conf \
  | grep -E "anonymousAuth|authorization|protectKernel|streamingConnection"
```

---

### Step 7.12 — MANUAL Checks: Handling Checks With No Remediation

`MANUAL` checks require a human to evaluate and provide evidence. The operator never creates a `ComplianceRemediation` for them.

```bash
# List all MANUAL checks with severity
oc get compliancecheckresults -n openshift-compliance \
  -l compliance.openshift.io/check-status=MANUAL \
  -o custom-columns=\
NAME:.metadata.name,\
SEVERITY:.metadata.labels.compliance\.openshift\.io/check-severity,\
RULE:.metadata.labels.compliance\.openshift\.io/check-rule
```

**Common MANUAL check categories and verification commands:**

| Check pattern | Verification command |
|---|---|
| `*-accounts-*-password*` | `oc debug node/<name> -- chroot /host grep -E "^[^:]+:[^\!*]" /etc/shadow` |
| `*-audit-for-*` | `oc get apiservers cluster -o yaml \| grep -A10 audit` |
| `*-configure-network-policies*` | `oc get netpol -A` |
| `*-rbac-limit-*` | `oc get clusterrolebindings -o wide` |
| `*-etcd-*-encryption*` | `oc get apiserver cluster -o jsonpath='{.spec.encryption}'` |
| `*-pod-security*` | `oc get ns -o custom-columns=NAME:.metadata.name,PSA:.metadata.labels.pod-security\.kubernetes\.io/enforce` |

Once verified, document acceptance in a `TailoredProfile` so future scans reflect your decision:

```yaml
# tailored-profile-manual-accept.yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: TailoredProfile
metadata:
  name: cis-manual-accepted
  namespace: openshift-compliance
spec:
  extends: ocp4-cis
  title: "CIS with accepted manual controls"
  setValues: []
  disabledRules:
    - name: ocp4-cis-configure-network-policies
      rationale: "NetworkPolicies verified manually on 2026-04-29; documented in audit log."
```

```bash
oc apply -f tailored-profile-manual-accept.yaml
```

> **Note:** Disabling a rule suppresses it from scan results. Use only for controls you have genuinely verified or accepted with documented rationale.

---

## 8. Common Errors and Fixes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Scanner pods stuck `Pending` | Master node taints | Add `scanTolerations` to ScanSetting |
| PVC stuck `Pending` | No default StorageClass | Add `storageClassName` or create PV |
| `ImagePullBackOff` | Registry unreachable | Check cluster proxy/firewall settings |
| ComplianceScan stuck in `Launching` | ProfileBundle not ready | Wait for profilebundle pods to finish |
| `Error: no profiles found` | Wrong profile name in binding | Run `oc get profiles.compliance -n openshift-compliance` |
| Remediations created but scan still FAIL | MachineConfig not applied yet | Wait for MCP rollout, then rescan |
| `compliancecheckresults` empty after DONE | Aggregator pod failed | Check aggregator pod logs |
| Suite phase never changes from `Pending` | ScanSettingBinding misconfigured | Check `settingsRef.name` matches ScanSetting name exactly |
| Remediation applied but MCP stuck `DEGRADED` | MachineConfig conflict between remediations | `oc describe mcp worker` to find conflicting MC; `oc get mc` to review duplicate keys |
| Check still FAIL after MCP rollout completes | kubelet not yet restarted on the node | `oc debug node/<name> -- chroot /host systemctl restart kubelet` then rescan |
| No `ComplianceRemediation` exists for a FAIL check | Check is `MANUAL` or operator provides no fix | See Step 7.12 — apply fix manually via MachineConfig or KubeletConfig |

---

### Quick Reference — Key Commands

```bash
# Status overview
oc get pods,pvc,compliancesuites,compliancescans -n openshift-compliance

# Watch everything
oc get all -n openshift-compliance

# Tail operator logs live
oc logs -f deployment/compliance-operator -n openshift-compliance

# Count pass/fail
oc get compliancecheckresults -n openshift-compliance \
  -o jsonpath='{range .items[*]}{.metadata.labels.compliance\.openshift\.io/check-status}{"\n"}{end}' \
  | sort | uniq -c

# Force rescan
oc annotate compliancescan <scan-name> \
  -n openshift-compliance \
  compliance.openshift.io/rescan= --overwrite
```
