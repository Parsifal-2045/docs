# Phase 2 Tracker Muons reconstruction
Tracker muons are reconstructed (either Inside-Out or Outside-In), in the `HLTMuonsSequence`, included in all HLT Muon trigger paths. In some cases, like `IsoMu_24`, another tracking sequence called `HLTPhase2L3MuonGeneralTracksSequence`is required to reconstruct `hltGeneralTracks` around the muon candidates and compute isolation variables. This section will explore both sequences and changes meant for Phase 2 tracking.

## Reconstruction and isolation sequences 

A good starting point to explore all these sequences is the [HLT configuration explorer for the `HLT_IsoMu24_FromL1TkMuon` path](https://cmshltupgrade.docs.cern.ch/Phase2Menu/Phase2Menu_Legacy/#HLT_IsoMu24_FromL1TkMuon)

- [`HLTMuonsSequence`](https://github.com/cms-sw/cmssw/blob/54a31a83f8a2a0e49b68a29169135c2630e3b1fc/HLTrigger/Configuration/python/HLT_75e33/sequences/HLTMuonsSequence_cfi.py#L10)
```python
HLTMuonsSequence = cms.Sequence(
    HLTL2MuonsFromL1TkSequence
    + HLTPhase2L3OISequence
    + HLTPhase2L3FromL1TkSequence
    + HLTIter0Phase2L3FromL1TkSequence
    + HLTIter2Phase2L3FromL1TkSequence
    + HLTPhase2L3MuonsSequence
)
```
The relevant sequences for Tracker Muons reconstruction are: `HLTPhase2L3OISequence`, `HLTPhase2L3FromL1TkSequence`, `HLTIter0Phase2L3FromL1TkSequence`, and `HLTIter2Phase2L3FromL1TkSequence` let's go through them one by one.

- [`HLTPhase2L3OISequence`](https://github.com/cms-sw/cmssw/blob/54a31a83f8a2a0e49b68a29169135c2630e3b1fc/HLTrigger/Configuration/python/HLT_75e33/sequences/HLTPhase2L3OISequence_cfi.py#L9)

```python
HLTPhase2L3OISequence = cms.Sequence(
    hltPhase2L3OISeedsFromL2Muons
    + hltPhase2L3OITrackCandidates
    + hltPhase2L3OIMuCtfWithMaterialTracks
    + hltPhase2L3OIMuonTrackCutClassifier
    + hltPhase2L3OIMuonTrackSelectionHighPurity
)
```

- [`HLTPhase2L3FromL1TkSequence`]()

- [`HLTIter0Phase2L3FromL1TkSequence`]()

- [`HLTIter2Phase2L3FromL1TkSequence`]()

This concludes the "standard" tracking sequences run in every muon path at the HLT. Additionally, when isolation is required, another complete general tracking sequence is included

- [`HLTPhase2L3MuonGeneralTracksSequence`](https://github.com/cms-sw/cmssw/blob/54a31a83f8a2a0e49b68a29169135c2630e3b1fc/HLTrigger/Configuration/python/HLT_75e33/sequences/HLTPhase2L3MuonGeneralTracksSequence_cfi.py#L27) 

## Pixel tracking
ADD INFO ON PIXELTRACKS (single iter / alpaka / legacy)

## General tracking

### Seeding
The configurations for muon isolation (`iso`) and `iter0` show no major differences.
Even the parameter that differs in name, 
`SeedCreatorPset`, is actually the same since the underlying Pset is identical.
The tested changes for Phase 2 are limited to:

- `useProtoTracksKinematics` set from False to True (which **should** mean that the previously mentioned Pset will not be used to fit 3 or 4 hits and create a seed for general tracking). 
- **INCLUDING VERTICES MAKES A HUGE DIFFERENCE**: remove the `inputVertexCollection` from the configuration (at least until I figure out what's going on with the full muon vertices collection and the trimmed one)

Changes to [`SeedGeneratorFromProtoTracksEDProducer`](https://github.com/cms-sw/cmssw/blob/c83ecb127f70592a95b0544c2e7e2404c9d7e440/HLTrigger/Configuration/python/HLT_75e33/modules/hltIter0Phase2L3FromL1TkMuonPixelSeedsFromPixelTracks_cfi.py#L4)

|        Parameter        |                   Baseline                  |           Proposed change          |
|:-----------------------:|:-------------------------------------------:|:----------------------------------:|
|     InputCollection     |      hltPhase2L3FromL1TkMuonPixelTracks     | hltL3MuonTracksSelectionFromL1TkMu |
|  InputVertexCollection  | hltPhase2L3FromL1TkMuonTrimmedPixelVertices |                 ""                 |
|     SeedCreatorPset     |         hltPhase2SeedFromProtoTracks        |               Removed              |
| useProtoTrackKinematics |                    False                    |                True                |

Changing the parameters of the two modules that follow ([`ckfTrackCandidateMaker`](https://github.com/cms-sw/cmssw/blob/c83ecb127f70592a95b0544c2e7e2404c9d7e440/HLTrigger/Configuration/python/HLT_75e33/modules/hltIter0Phase2L3FromL1TkMuonCkfTrackCandidates_cfi.py#L3) and [`TrackProducer`](https://github.com/cms-sw/cmssw/blob/c83ecb127f70592a95b0544c2e7e2404c9d7e440/HLTrigger/Configuration/python/HLT_75e33/modules/hltIter0Phase2L3FromL1TkMuonCtfWithMaterialTracks_cfi.py#L4)) does not seem to produce vastly different results on the sample used for development (ZMM 200PU), some details on the tests run using parameters from Muon isolation are left as additional information.

### Additional information on ckf 

- Module `CkfTrackCandidateMaker` config differences

|          Parameter          |                             Iter0                             |                     Iso                     |
|:---------------------------:|:-------------------------------------------------------------:|:-------------------------------------------:|
|     RedundantSeedCleaner    |                              none                             |       CachingSeedCleanerBySharedInput       |
|    TrajectoryBuilderPset    | HLTIter0Phase2L3FromL1TkMuonPSetGroupedCkfTrajectoryBuilderIT | hltPhase2L3MuonInitialStepTrajectoryBuilder |
|      TrajectoryCleaner      |              hltESPTrajectoryCleanerBySharedHits              |        TrajectoryCleanerBySharedHits        |
|     propagatorAlongTISE     |               PropagatorWithMaterialParabolicMf               |            PropagatorWithMaterial           |
|    propagatorOppositeTISE   |           PropagatorWithMaterialParabolicMfOpposite           |        PropagatorWithMaterialOpposite       |
|  cleanTrajectoryAfterInOut  |                             False                             |                     True                    |
| onlyPixelHitsForSeedCleaner |                               NA                              |                     True                    |
|     reverseTrajectories     |                               NA                              |                    False                    |
|       useHitsSplitting      |                              True                             |                    False                    |

No noticeable changes when moving from `iter0` to `iso` configuration.

- Module `TrackProducer` config differences

|      Parameter      |               Iter0               |             Iso             |
|:-------------------:|:---------------------------------:|:---------------------------:|
|    AlgorithmName    |              hltIter0             |         initialStep         |
| GeometricInnerState |                True               |            False            |
|   NavigationSchool  |                none               |    SimpleNavigationSchool   |
|      Propagator     | hltESPRungeKuttaTrackerPropagator | RungeKuttaTrackerPropagator |

No noticeable changes when moving from `iter0` to `iso` configuration.
