---
name: analyzing-eventntuple-rooutil
description: Analyze Mu2e EventNtuple ROOT files using the RooUtil C++ interface. Use when writing ROOT macros to loop over events, select tracks or calorimeter clusters, apply cut functions, plot histograms, or create reduced ntuples from EventNtuple datasets.
compatibility: Requires mu2einit, muse setup (AnalysisMDC2020 or AnalysisMDC2025), ROOT, and a Mu2e Offline environment. Common_cuts.hh requires compiled macros (root -l macro.C+).
metadata:
  version: "1.0.0"
  last-updated: "2026-03-17"
---

# Analyzing EventNtuple with RooUtil

## Overview

**RooUtil** is a C++ utility that provides an analyzer-friendly interface to the Mu2e EventNtuple ROOT files. It handles branch management and provides higher-level classes (Track, TrackSegment, etc.) that correctly relate parallel branches.

Use this skill when:
- Writing ROOT macros to analyze EventNtuple files
- Selecting specific events, tracks, track segments, or calorimeter clusters
- Applying standard or custom cut functions
- Plotting histograms of reconstructed or MC-truth quantities
- Creating reduced EventNtuples or per-track output ntuples

**For Python-based analysis**, use `pyutils` instead (out of scope for this skill).

Related skills:
- [finding-data-metacat](../finding-data-metacat/SKILL.md): locate and stage EventNtuple files
- [finding-data-sam](../finding-data-sam/SKILL.md): locate files with SAM

### Agent Guardrails (Read First)

- Always `#include "EventNtuple/rooutil/inc/RooUtil.hh"` — this brings in all required headers
- Add `using namespace rooutil;` to avoid verbosity
- The tree name is `EventNtuple/ntuple` (default in `RooUtil` constructor)
- `common_cuts.hh` requires **compiled** macros: use `root -l -b macro.C+` (note the `+`)
- Never assume a `TrackSegment` has both reco and MC info — check with `has_reco_step()` / `has_mc_step()`
- `SelectTracks(cut)` **modifies** the event in-place; use `GetTracks(cut)` if you need the full event again
- Speed optimization: `TurnOffAllBranches()` before enabling only what you need (up to 10× faster)

---

## Setup

```bash
mu2einit
muse setup AnalysisMDC2020   # or AnalysisMDC2025 depending on dataset
```

This provides ROOT, the EventNtuple package (including RooUtil headers), and Offline data products.

Run a macro interpreted:
```bash
root -l -b macro.C
```

Run compiled (required when using `common_cuts.hh`):
```bash
root -l -b macro.C+
```

---

## Loading Files

`RooUtil` accepts a single `.root` file or a plain-text file list (one path per line):

```cpp
#include "EventNtuple/rooutil/inc/RooUtil.hh"
using namespace rooutil;

void LoadRooUtil(std::string filename) {
  RooUtil util(filename);
  std::cout << "Events: " << util.GetNEvents() << std::endl;
}
```

Optional constructor arguments:
```cpp
RooUtil util(filename, /*debug=*/false, /*treename=*/"EventNtuple/ntuple");
```

---

## Event Loop

All branches are accessible through the `Event` class:

```cpp
for (int i_event = 0; i_event < util.GetNEvents(); ++i_event) {
  auto& event = util.GetEvent(i_event);

  // Single-object branch
  event.evtinfo->leafname;

  // Vector branch — loop manually
  for (auto& obj : *(event.trk)) {
    obj.leafname;
  }
}
```

Use `ntuplehelper` to discover branches and leaves:
```bash
ntuplehelper --list-all-branches
ntuplehelper branchname
```

See [references/branches.md](references/branches.md) for the full branch table.

---

## User-Friendly Classes

Rather than looping raw branches, prefer the higher-level classes that correctly pair related branches:

| Class | Accessed via | Contains |
|---|---|---|
| `Track` | `event.GetTracks()` | `trk`, `trkmc`, `trkcalohit`, `trkqual`, `trkpid`, plus vectors |
| `TrackSegment` | `track.GetSegments()` | `trkseg`, `trksegmc`, `trksegpars_{lh,ch,kl}` |
| `TrackHit` | `track.GetHits()` | `trkhit`, `trkhitmc`, `trkhitcalib` |
| `MCParticle` | `track.GetMCParticles()` | `mcsim` |
| `CaloCluster` | `event.GetCaloClusters()` | `calocluster`, `caloclustermc`, `calohits`, `calomcsim` |
| `CrvCoinc` | `event.GetCrvCoincs()` | `crvcoinc`, `crvcoincmc` |

All `Get*()` methods also have `Count*()` variants.

### Track loop example

```cpp
#include "EventNtuple/rooutil/inc/RooUtil.hh"
using namespace rooutil;

void TrackLoop(std::string filename) {
  RooUtil util(filename);
  for (int i = 0; i < util.GetNEvents(); ++i) {
    auto& event = util.GetEvent(i);
    auto tracks = event.GetTracks();
    for (auto& track : tracks) {
      track.trk->leafname;
    }
  }
}
```

### TrackSegment loop example

```cpp
auto tracks = event.GetTracks();
for (auto& track : tracks) {
  auto segments = track.GetSegments();
  for (auto& seg : segments) {
    if (seg.trkseg != nullptr) {
      seg.trkseg->mom.R();   // reconstructed momentum magnitude
    }
  }
}
```

