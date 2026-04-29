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

### Option C: Use oc-compliance plugin (recommended for HTML reports)

```bash
# Install the plugin if not present
# Download from: https://github.com/openshift/oc-compliance/releases
# Place binary in your PATH

# Fetch results as a tar archive
oc compliance fetch-raw scansettingbindings my-scan-binding \
  -o ./my-compliance-results \
  -n openshift-compliance

# The archive contains ARF XML files which can be opened with:
# - OpenSCAP oscap tool
# - SCAP Workbench (GUI)
# - https://github.com/ComplianceAsCode/content

# Generate HTML report from ARF (requires oscap installed locally)
oscap xccdf generate report <arf-result-file.xml> > report.html
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
