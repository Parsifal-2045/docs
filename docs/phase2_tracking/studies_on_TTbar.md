# Beamspot comparison (No PU)

Comparing the realistic HL-LHC beamspot condition with an ideal pointlike one requires to generate two different samples (step1). 
These are the `cmsDriver` commands used to generate the 5k events TTbar sample used for these studies (`CMSSW_15_0_0_pre3`).

- Pointlike beamspot
```bash
cmsDriver.py TTbar_14TeV_TuneCP5_cfi \
-s GEN,SIM \
--conditions auto:phase2_realistic_T33 \
--beamspot NoSmear \
--datatier GEN-SIM \
--eventcontent FEVTDEBUG \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--relval 9000,100 \
-n 5000 \
--no_exec
```

- Realistic HL-LHC beamspot
```bash
cmsDriver.py TTbar_14TeV_TuneCP5_cfi \
-s GEN,SIM \
--conditions auto:phase2_realistic_T33 \
--beamspot DBrealisticHLLHC \
--datatier GEN-SIM \
--eventcontent FEVTDEBUG \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--relval 9000,100 \
-n 5000 \
--no_exec
```

The following step is the same for both beamspot conditions and the configuration is as follows:
```bash
cmsDriver.py step2 \
-s DIGI:pdigi_valid,L1TrackTrigger,L1,L1P2GT,DIGI2RAW,HLT:75e33,VALIDATION \
--conditions auto:phase2_realistic_T33 \
--datatier GEN-SIM-DIGI-RAW,DQMIO \
-n -1 \
--eventcontent FEVTDEBUGHLT,DQMIO \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--procModifiers alpaka \
--no_exec
```

Finally, the harvesting step can be obtained using the command
```bash
cmsDriver.py step5 \
-s HARVESTING:@trackingOnlyValidation+@HLTMon+postProcessorHLTtrackingSequence \
--conditions auto:phase2_realistic_T33 \
--mc \
--geometry ExtendedRun4D110 \
--scenario pp \
--filetype DQM \
--era Phase2C17I13M9 \
-n 10 \
--filein file:step2_inDQM.root
```

<div class="warning" style='background-color:#E9D8FD; color: #69337A; border-left: solid #805AD5 4px; border-radius: 4px; padding:0.7em;'>
<span>
<p style='margin-top:1em; text-align:center'>
<b>NOTE</b></p>
<p style='margin-left:1em;'>
From <code>CMSSW_15_1_X</code> a new option allows to run only the validation for the HLT and its corresponding harvesting. Simply substitute the entire <code>VALIDATION:*</code> (in step2) and <code>HARVESTING:*</code> (in step5) with <code>VALIDATION:@hltValidation</code> and <code>HARVESTING:@hltValidation</code>, respectively
</span>
</div>