See [PlotEntranceMomentum.C](../../EventNtuple/rooutil/examples/PlotEntranceMomentum.C) for a complete example.

---

## Cut Functions

A cut function has the signature:
```cpp
bool my_cut(ObjectType& obj);
```

Pass it to `Get*()` or `Count*()`:
```cpp
auto e_minus_tracks = event.GetTracks(is_e_minus);
int n = event.CountTracks(is_downstream);
```

### Common cuts

Many cuts are pre-defined in `common_cuts.hh`:

```cpp
#include "EventNtuple/rooutil/inc/common_cuts.hh"
```

List all available cuts:
```bash
rooutilhelper --list-available-cuts
```

Key common cuts:

| Cut | Applies to | Effect |
|---|---|---|
| `is_e_minus` | `Track` | e-minus fit hypothesis |
| `is_e_plus` | `Track` | e-plus fit hypothesis |
| `is_mu_minus` | `Track` | mu-minus fit hypothesis |
| `is_downstream` | `Track` / `TrackSegment` | going in +z direction |
| `is_upstream` | `Track` / `TrackSegment` | going in -z direction |
| `tracker_entrance` | `TrackSegment` | at tracker entrance surface |
| `tracker_middle` | `TrackSegment` | at tracker middle surface |
| `tracker_exit` | `TrackSegment` | at tracker exit surface |
| `has_reco_step` | `TrackSegment` | has a reconstructed surface step |
| `has_mc_step` | `TrackSegment` | has an MC-truth surface step |
| `passes_trkqual(track, cut_val)` | `Track` | trkqual MVA > cut_val |
| `passes_trigger(event, "name")` | `Event` | named trigger fired |

### Combining cuts

Option 1 — named function:
```cpp
bool my_cut(const Track& track) {
  return is_e_minus(track) && is_downstream(track);
}
auto tracks = event.GetTracks(my_cut);
```

Option 2 — lambda:
```cpp
auto tracks = event.GetTracks([](const Track& track){
  return is_e_minus(track) && is_downstream(track);
});
```

**Full example** — momentum at tracker entrance for e-minus tracks:
```cpp
#include "EventNtuple/rooutil/inc/RooUtil.hh"
#include "EventNtuple/rooutil/inc/common_cuts.hh"
#include "TH1F.h"
using namespace rooutil;

void PlotMomentum(std::string filename) {
  TH1F* h = new TH1F("hMom", "Entrance Momentum; p (MeV/c); events", 50, 95, 110);
  RooUtil util(filename);
  for (int i = 0; i < util.GetNEvents(); ++i) {
    auto& event = util.GetEvent(i);
    for (auto& track : event.GetTracks(is_e_minus)) {
      auto segs = track.GetSegments([](TrackSegment& s){
        return tracker_entrance(s) && has_reco_step(s);
      });
      for (auto& seg : segs) h->Fill(seg.trkseg->mom.R());
    }
  }
  h->Draw("HIST E");
}
```

---

## Creating Output Ntuples

### Reduced EventNtuple (same structure, fewer events/tracks)

```cpp
RooUtil util(filename);
util.CreateOutputEventNtuple("output.root");

for (int i = 0; i < util.GetNEvents(); ++i) {
  auto& event = util.GetEvent(i);
  event.SelectTracks(is_e_minus);   // removes non-e-minus tracks from event
  util.FillOutputEventNtuple();
}
```

The output file is a valid EventNtuple readable by RooUtil again.

See [CreateNtuple.C](../../EventNtuple/rooutil/examples/CreateNtuple.C).

### Per-track ntuple (different structure)

See [CreateTrackNtuple.C](../../EventNtuple/rooutil/examples/CreateTrackNtuple.C).

---

## Speed Optimizations

By default all branches are read. For large datasets, enable only what you need:

```cpp
RooUtil util(filename);
util.TurnOffAllBranches();
util.TurnOnBranches({"trk", "trksegs", "trksegsmc"});
```

This can provide up to a **10× speedup**. Run the timing test:
```bash
root -l -b EventNtuple/validation/plot_rooutil_timing.C
```

---

## Debugging

```cpp
RooUtil util(filename);
util.SetDebug(true);   // verbose branch-loading messages
```

Check an EventNtuple file for version number and trigger branches:
```bash
checkEventNtuple file.root
```

Browse branches interactively:
```bash
ntuplehelper --list-all-branches
ntuplehelper branchname leafname
```

---

## References

- [RooUtil README](../../EventNtuple/rooutil/README.md) — quick reference for all classes and methods
- [Tutorial: Analyzing with RooUtil](../../EventNtuple/tutorial/eventntuple-rooutil.md) — step-by-step tutorial with challenges
- [Example macros](../../EventNtuple/rooutil/examples/) — complete working examples
- [Branch table](references/branches.md) — all EventNtuple branches and their leaf structs
- [common_cuts.hh](../../EventNtuple/rooutil/inc/common_cuts.hh) — all pre-defined cut functions
- [Mu2e Analysis Tools Tutorial](https://mu2ewiki.fnal.gov/wiki/Analysis_Tools_Tutorial)
- [#analysis-tools Slack channel](https://mu2e.slack.com/archives/analysis-tools)
