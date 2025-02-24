# Phase 2 HLT Muon objects and workflow
All paths in the current Phase 2 menu use the same algorithms to reconstruct muons, with some of them producing slightly fewer collections than others. 
To the best of the current knowledge, the most complete Phase 2 HLT Muon reconstruction workflow is shown in the following figure:

![HLT Muon reconstruction diagram](figures/associators_diagram.png)
/// caption
Phase-2 HLT Muon reconstruction diagram
///

- All full shapes correspond to tracks (collection of hits in the inner tracker (L3) / muon chambers (L2))
- All empty shapes are Muons (collections of inner + outer tracks, global or inner track + hits / segments in the muon chambers or any combination of these).

The validation for all Phase 2 objects was implemented in [#46860](https://github.com/cms-sw/cmssw/pull/46860). It relies on the multitrack validator, producing performance plots for all the objects shown in the workflow image.

Moreover, [#47069](https://github.com/cms-sw/cmssw/pull/47069) introduced the abilty to produce NanoAOD ntuples for all HLT Muon objects (including from L1). This allows further analysis of the output. The option to produce ntuples is enabled automatically for `.777` and `.778` workflows, but can be applied to any `cmsDriver` by adding `NANOAODSIM` to both the `datatier` and `eventcontent` parameters of the command like in the following example:
```bash
cmsDriver.py step2 \
-s DIGI:pdigi_valid,L1TrackTrigger,L1,L1P2GT,DIGI2RAW,HLT:@relvalRun4,NANO:@MUHLT \
--conditions auto:phase2_realistic_T33 \
--datatier GEN-SIM-DIGI-RAW,NANOAODSIM \
--eventcontent FEVTDEBUGHLT,NANOAODSIM \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--filein file:step1.root \
--fileout file:step2.root \
-n 10
```