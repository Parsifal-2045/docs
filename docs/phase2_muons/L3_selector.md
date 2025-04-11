# Streamlined L3 Tracker Muon reconstruction

## Current reconstruction approach

In the current Phase 2 approach to L3 Tracker Muon reconstruction, all candidates are reconstructed twice: 

- Inside-Out: starting from L1Tk Muons as seeds and reconstructing towards the outer tracker
- Outside-In: starting from seeds produced using Standalone Muons and reconstructing from the outer tracker towards the reconstructed vertex 

This is the full HLT reconstruction sequence, also shown in the [introduction page](/docs/phase2_muons/introduction.md)

![HLT Muon reconstruction diagram](figures/associators_diagram.png)
/// caption
Phase-2 HLT Muon reconstruction diagram
///


## Streamlining 

Since, in Phase 2, the performance for these approaches is mostly compatible, the streamlining refers to the possibility of using a single reconstruction path for all candidates (either Inside-Out or Outside-In) and perform the second pass only for candidates that were not reconstructed during the first one or whose quality wasn't good enough. 
In particular, the selector module was designed to be flexible enough to allow the selection of either Inside-Out or Outside-In reconstruction first, via a [simple parameter change in the module's configuration file](https://github.com/cms-sw/cmssw/blob/059eb10d66057da8cb5931d01733065475096f2a/HLTrigger/Configuration/python/HLT_75e33/modules/hltPhase2L3MuonFilter_cfi.py#L7).
Technical details can be found in the implementation PR [#46897](https://github.com/cms-sw/cmssw/pull/46897) where performance plots as well as specific implementation details are discussed.

### Inside-Out first

The Inside-Out first approach has shown better performance both in terms of timing and of physics output during the development, thus it is the default approach. It is used in all `.777` workflows and can be activated on any workflow using the option `--procModifiers phase2L2AndL3Muons`. The simplified reconstruction workflow is shown in the following figure:

![HLT Muon reconstruction diagram](figures/hlt_muon_reco_IO_first_diagram.png)
/// caption
Phase-2 Inside-Out first HLT Muon reconstruction diagram
///

When this approach is used, L1 Tracker Muons are used to seed and produce both Standalone Muons (as described in the section on the [new Phase 2 seeding](/docs/phase2_muons/new_seeder.md)) and L3 Tracker Muons starting from the vertex and moving towards the outer tracker. The two resulting collections are geometrically matched and the quality of the L3 Tracker Muons is assessed, requiring a good-enough $\Chi^2$, at least 1 hit in the pixel and at least 5 in the tracker (similar criteria to the ones applied by the HLT Muon ID) 

The default behaviour is Inside-Out first, as in any `.777` workflow and toggleable via the `phase2L2AndL3Muons` procModifier (from `CMSSW_15_0_0_pre2`). This is an example of the `cmsDriver` command for a step 2 using the new Standalone Muon seeding and the L3 Inside-Out first approach:

```bash
cmsDriver.py step2 \
-s DIGI:pdigi_valid,L1TrackTrigger,L1,L1P2GT,DIGI2RAW,HLT:@relvalRun4 \ 
--conditions auto:phase2_realistic_T33 \
--datatier GEN-SIM-DIGI-RAW \
--eventcontent FEVTDEBUGHLT \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--procModifiers phase2L2AndL3Muons \
--filein file:step1.root \
--fileout file:step2.root \
-n 10
```

The Outside-In first reconstruction is available in any `.778` workflow and can be activated by using the procModifier `phase2L3MuonsOIFirst` on top of `phase2L2AndL3Muons` (from `CMSSW_15_0_0_pre2`). This is an example of the `cmsDriver` command for a step 2 using the new Standalone Muon seeding and the L3 Outside-In first approach:

```bash
cmsDriver.py step2 \
-s DIGI:pdigi_valid,L1TrackTrigger,L1,L1P2GT,DIGI2RAW,HLT:@relvalRun4 \ 
--conditions auto:phase2_realistic_T33 \
--datatier GEN-SIM-DIGI-RAW \
--eventcontent FEVTDEBUGHLT \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--procModifiers phase2L2AndL3Muons,phase2L3MuonsOIFirst \
--filein file:step1.root \
--fileout file:step2.root \
-n 10
```