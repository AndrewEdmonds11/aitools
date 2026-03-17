# EventNtuple Branch Reference

> This file is a reference for the `analyzing-eventntuple-rooutil` skill.
> It is auto-generatable from a live EventNtuple file with:
> ```bash
> ntuplehelper --list-all-branches --export-to-md > doc/branches.md
> ```
> Source: [EventNtuple/doc/branches.md](../../../EventNtuple/doc/branches.md)

---

## Event Branches

These branches contain one element per event.

| branch | structure | explanation | leaf struct |
|--------|-----------|-------------|-------------|
| evtinfo | Single object | event-level information | EventInfo.hh |
| evtinfomc | Single object | MC-truth event-level information | EventInfoMC.hh |
| hitcount | Single object | counts of different hit types in an event | HitCount.hh |
| tcnt | Single object | counts track types and track-related quantities *(MARKED FOR REMOVAL)* | TrkCount.hh |

---

## Track Branches

Each element corresponds to a different Kalman fit hypothesis. Typically 4 per event:
- downstream electron, downstream positron, downstream negative muon, downstream positive muon

Reflected tracks have 8 elements (upstream variants added).

| branch | structure | explanation | leaf struct |
|--------|-----------|-------------|-------------|
| trk | Vector | reconstructed track information | TrkInfo.hh |
| trkmc | Vector | MC-truth information about the track | TrkInfoMC.hh |
| trkcalohit | Vector | calorimeter cluster assigned to a track | TrkCaloHitInfo.hh |
| trkcalohitmc | Vector | MC-truth information for calorimeter clusters | CaloClusterInfoMC.hh |
| trkqual | Vector | output of a multi-variate analysis (MVA) | MVAResultInfo.hh |
| trkpid | Vector | output of a multi-variate analysis (MVA) | MVAResultInfo.hh |

---

## Track Segment Branches

4 elements per event (one per fit hypothesis). Each element is a vector over surface intersections (identified by surface id `sid`).

| branch | structure | explanation | leaf struct |
|--------|-----------|-------------|-------------|
| trksegs | Vector-of-vector | track fit results at particular surfaces | TrkSegInfo.hh |
| trksegpars_lh | Vector-of-vector | LoopHelix track parameters (looping tracks) | LoopHelixInfo.hh |
| trksegpars_ch | Vector-of-vector | CentralHelix track parameters (field-on cosmics) | CentralHelixInfo.hh |
| trksegpars_kl | Vector-of-vector | KinematicLine track parameters (field-off cosmics) | KinematicLineInfo.hh |
| trksegsmc | Vector-of-vector | MC-truth SurfaceSteps through passive elements / virtual detectors | SurfaceStepInfo.hh |

---

## Straw Hit Branches

Vectors where each element is a straw hit assigned to a track. Length given by `trk.nhits`.

| branch | structure | explanation | leaf struct |
|--------|-----------|-------------|-------------|
| trkhits | Vector-of-vector | straw hits assigned to a track | TrkStrawHitInfo.hh |
| trkmats | Vector-of-vector | straw materials used in the Kalman fit | TrkStrawMatInfo.hh |
| trkhitsmc | Vector-of-vector | MC-truth straw hits | TrkStrawHitInfoMC.hh |
| trkhitcalibs | Vector-of-vector | calib and alignment info for straw hits | TrkStrawHitCalibInfo.hh |

---

## Track MC Genealogy Branches

4 elements per event (one per fit hypothesis). Each element is a vector of SimParticles in reverse chronological order (last = initial GEANT4 particle, earlier = daughters).

| branch | structure | explanation | leaf struct |
|--------|-----------|-------------|-------------|
| trkmcsim | Vector-of-vector | SimParticles in genealogy | SimInfo.hh |

---

## General MC Branches

| branch | structure | explanation | leaf struct |
|--------|-----------|-------------|-------------|
| mcsteps | Vector | StepPointMC information | MCStepInfo.hh |

---

## Low-Level Reco Branches

| branch | structure | explanation | leaf struct |
|--------|-----------|-------------|-------------|
| timeclusters | Vector | reconstructed time cluster information | TimeClusterInfo.hh |

---

## Calorimeter Branches

Vectors of clusters / hits / digis in the event. Clusters reference their hits via `hits_` index vectors; hits reference their parent cluster via `clusterIdx_`.

| branch | structure | explanation | leaf struct |
|--------|-----------|-------------|-------------|
| caloclusters | Vector | calorimeter clusters with indices of hits | CaloClusterInfo.hh |
| calohits | Vector | calorimeter hits with indices of recodigis and parent cluster | CaloHitInfo.hh |
| calorecodigis | Vector | calorimeter reco digis with index of raw digi and parent hit | CaloRecoDigiInfo.hh |
| calodigis | Vector | calorimeter raw digis | CaloDigiInfo.hh |

---

## Calorimeter MC Branches

`caloclustersmc` is aligned with `caloclusters` (same size and indices). `calomcsim` lists unique SimParticles across all MC clusters.

| branch | structure | explanation | leaf struct |
|--------|-----------|-------------|-------------|
| caloclustersmc | Vector | MC-truth calorimeter cluster info | CaloClusterInfoMC.hh |
| calomcsim | Vector | SimParticles in genealogy | SimInfo.hh |

---

## CRV Branches

| branch | structure | explanation | leaf struct |
|--------|-----------|-------------|-------------|
| crvsummary | Single object | CRV event summary | CrvSummaryReco.hh |
| crvsummarymc | Single object | MC-truth CRV event summary | CrvSummaryMC.hh |
| crvcoincs | Vector | CRV coincidence cluster information | CrvHitInfoReco.hh |
| crvcoincsmc | Vector | MC track most likely causing coincidence | CrvHitInfoMC.hh |
| crvcoincsmcplane | Vector | MC trajectory crossing the CRV-T xz plane | CrvPlaneInfoMC.hh |

---

## Trigger Branches

Trigger branches are prefixed with `trig_`. To list the trigger branches present in a file:
```bash
checkEventNtuple filename.root
```

---

## Deprecated Branches

| branch | structure | explanation |
|--------|-----------|-------------|
| trkmcsci | Vector-of-vector | StepPointMC info *(deprecated, from trkana)* |
| trkmcssi | Vector-of-vector | StepPointMC summary info *(deprecated, from trkana)* |

---

## Branches Not In Any User-Friendly Class

The following branches exist in the `Event` class but are not yet wrapped in a higher-level class. Access them directly via `event.branchname`:

- `trkcalohitmc`
- `crvdigis`
- `crvpulses`, `crvpulsesmc`
- `crvcoincsmcplane`
- `calorecodigis`, `calodigis`
- `trig_*` branches (use `passes_trigger()` helper instead)
- `mcsteps_virtualdetector`

To request additions, contact the developers on the `#analysis-tools` Slack channel.
