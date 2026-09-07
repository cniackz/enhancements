# KEP-6058: CSI global mount reconstruction

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1: node drain and reboot while unstage is in progress](#story-1-node-drain-and-reboot-while-unstage-is-in-progress)
    - [Story 2: the pod-local vol_data.json is lost or corrupt](#story-2-the-pod-local-vol_datajson-is-lost-or-corrupt)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [x] (R) Design details are appropriately documented
- [x] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [x] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md#summary-of-changes) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [x] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation, e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

A CSI volume staged on a node keeps a global mount under
`/var/lib/kubelet/plugins/kubernetes.io/csi/<driver>/<sha256(volumeHandle)>/globalmount`.
When kubelet restarts it rebuilds its in-memory volume state by walking
`/var/lib/kubelet/pods`, so a global mount is only ever revisited through a pod
directory that references it. A global mount that outlives its pod directory is
therefore invisible to reconstruction: nothing unstages it, and nothing keeps
the volume in `node.status.volumesInUse`. The attach/detach controller reads
that as a node which has released the volume, attaches it elsewhere, and on an
RWO filesystem (FibreChannel, iSCSI, EBS, and others) two nodes write to the
same device.

The common way to get there needs no corruption and no operator mistake.
Kubelet lets a pod be deleted once `TearDown` / `NodeUnpublishVolume` has
succeeded for the pod's volumes; it does not wait for `UnmountDevice` /
`NodeUnstageVolume`. A node drain during a storage network hiccup completes on
that basis, the node reboots, and the global mount survives with its pod
directory already gone ([#121937][]).

This KEP makes reconstruction find those mounts on its own. Alongside the
existing walk of `/var/lib/kubelet/pods`, reconstruction scans the CSI plugin
directory for global mounts, independent of any pod directory. Each one it
finds is registered in the ActualStateOfWorld as *uncertain*, following the
semantics KEP-3756 introduced, and the volume manager resolves it: a volume a
pod still needs is re-verified as usual, and a volume no pod needs is unstaged
through `NodeUnstageVolume` before its directory is removed. Registering the
mount before touching it is what keeps the volume in `volumesInUse` for the
whole cleanup, so no competing attach can start in the middle.

A second, rarer failure mode reaches the same orphaned global mount from the
other direction: the pod directory is still there, but its `vol_data.json` is
missing or corrupt, so `ConstructVolumeSpec` cannot rebuild the volume and
`cleanupMounts` cannot release it either, because building an unmounter reloads
the same unreadable file ([#101791][]). KEP-3756 documented this case and
prescribed manual operator cleanup. This KEP closes it too, by making the
global mount's own `vol_data.json` self-sufficient: `MountDevice` already
writes `volumeHandle` and `driverName` there, and this KEP adds `specVolID` and
`volumeLifecycleMode`, so reconstruction and unmounting can both fall back to
that file when the pod-local copy cannot be loaded.

Both recovery paths sit behind one new alpha feature gate,
`CSIGlobalMountReconstruction`, default off.

## Motivation

Kubelet does not wait for the device-level unstage before it lets a pod go. A
pod is deleted once `TearDown` / `NodeUnpublishVolume` has succeeded for its
volumes, while `UnmountDevice` / `NodeUnstageVolume` is still in flight. A node
drain waits for pods, so the drain reports success and the node reboots with
the unstage unfinished. What is left on disk is a global mount directory and
its `vol_data.json`, with no pod directory anywhere that names the volume.

That is the shape of [#121937][], and it is the case reconstruction cannot see.
Reconstruction is driven by `/var/lib/kubelet/pods`: for each pod-local mount
it finds, it asks the volume plugin to rebuild a `volume.Spec` and records the
volume in the ActualStateOfWorld. A global mount with no pod directory
generates no candidate, so `ConstructVolumeSpec` is never called for it, the
volume never enters the ActualStateOfWorld, and it never appears in
`node.status.volumesInUse`. Nothing on the node unstages the mount, and nothing
tells the driver the volume is gone, while the attach/detach controller is free
to attach it to another node. No recovery keyed off the pod directory can reach
this, because the pod directory is exactly what is missing. Recovery has to
start from the global mount itself.

Issue [#101791][] has tracked the same symptom, a live global mount with no
in-memory record, since 2021, and reaches it by a second route. The pod-local
`vol_data.json` is the only source of truth reconstruction uses, so any
condition that loses or corrupts it (disk full at write time, a partial write
during shutdown, an operator deleting the directory while chasing a different
problem, filesystem corruption) produces the same orphaned mount with the pod
directory still in place. KEP-3756 made reconstruction robust against most
kubelet bugs but kept that contract: an unreadable pod-local file means
reconstruction fails and an operator must clean up by hand, as documented in
its troubleshooting section. In practice operators rarely catch that window
before the controller detaches and re-attaches.

Both routes are recoverable from data already on disk. The global mount's own
`vol_data.json`, written by `csiAttacher.MountDevice`, already stores
`volumeHandle` and `driverName`; two more fields make it self-sufficient. With
that file trusted as a source, reconstruction can start from the global mount
whether or not a pod directory exists, and a manual recovery path documented in
KEP-3756 becomes an automatic one, with no change to any contract with CSI
drivers.

[#101791]: https://github.com/kubernetes/kubernetes/issues/101791
[#121937]: https://github.com/kubernetes/kubernetes/issues/121937

### Goals

- Recover CSI global mounts whose pod directory no longer exists, the case a
  kubelet restart or a node reboot during `NodeUnstageVolume` leaves behind, by
  scanning the CSI plugin directory during reconstruction, registering what it
  finds as uncertain in the ActualStateOfWorld, and letting the volume manager
  unstage it cleanly before the directory is removed.
- Recover the same orphaned global mount when the pod directory is still
  present but its `vol_data.json` is missing or corrupt, by falling back to the
  global mount's own `vol_data.json`, for reconstruction and for unmounting.
- Keep `node.status.volumesInUse` accurate for the whole of that cleanup, so
  the attach/detach controller cannot start a competing attach against a volume
  that is still staged on the node.
- Keep the change additive: no behavior change on the path where reconstruction
  already succeeds.
- Gate the new behavior behind an alpha feature gate, default off.

### Non-Goals

- In-tree FibreChannel and iSCSI. They derive their state from `/proc/mounts`
  rather than from a state file, so this failure mode takes a different shape
  there. The same recovery could be built for them, but it is out of scope
  here, and both are migrating to CSI.
- The pod-local fallback for raw block volumes. Block volumes keep no pod-local
  `vol_data.json` to lose: `NewBlockVolumeMapper` writes theirs to
  `plugins/kubernetes.io/csi/volumeDevices/<specVolID>/data`, already
  node-global, and both `ConstructBlockVolumeSpec` and `NewBlockVolumeUnmapper`
  read it from there. Their pod-local artifact is a symlink rather than a bind
  mount, so there is no mount reference to follow back either. The staging path
  a block volume can still leak is the plugin directory scan's territory, and
  it is covered with the scan rather than separately; note that the scan has to
  treat `volumeDevices` as its own subtree, since it is a sibling of the
  per-driver directories and its volume data sits one level deeper.
- Changing the CSI specification or any contract with drivers. Nothing here
  requires a driver change.
- Recovering a volume whose global `vol_data.json` is also unreadable. With
  both copies gone there is nothing on disk that still names the driver and the
  volume handle, and operator intervention remains necessary.

## Proposal

`csiAttacher.MountDevice` writes `/var/lib/kubelet/plugins/kubernetes.io/csi/<driver>/<sha256(volumeHandle)>/vol_data.json`
with `volumeHandle` and `driverName`. We extend it to also write `specVolID`
(the PV name, equal to the basename of the pod-local mount path) and
`volumeLifecycleMode` (hardcoded to `Persistent` since `MountDevice` only
runs for device-mountable volumes). With those fields the global file names the volume it belongs to, which is
what lets reconstruction start from it with no pod directory in hand. Writing
them changes no behavior on its own.

Reconstruction today enumerates candidates only from `/var/lib/kubelet/pods`,
so a global mount whose pod directory is gone is never visited. The
reconstruction pass gains a second, independent source of candidates: a scan
of
`/var/lib/kubelet/plugins/kubernetes.io/csi/*/*`, the parent directories of
every CSI global mount on the node. For each directory found that is not
already tracked in the ActualStateOfWorld:

- If it contains a readable `vol_data.json`, a `volume.Spec` is rebuilt from
  that file and the global mount is added to the ActualStateOfWorld marked
  as *uncertain*, following the reconstruction semantics introduced by
  KEP-3756. The volume manager then resolves the uncertainty: if a pod in
  the desired state still uses the volume, the mount is re-verified as
  usual; if no pod does, the reconciler calls `UnmountDevice` (for CSI,
  `NodeUnstageVolume`) and removes the directory only after the unstage
  succeeds.
- If the directory is empty, there is no `vol_data.json` to rebuild a spec
  from and nothing can be unstaged; it is removed directly.
- If the directory is not empty but has no readable `vol_data.json`,
  reconstruction of that mount fails with a clear error and operator
  intervention is required, same as today.

Registering the mount in the ActualStateOfWorld before any cleanup keeps
the volume in `node.status.volumesInUse` until `NodeUnstageVolume`
completes, so the attach/detach controller cannot start a competing attach
in the middle of the cleanup. The scan is gated by the same
`CSIGlobalMountReconstruction` feature gate; with the gate off,
reconstruction scans only `/var/lib/kubelet/pods` as today.

The scan starts from the plugin directory, so it cannot help the case where the
pod directory is still present and it is the pod-local `vol_data.json` that is
unreadable. For that, `ConstructVolumeSpec` is changed to fall back to the
global file when the pod-local load fails. It asks the mount table which global mount the pod-local
mount is bound to: `NodePublishVolume` publishes the staged volume into the pod
directory, which drivers implement as a bind mount from the staging path, so
the global mount is a mount reference of `<mountPath>/mount` and the data
directory is its parent. If no global mount is bind mounted there, or its
`vol_data.json` names no driver and no volume handle, reconstruction fails as
today.

The fallback is wrapped by `utilfeature.DefaultFeatureGate.Enabled(features.CSIGlobalMountReconstruction)`.
With the gate off, `ConstructVolumeSpec` behaves exactly as before. With the
gate on, only the failure path is altered: the success path is unchanged.

### User Stories

#### Story 1: node drain and reboot while unstage is in progress

A node is drained while the storage network is having a hiccup. Every pod
finishes `NodeUnpublishVolume`, so kubelet lets the pods go and the drain
reports success, but `NodeUnstageVolume` never completes before the node
reboots. The global mount directory and its `vol_data.json` are still on disk
after the restart, with no pod directory that references the volume.
Reconstruction, driven by `/var/lib/kubelet/pods`, never sees it ([#121937][]).
The volume is absent from `node.status.volumesInUse`, the attach/detach
controller attaches it to another node, and two nodes write to the same RWO
filesystem.

With the gate enabled, the plugin directory scan finds the global mount and
registers it as uncertain before anything is cleaned up, so the volume stays in
`volumesInUse`. No pod claims it, so the volume manager calls
`NodeUnstageVolume`, the driver gets its unstage, the directory is removed, and
only then does the volume leave `volumesInUse` and detach cleanly. No operator
involvement.

#### Story 2: the pod-local vol_data.json is lost or corrupt

A node runs short on memory and the kernel kills kubelet mid-write to a pod's
`vol_data.json`, or an operator investigating a stuck pod deletes the contents
of the pod's volume directory. Either way the pod directory survives with an
unreadable `vol_data.json`. On the next kubelet start reconstruction errors
out, and `cleanupMounts` cannot release the volume either, because building an
unmounter reloads the same file. The global mount stays live and the controller
eventually re-attaches the volume elsewhere: the same orphaned global mount as
Story 1, reached without any pod directory going missing.

With the gate enabled, reconstruction reads the global mount's own
`vol_data.json` and rebuilds the spec from it, and unmount generation reads the
same file, so the volume goes through the normal unmount path on both layers.

### Notes/Constraints/Caveats

- The fallback relies on the pod-local bind mount still being present, which
  is the case it exists for: reconstruction runs before `cleanupMounts`, and a
  global mount leaks precisely because nothing unmounted it.
- Reading the mount table costs one parse of `/proc/self/mountinfo` per failed
  pod-local load, through `GetReliableMountRefs`, which iSCSI and FibreChannel
  reconstruction already use for the same reason. Reconstruction reads it once
  per volume at startup; unmount generation reads it once per reconciler pass
  for as long as a volume stays stuck without its pod-local file.
- The orphaned global mount scan runs once per reconstruction pass (that
  is, once per kubelet startup), over the same bounded set of directories.

### Risks and Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Wrong global mount is matched | Low | The candidate comes from the mount table, so it shares this pod-local mount's device. A driver that stages several volumes from one export can offer more than one, so the recovered `specVolID` must name the volume being reconstructed; a mismatch fails reconstruction rather than guessing. A pre-feature global file carries no `specVolID` and is accepted on the mount reference alone |
| Global vol_data.json is also corrupt | Medium: same root cause may have hit both files | Fallback returns an error and reconstruction fails with the original message plus a wrapped fallback message; behavior matches the no-fallback case |
| Feature gate disabled mid-cluster (skew) | Low | Field additions to global vol_data.json are written unconditionally (gate guards the read fallback only), so a node with the gate disabled still produces files a future enabled node can use |
| Stale global mount data after volume detach | Low | Detach unmounts and removes the global mount directory along with `vol_data.json`; if detach failed previously this KEP is exactly what is supposed to recover from it |
| Orphaned global mount is unstaged while a pod still needs it | Low | The scan registers the mount as uncertain instead of unmounting it directly; the reconciler re-verifies mounts that are still in the desired state and only calls `NodeUnstageVolume` for volumes no pod references |
| Empty-directory removal races with an in-flight `MountDevice` | Very low | The scan runs during reconstruction at kubelet startup, before the reconciler issues new `MountDevice` calls; `NodeStageVolume` is idempotent and `MountDevice` recreates the directories it needs |

## Design Details

The change touches the volume manager reconstruction pass, plus three CSI
files in `pkg/volume/csi` (`csi_plugin.go` in two places):

1. `csi_attacher.go`: `MountDevice` adds `specVolID` (from `spec.Name()`) and
   `volumeLifecycleMode` (constant `string(storage.VolumeLifecyclePersistent)`)
   to the data map written to the global `vol_data.json`. Field additions are
   unconditional: written regardless of the feature gate, so a downgrade does
   not produce stale or partial files.

2. `pkg/kubelet/volumemanager` (reconstruction): after the existing scan of
   `/var/lib/kubelet/pods`, a new step scans
   `/var/lib/kubelet/plugins/kubernetes.io/csi/*/*` and skips every
   directory whose volume is already tracked in the ActualStateOfWorld. For
   each remaining directory: an empty one is removed directly; one with a
   readable `vol_data.json` is turned into a `volume.Spec` (from the same
   four fields the fallback uses) and added to the ActualStateOfWorld with
   its device mount state marked uncertain, so the reconciler either
   re-verifies it (volume still in the desired state) or calls
   `UnmountDevice` / `NodeUnstageVolume` and removes the directory after a
   successful unstage; one that is neither empty nor readable is logged as
   an error and left for operator intervention, as today.

   The pattern needs one exclusion. `plugins/kubernetes.io/csi` holds one
   subdirectory per driver, whose children are the per-volume data directories
   this scan wants, but it also holds `volumeDevices`, the raw block subtree,
   whose children are `staging`, `publish`, and one directory per block volume.
   Those match `*/*` just as well, are never empty, and keep their volume data
   one level deeper under `data/`, so a plain glob would report every block
   volume on the node as a directory with no readable volume data and ask an
   operator to look at it. The scan skips that entry, and covering block volumes
   properly means walking that subtree on its own terms, which is why it is
   listed under Beta rather than claimed here.

   An implementation of this scan already exists as
   [#136771](https://github.com/kubernetes/kubernetes/pull/136771), opened by
   @shivamwayal37 in February 2026 against issue [#121937][] and reviewed over
   four rounds. It scans the same directories and marks what it finds uncertain
   in the ActualStateOfWorld. What that PR does not carry is a feature gate.
   Note also that it reaches the ActualStateOfWorld through `reconstructVolume`,
   which calls `ConstructVolumeSpec`, so it reads the `specVolID` that item 1
   adds: without that field the reconstructed volume has an empty name. Whether this path lands there as the
   [#121937][] bug fix or here behind the gate is for SIG Storage to decide; this
   KEP tracks it either way rather than proposing a duplicate.

3. `csi_util.go`: new helper `findGlobalMountDataFromPodMount(host, mountPath)`
   that asks the mount table which global mount the pod-local mount belongs to.
   `NodePublishVolume` publishes the staged volume into the pod directory, which
   drivers implement as a bind mount from the staging path, so
   `GetReliableMountRefs` on `<mountPath>/mount` returns it, and the data
   directory is its parent. A driver that publishes some other way leaves no
   reference, and the fallback declines rather than guessing. A reference is accepted when it is named
   `globalmount` and the `vol_data.json` beside it names both a driver and a
   volume handle. References that fail either check are logged at V(4) and
   skipped; "no global mount is bind mounted here" is propagated up.

   The mount table is what makes this unambiguous. Matching on the directory
   name instead is not sound: `Spec.Name()` is the PV name for a persistent
   volume but the pod-spec entry for an inline ephemeral volume, and those two
   namespaces overlap, so a name like `data` can denote both. An inline volume
   also never stages a global mount of its own, because `CanDeviceMount` is
   false for ephemeral, so every name match it could produce would belong to
   somebody else's volume. Reading it from the mount table also works for
   global mounts staged by a kubelet that predates this KEP, since it needs no
   field that was not already written.

4. `csi_plugin.go`: `ConstructVolumeSpec` calls `loadVolumeData` as today.
   On error, if `CSIGlobalMountReconstruction` is enabled, it calls the helper
   and on success continues with the parsed map. On failure of both loads, it
   returns the original error plus the fallback error in a wrapped message.
   The success path is unchanged. If the recovered data carries no `specVolID`,
   which is the case for a global file staged by a kubelet older than this
   feature, the `volumeName` argument is used instead: it is the name of the
   pod directory, the same value `SetUpAt` would have stored. The case this
   fallback cannot reach, a global mount whose pod-local bind mount is already
   gone, is what item 2 above covers by scanning independently of the pod
   directories.

5. `csi_plugin.go`: `NewUnmounter` gets the same fallback, behind the same
   gate, sharing the same helper. This is not symmetry for its own sake.
   `GenerateUnmountVolumeFunc` builds the unmounter before it runs `TearDown`,
   and `NewUnmounter` reads the same pod-local `vol_data.json` that item 4 had
   to recover from, so without it the unmount operation is never generated at
   all: the pod is never dropped from the ActualStateOfWorld,
   `GetUnmountedVolumes` never returns the volume, `UnmountDevice` is never
   reached, and the global mount stays exactly as orphaned as before. Rescuing
   the spec without rescuing the unmount closes nothing.

   The alternative, writing the recovered data back to the pod-local file so
   that the existing `NewUnmounter` finds it, was rejected. That file has one
   writer today, `SetUpAt`, and one remover, `removeMountDir`; reconstruction
   is a reader, and adding a second writer to a directory that may be
   mid-teardown is a larger change than the one this KEP makes. It would also
   make a transient failure permanent: once written, the pod-local file parses,
   the fallback condition no longer holds, and the recovery never runs again
   for that volume, so a global mount that was matched wrongly stays matched
   wrongly. And `saveVolumeData` truncates in place with no atomic rename, so a
   repair that fails partway leaves a file an operator can no longer inspect to
   see what was lost.

Feature gate registration is in `pkg/features/kube_features.go` with
`Default: false, PreRelease: featuregate.Alpha`.

### Test Plan

[X] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

None.

##### Unit tests

Coverage for the changed packages:

- `k8s.io/kubernetes/pkg/volume/csi`: 78%

Unit tests in `pkg/volume/csi`, already part of the implementation PR
[#138454](https://github.com/kubernetes/kubernetes/pull/138454), cover both
sides of the gate and the ways the lookup can go wrong:

1. Pod-local `vol_data.json` absent, the pod mount still bind mounted from the
   global one: with the gate enabled `ConstructVolumeSpec` returns a
   `volume.Spec` rebuilt from the global file, and with the gate disabled the
   same call returns the original error.
2. An inline ephemeral volume whose short name matches an unrelated staged
   PersistentVolume is not paired with it, because it shares no mount with
   anything under the plugin directory.
3. A pod mount with no mount references is reported as such rather than
   guessed at.
4. A mount reference whose `vol_data.json` is unreadable, or which names no
   driver, is skipped rather than trusted.
5. A global file with no `specVolID`, which is what a kubelet older than this
   feature wrote, is named from the pod directory; one whose `specVolID` names
   a different volume is refused rather than reconstructed.
6. `NewUnmounter` recovers the driver name and volume handle from the global
   mount with the gate on, and fails as before with the gate off.

Both fallbacks were also exercised end to end on a kind cluster running a
kubelet built from this branch, with a driver that stages and bind mounts and
with the pod deleted, confirming that the rescued volume reaches
`UnmountDevice` and that `NodeUnpublishVolume` and `NodeUnstageVolume` are
called. The orphaned global mount scan has no code on that branch, so it is
not covered by that run.

For the orphaned global mount scan, new unit tests in
`pkg/kubelet/volumemanager` will cover:

1. A global mount directory with a readable `vol_data.json` and no pod
   directory is registered in the ActualStateOfWorld as uncertain.
2. A registered orphan whose volume is not in the desired state gets
   `UnmountDevice` / `NodeUnstageVolume` called and its directory removed
   afterwards.
3. An empty global mount directory is removed directly.
4. A non-empty directory without a readable `vol_data.json` is left in
   place and reported.
5. With the feature gate disabled, the scan does not run.

##### Integration tests

None planned for alpha. The relevant code paths are reached only at kubelet
startup, which integration tests do not exercise meaningfully without a real
node.

##### e2e tests

For beta: node e2e tests that cover both recovery paths.

Pod-local fallback:

1. Mount a CSI volume via a pod using a mock CSI driver.
2. Kill kubelet, delete the pod-local `vol_data.json`, restart kubelet.
3. Assert the volume is reconstructed (no `volumesFailedReconstruction`
   entry, normal unmount on pod deletion).

Orphaned global mount:

1. Mount a CSI volume via a pod using a mock CSI driver that delays
   `NodeUnstageVolume`.
2. Delete the pod and stop kubelet after `NodeUnpublishVolume` succeeds but
   before the unstage completes, then restart kubelet.
3. Assert the scan registers the global mount, `NodeUnstageVolume` is
   called, the directory is removed, and the volume leaves
   `node.status.volumesInUse` only after the unstage.

Tests will live in `test/e2e_node/csi_volume_reconstruction_test.go`.

### Graduation Criteria

#### Alpha

- The global mount's `vol_data.json` carries `specVolID`, so a staged volume
  can be identified without a pod directory. Written unconditionally, so a node
  with the gate off still produces files a future enabled node can use.
- `ConstructVolumeSpec` and `NewUnmounter` fall back to that file when the
  pod-local one cannot be loaded, behind `CSIGlobalMountReconstruction`
  (default off), refusing any global mount whose `specVolID` names a different
  volume.
- Unit tests in `pkg/volume/csi` for both gate states, including the refusal
  path.
- KEP merged.

Alpha deliberately stops at the recovery paths that have code and tests today.
The plugin directory scan is the case this KEP is mainly about, and it is not
listed here because its implementation is
[#136771](https://github.com/kubernetes/kubernetes/pull/136771), opened against
[#121937][] by another contributor and reviewed over four rounds. Where it
lands is Design Details item 2, and it is a question for SIG Storage rather
than a promise this KEP can make. What alpha ships is the field that scan
depends on: it reaches the ActualStateOfWorld through `reconstructVolume`,
which calls `ConstructVolumeSpec`, which reads `specVolID`, and without that
field the reconstructed volume has no name.

#### Beta

- The plugin directory scan behind the same gate, wherever SIG Storage routes
  it, with unit tests in `pkg/kubelet/volumemanager`, and covering raw block
  volumes, whose staging path leaks the same way.
- Node e2e test in CI for at least one release, covering both recovery paths.
- Metrics: a label on `reconstruct_volume_operations_total` distinguishing
  `pod-local` from `global-mount`, so operators can see recovery frequency.
- One release cycle at alpha with no open bugs against either recovery path.
- Default gate flipped to on.

#### GA

- Two releases at beta with the gate default-on, no regressions.
- Conformance test if SIG Architecture deems applicable.
- Production usage documented (CSI driver vendors confirm no surprises).

### Upgrade / Downgrade Strategy

- Upgrade with gate disabled: no behavior change. New global
  `vol_data.json` files include extra fields; older code ignores unknown
  fields, so a future downgrade is safe.
- Upgrade with gate enabled: reconstruction uses the fallback when needed.
  No interaction with control plane.
- Downgrade: kubelet stops reading the extra fields. Global files written
  during the upgraded period contain extra keys that are ignored. No data
  migration needed.

### Version Skew Strategy

This is a kubelet-only feature gate. No skew between control plane and
kubelet is possible. Skew between two kubelets is not possible (volumes are
node-local).

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [X] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: `CSIGlobalMountReconstruction`
  - Components depending on the feature gate: `kubelet`

###### Does enabling the feature change any default behavior?

No. With both `vol_data.json` files intact, the success path is unchanged.
The feature only alters failure paths that today produce orphaned global
mounts: a failed pod-local load now falls back to the global file, and a
global mount left behind by an interrupted unstage is now unstaged cleanly
instead of leaking.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Disabling the gate restores the previous behavior on the next kubelet
restart, and there is no on-disk state migration. `MountDevice` writes the two
extra fields unconditionally, so a kubelet with the gate off keeps producing
files that carry them; the disabled code path simply does not read them. They
are not leftover state either: kubelet deletes that file when it cleans up the
global mount, since `UnmountDevice` calls `removeMountDir` after a successful
`NodeUnstageVolume`, which removes the `globalmount` directory, the
`vol_data.json` beside it, and the volume directory itself. The fields live
exactly as long as the mount they describe.

###### What happens if we reenable the feature if it was previously rolled back?

Same as initial enablement: the next failed pod-local load triggers the
fallback. No state recovery needed.

###### Are there any tests for feature enablement/disablement?

The added unit tests exercise both gate-on and gate-off paths.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

The reconstruction fallback runs only at kubelet startup. The `NewUnmounter`
fallback does not: it runs whenever the reconciler generates an unmount for a
volume whose pod-local `vol_data.json` is unreadable, so for such a volume it
parses `/proc/self/mountinfo` once per reconciler pass until the unmount
succeeds. Volumes with an intact pod-local file never reach either path.

A rollout failure would manifest as a spurious successful reconstruction that
uses incorrect data. Mitigation: the candidate comes from the mount table
rather than from a name that could be shared, and its stored `specVolID` must
name the volume being reconstructed, so a global mount belonging to another
volume is refused instead of used. The resulting `volume.Spec` is built from the same fields the
pod-local file would have provided, so any downstream component that
previously trusted the pod-local file can trust the global file.

For the orphaned global mount scan, the failure to watch for is unstaging a
volume a pod still needs. The uncertain registration prevents that: the
reconciler re-verifies mounts that are still in the desired state instead
of unmounting them, and `NodeUnstageVolume` is only called for volumes no
pod references.

###### What specific metrics should inform a rollback?

`reconstruct_volume_operations_errors_total` should not increase post-rollout
versus pre-rollout. If it does, disable the gate.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Will be exercised in the e2e test added at beta.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

At alpha: kubelet logs at V(2) emit `plugin.ConstructVolumeSpec recovered
vol_data from global mount %s` when reconstruction falls back, and `unmounter
recovered vol_data from global mount %s` when unmount generation does,
and a similar V(2) line when the plugins directory scan registers an
orphaned global mount as uncertain and when its directory is removed after
a successful unstage.

The `reconstruct_volume_operations_total` metric already exists in kubelet at
ALPHA stability, today as an unlabelled counter incremented once per pod
volume directory reconstruction attempt, with
`reconstruct_volume_operations_errors_total` counting the failures among them.
As part of this KEP we plan to add a label to it distinguishing `pod-local`
from `global-mount`, which turns the counter into a labelled one, a `NewCounter` to `NewCounterVec` change across its callers; its alpha
stability allows that without a deprecation cycle. The orphaned global mount
scan increments it too, so a mount recovered with no pod directory is counted
rather than invisible.

###### How can someone using this feature know that it is working for their instance?

Look for the V(2) log line above; or, after beta, inspect the metric label.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

The scan runs once per kubelet start, over a directory tree bounded by the CSI
volumes staged on the node, and reads one small file per entry. On the happy
path the pod-local fallback costs one parse of `/proc/self/mountinfo` and one
file read, tens of milliseconds. `GetReliableMountRefs` retries for up to
one minute when the mount table reads inconsistently, so a pathological node
can spend that long per attempt; kubelet startup does not block on it, the
reconciler retries.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- Count of `reconstruct_volume_operations_errors_total` (should not rise
  after enabling).
- Frequency of the V(2) fallback log line (should be near zero in healthy
  clusters; non-zero indicates a real corruption issue worth investigating
  separately).

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

The labelled counter described above distinguishes the two recovery paths at
beta, so an operator can see which one is firing.

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No. CSI drivers must implement `NodeStageVolume` (only such drivers create a
global mount in the first place); this is already standard.

### Scalability

###### Will enabling / using this feature result in any new API calls?

No new call types. The orphaned global mount scan registers what it finds in
the ActualStateOfWorld, so those volumes appear in the node status update
kubelet already sends: `node.status.volumesInUse` carries one extra entry per
recovered mount until `NodeUnstageVolume` completes. No additional request is
issued, the existing periodic update carries a slightly longer list.

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

Yes, on the `Node` object, and that growth is the fix rather than a cost of it.
`node.status.volumesInUse` is kubelet's statement of which attachable volumes
the node still holds, and it is the attach/detach controller's only interlock
against detaching a device that is still staged: `processVolumesInUse` copies
the list into the controller's actual state of world as `MountedByNode`, and its
detach reconciler skips any volume carrying that flag unless a force detach or
the `node.kubernetes.io/out-of-service` taint overrides it. A global mount that
outlived a kubelet restart is still staged, so it belongs in that list by the
field's own definition, and that it is missing today is the defect that lets
the controller attach the volume elsewhere and produce the double mount
described in the Summary. Concretely: no new API objects, and one extra
`UniqueVolumeName` entry per recovered global mount, on the affected node only,
until `NodeUnstageVolume` completes. The count is bounded by the global mounts
that outlived a kubelet restart, which on a healthy node is zero.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

Negligible for reconstruction, which runs once at startup. For a volume stuck
without a pod-local `vol_data.json`, unmount generation adds one mount table
parse per reconciler pass until the volume is released.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

No. Two extra string fields per global `vol_data.json` (a few dozen bytes).
The fallback parses `/proc/self/mountinfo` and reads one file already on
disk.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

Reconstruction itself reads local disk only and completes without the API
server, including both recovery paths. Acting on what the scan found does need
it: the reconciler decides whether a recovered mount is still wanted by
comparing against the desired state of world, which kubelet populates from the
API server. With the API server unavailable that comparison never runs, the
recovered mounts stay *uncertain*, and nothing is unstaged. That is the safe
direction to fail, because the feature never removes a mount it cannot prove is
unwanted.

###### What are other known failure modes?

- Both `vol_data.json` files corrupt: reconstruction fails with a wrapped
  error message naming both files. Operator intervention required, same as
  today.
- Mount table unreadable, or the pod-local mount has no reference to any
  global mount: the fallback returns an error which is wrapped alongside the
  original one, and reconstruction fails as it does today.
- Orphaned global mount directory that is not empty and has no readable
  `vol_data.json`: the scan logs an error and leaves the directory in
  place. Operator intervention required, same as today.

###### What steps should be taken if SLOs are not being met to determine the problem?

If `reconstruct_volume_operations_errors_total` rises after enabling, capture
kubelet logs and inspect for the wrapped fallback error message. If the
issue is the fallback itself (not the underlying corruption), disable the
gate and report the bug.

## Implementation History

- 2026-04-18: Implementation PR opened against kubernetes/kubernetes ([#138454](https://github.com/kubernetes/kubernetes/pull/138454)).
- 2026-05-04: KEP drafted.
- 2026-07-07: Design extended per SIG Storage review: reconstruction also
  scans the CSI plugins directory for global mounts with no pod directory,
  registers them as uncertain in the ActualStateOfWorld, and lets the
  volume manager call `NodeUnstageVolume` before removing the directory
  (empty directories removed directly).
- 2026-09-06: Retargeted to alpha in v1.38 and moved to `implementable`.
- 2026-09-07: Credited [#136771](https://github.com/kubernetes/kubernetes/pull/136771)
  as the existing implementation of the orphaned global mount scan, and corrected
  the Scalability and Troubleshooting answers, which had been written for the
  narrower pod-local design and no longer described the scan.
- 2026-09-07: Replaced the specVolID directory scan with a lookup through the
  mount table, which cannot pair an inline ephemeral volume with an unrelated
  PersistentVolume of the same name, and needs no field an older kubelet did
  not already write.
- 2026-09-07: Gave `NewUnmounter` the same fallback, without which a rescued
  volume could never be unmounted and its global mount stayed orphaned. Required
  the recovered `specVolID` to name the volume being reconstructed, and fell back
  to the `volumeName` argument when an older kubelet never wrote one. Verified on
  a kind cluster running a kubelet built from this branch.
- 2026-09-07: Restructured per SIG Storage review. The scan for global mounts
  with no pod directory is the case this KEP is mainly about, and the pod-local
  `vol_data.json` fallback is the secondary one, so the Summary, Motivation,
  Goals and User Stories now lead with it and the feature gate was renamed to
  match. Alpha was scoped to the two recovery paths that have code and tests
  today, with the scan moved to beta rather than promised here, since its
  implementation is [#136771](https://github.com/kubernetes/kubernetes/pull/136771)
  and where it lands is a question for the SIG. Raw block volumes recorded in
  Non-Goals for the pod-local path and folded into the scan for beta.

## Drawbacks

The plugin directory scan adds a second candidate source to reconstruction,
which is the larger of the two changes: it registers volumes in the
ActualStateOfWorld that no pod asked about, and lets the volume manager unstage
and remove directories on that basis. The pod-local fallback adds a smaller
amount of complexity to the CSI plugin's reconstruction path. The mitigating factor is that the alternative is to keep
shipping a known data-corruption bug behind the documentation in KEP-3756.

## Alternatives

1. Make the pod-local write atomic and durable enough that it can never be
   partial or missing. Useful but does not address operator-deletion or
   filesystem-level corruption, and atomic writes alone do not protect
   against disk-full at write time.

2. Detect the orphaned global mount during cleanup and unmount it
   defensively. Considered in early discussion of [#101791][]. The
   problem is that by the time `cleanupMounts` runs, the volume has already
   been removed from `volumesInUse`; the controller may have started a
   competing attach. The fallback approach prevents the volume from leaving
   `volumesInUse` in the first place. The orphaned global mount scan in
   this KEP avoids that pitfall the same way: the mount is registered in
   the ActualStateOfWorld first, so `volumesInUse` stays accurate until
   `NodeUnstageVolume` completes.

3. Reconstruct purely from `/proc/mounts`. Possible for some plugins but
   loses the spec information CSI needs (driver-specific options, lifecycle
   mode, etc.). Falling back to a structured file we already write keeps the
   spec intact.