All the results form the analysis of `simDoublets` are obtained using the step2 and harvester in the [test folder](https://github.com/cms-sw/cmssw/tree/master/Validation/TrackingMCTruth/test) of the TrackingMCTruth module.

# First optimizer results and Single Muon parameters on TTbar

These results refer to the first test run of the optimizer on a TTbar sample with no PU.
During this optimization round, all the scalar parameters were left floating, including phase-space changing ones like the cut on $p_T$.
This resulted in two agents labeled `78` and `87`, showing better overall performance than the others.
Details on the parameters changed and the updated values are shown in the following table (all the parameters refer to `hltPhase2PixelTrackSoA`, default values are used unless specified otherwise):

|    Parameter    | Default value | Agent 78 | Agent 87 | Single Mu optimization* |
|:---------------:|:-------------:|:--------:|:--------:|:-----------------------:|
|  cellMinYSizeB1 |       25      |     9    |    26    |            10           |
|  cellMinYSizeB2 |       15      |     1    |    14    |            10           |
|    cellZ0Cut    |      7.5      |     6    |     7    |            15           |
| cellMaxDYSize12 |       12      |     3    |     1    |            12           |
|  cellMaxDYSize  |       10      |     7    |     8    |            10           |
|  cellMaxDYPred  |       20      |    53    |    75    |      26* (was 140)      |
|    cellPtCut    |      0.85     |   0.192  |   0.892  |           0.85          |

Single Muon parameters had to be adjusted to avoid crashes like the following:
```bash
cmsRun: /cvmfs/cms.cern.ch/el9_amd64_gcc12/cms/cmssw/CMSSW_15_0_0_pre3/src/HeterogeneousCore/AlpakaInterface/interface/OneToManyAssoc.h:83: void cms::alpakatools::OneToManyAssocBase<I, ONES, SIZE>::count(const TAcc&, I) [with TAcc = alpaka::AccCpuSerial<std::integral_constant<long unsigned int, 1>, unsigned int>; I = unsigned int; int ONES = 16; int SIZE = 262144]: Assertion `b < static_cast<uint32_t>(nOnes())' failed.
```
while at first it looked like we had too many tracks, the actual cause of the error was the maximum number of hits per track (`maxHitsOnTrack`), set in [CAStructures.h](https://github.com/cms-sw/cmssw/blob/1a6eba4a2910fa7af4d98b4c35a7ae501d9a1615/RecoTracker/PixelSeeding/plugins/alpaka/CAStructures.h#L28) using the value set in [SimplePixelTopology.h](https://github.com/cms-sw/cmssw/blob/1a6eba4a2910fa7af4d98b4c35a7ae501d9a1615/Geometry/CommonTopologies/interface/SimplePixelTopology.h#L328), I observed more than one case in which the number of hits was 15, thus exactly the same as the upper limit, causing the assertion to fail. By purely geometrical considerations (so not considering the possibility of keeping some extra hits saved after fishbone), the largest amount of hits per track should be 18 on average, thus the parameter `maxHitsOnTrack` has been increased to 18 for PU studies.

# Studies with PU

At this point we've decided to fix the value of the pT cut as to not change the phase-space when validating and comparing results. Vector parameters have also been left untouched since, for the most part, they are related to coarse cuts meant to reduce combinatorics.
Studies with PU can be carried out in two ways:

- Mixing PU on the same sample used for the previous studies
- Using RelVals with PU already mixed in

The recipe to mix PU on the TTbar sample used before is
```bash
cmsDriver.py step2 \
-s DIGI:pdigi_valid,L1TrackTrigger,L1,L1P2GT,DIGI2RAW,HLT:75e33,VALIDATION \
--conditions auto:phase2_realistic_T33 \
--datatier GEN-SIM-DIGI-RAW,DQMIO \
-n -1 \
--eventcontent FEVTDEBUGHLT,DQMIO \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--pileup AVE_200_BX_25ns \
--pileup_input das:/RelValMinBias_14TeV/CMSSW_14_1_0-141X_mcRun4_realistic_v1_STD_RegeneratedGS_2026D110_noPU-v1/GEN-SIM \
--procModifiers alpaka \
--no_exec
```

While, to use RelVals the recipe is slightly different since running the DIGI step on a premixed input file **removes** the PU information. Therefore, the recipe to run on RelVals is 
```bash
cmsDriver.py step2 -s L1P2GT,HLT:75e33,VALIDATION \
--processName HLTX \
--conditions auto:phase2_realistic_T33 \
--datatier GEN-SIM-DIGI-RAW,DQMIO \
-n 10 \
--eventcontent FEVTDEBUGHLT,DQMIO \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--procModifiers alpaka \
--filein file:/eos/cms/store/relval/CMSSW_15_0_0_pre3/RelValTTbar_14TeV/GEN-SIM-DIGI-RAW/PU_141X_mcRun4_realistic_v3_STD_Run4D110_PU-v2/2580000/01b8c5dd-42a1-46b7-8607-f4c990ba3ab3.root \
--fileout file:step2.root
```
This second approach is also the favourite one to run The Optimizer, since mixing is very slow.

Some preliminary configurations have been tested (no optimizer), to verify general behaviours and compare to legacy, as shown in the following table:

|    Parameter    | Default value | Single Mu optimization (SM) |  MAX |
|:---------------:|:-------------:|:---------------------------:|:----:|
|  cellMinYSizeB1 |       25      |              10             |   1  |
|  cellMinYSizeB2 |       15      |              10             |   1  |
|    cellZ0Cut    |      7.5      |              15             |  50  |
| cellMaxDYSize12 |       12      |              12             |  140 |
|  cellMaxDYSize  |       10      |              10             |  140 |
|  cellMaxDYPred  |       20      |             140             |  140 |
|    cellPtCut    |      0.85     |             0.85            | 0.85 |

The resulting plots and DQM files are stored [here](https://lferragi.web.cern.ch/plots/tracking_phase2/ttbarPU_studies/) and where obtained using the `makeTrackValidationPlots.py` utility provided by CMSSW. The full documentation regarding tracking validation can be found in [this twiki](https://twiki.cern.ch/twiki/bin/view/CMSPublic/SWGuideMultiTrackValidator#Plotting), while the documentation for the plotter only is at this [link](https://twiki.cern.ch/twiki/bin/viewauth/CMS/TrackingValidationMC#Producing_plots). The basic recipe to produce plots (in a CMSSW environment) is: 
```bash
makeTrackValidationPlots.py -o plots --extended --separate --no-html [--png] $REFERENCE $TARGETS
```
according to the documentation, the plotter supports up to five `DQM` files to compare and superimpose. This command results in a `plots` directory containing separate folders for validation metrics of all the hlt track collections. By default, the plots are produced in `pdf` format, passing the option `--png` before the list of `DQM` files will produce the same plots in `.png` format.


cmsDriver.py step2 -s L1P2GT,HLT:75e33,VALIDATION:@hltValidation \
--processName HLTX \
--conditions auto:phase2_realistic_T33 \
--datatier GEN-SIM-DIGI-RAW,DQMIO \
-n 10 \
--eventcontent FEVTDEBUGHLT,DQMIO \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--procModifiers alpaka \
--filein file:/eos/cms/store/relval/CMSSW_15_1_0_pre1/RelValZMM_14/GEN-SIM-DIGI-RAW/PU_141X_mcRun4_realistic_v3_STD_Run4D110_PU-v1/2580000/012d3900-c324-416c-9c6e-33ebdda2ed6b.root \
--fileout file:step2.root