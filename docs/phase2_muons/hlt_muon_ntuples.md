# NANO AOD HLT Muon objects

Starting from `CMSSW_15_1_X`, it is now possible to produce ntuples of HLT Muon objects, together with GEN Muons and L1 Tracker Muons. The option is on by default in `.777` and `.778` workflows, but can be manually triggered using the following `cmsDriver.py` options:

- `NANO:@MUHLT` in the `-s` option
- `NANOAODSIM` in both the `--eventcontent` and `--datatier`

A full `cmsDriver` command might look something like this:

```bash
cmsDriver.py step2 \
-s DIGI:pdigi_valid,L1TrackTrigger,L1,L1P2GT,DIGI2RAW,HLT:@relvalRun4,NANO:@MUHLT \
--conditions auto:phase2_realistic_T33 \
--datatier GEN-SIM-DIGI-RAW,NANOAODSIM \
-n 10 \
--eventcontent FEVTDEBUGHLT,NANOAODSIM \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--pileup AVE_200_BX_25ns \
--pileup_input das:/RelValMinBias_14TeV/CMSSW_14_1_0-141X_mcRun4_realistic_v1_STD_RegeneratedGS_2026D110_noPU-v1/GEN-SIM \
--filein file:step1.root \
--fileout file:step2.root
```

Validation can also be attached to this same reconstruction step via the following options:

- `VALIDATION` in the `-s` option
- `DQMIO` in both the `--eventcontent` and `--datatier`

In this way, the full `cmsDriver` command becomes:

```bash
cmsDriver.py step2 \
-s DIGI:pdigi_valid,L1TrackTrigger,L1,L1P2GT,DIGI2RAW,HLT:@relvalRun4,NANO:@MUHLT,VALIDATION:@hltValidation \
--conditions auto:phase2_realistic_T33 \
--datatier GEN-SIM-DIGI-RAW,NANOAODSIM,DQMIO \
-n 10 \
--eventcontent FEVTDEBUGHLT,NANOAODSIM,DQMIO \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--pileup AVE_200_BX_25ns \
--pileup_input das:/RelValMinBias_14TeV/CMSSW_14_1_0-141X_mcRun4_realistic_v1_STD_RegeneratedGS_2026D110_noPU-v1/GEN-SIM \
--filein file:step1.root \
--fileout file:step2.root
```

Note the usage of `@hltValidation` as a specifier of the `VALIDATION` step. This allows to run the HLT validation directly during a `step2` without the need of performing the full reconstruction. Using this option, the harvesting of the validation plots results from the following `cmsDriver` command:

```bash
cmsDriver.py step5 \
-s HARVESTING:@hltValidation \
--conditions auto:phase2_realistic_T33 \
--mc \
--geometry ExtendedRun4D110 \
--scenario pp \
--filetype DQM \
--era Phase2C17I13M9 \
--filein file:step2_inDQM.root \
--pileup AVE_200_BX_25ns \
--pileup_input das:/RelValMinBias_14TeV/CMSSW_14_1_0-141X_mcRun4_realistic_v1_STD_RegeneratedGS_2026D110_noPU-v1/GEN-SIM
```