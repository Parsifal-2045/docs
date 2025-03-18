# How to perform HLT timing measuments
As any CMS expert would say, there is exactly one way to perform correct and sensible timing measurements. Therefore I will try to outline that way in the following sections, using different approaches:
- [Using the timing server](#using-the-timing-server) 
- [Manually running the timing scripts](#manual-timing-requires-access-to-a-benchmark-machine)

I will also try to detail the production of specific samples to be used when measuring the HLT performance. This is a fundamental prerequisite since we don't want to measure the time taken by the L1 emulation while minimizing the I/O impact having the samples available locally on the machine. 

A general requirement is a CMSSW configuration file to run (what is usually called step2). This can be obtained using the following `cmsDriver` command, which can be further customized to add process modifiers as needed:
```bash
cmsDriver.py Phase2 -s L1P2GT,HLT:75e33_timing \
--processName=HLTX \
--conditions auto:phase2_realistic_T33 \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--customise SLHCUpgradeSimulations/Configuration/aging.customise_aging_1000 \
--eventcontent FEVTDEBUGHLT \
--filein=file:TTbar-PU200_L1_skim_15_0_0.root \
--mc \
--inputCommands="keep *, drop *_hlt*_*_HLT, drop triggerTriggerFilterObjectWithRefs_l1t*_*_HLT" \
-n -1 \
--no_exec \
--output={} \
--python_filename phase2_HLT_Timing.py
```
To add process modifiers one can simply add the option:
```bash
--procModifiers phase2L2AndL3Muons,phase2L3MuonsOIFirst
```

## Using the Timing server
Here I show a short recipe for using the timing server using an area-based submission 
<div class="warning" style='background-color:#E9D8FD; color: #69337A; border-left: solid #805AD5 4px; border-radius: 4px; padding:0.7em;'>
<span>
<p style='margin-top:1em; text-align:center'>
<b>NOTE</b></p>
<p style='margin-left:1em;'>
You need to be on a rhel8 machine for the submission to work since the current reference timing machine is rhel8 based, thus it is suggested to use any of the lxplus8 nodes
</span>
</div>

To submit a timing job using the area-based submission method you want to be in a CMSSW environment (`CMSSW_X_Y_Z/src && cmsenv`), here you should have produced the configuration file for HLT timing using the cmsDriver command shown in the previous section. The submission of the jobs is executed using the [timing repository on gitlab](https://gitlab.cern.ch/cms-tsg/steam/timing). Here is the basic recipe for a submission:
```bash
git clone ssh://git@gitlab.cern.ch:7999/cms-tsg/steam/timing.git
python3 -m venv venv
source venv/bin/activate
pip3 install -r timing/requirements.txt
```
At this point you can see the samples available on the timing server with the command
```bash
python3 timing/submit.py --show-samples
```
The list of samples also shows most of the parameters needed to submit a job. Phase 2 timing uses either a TTbar or a minBias L1 skim from Spring24, note that the TTbar sample is available with two different L1 selections. The full submission command is:
```bash
python3 timing/submit.py \
--ignore-unprescaled-check \
--l1menu L1Menu_Phase2_1500 \
--event-type Phase2C17I13M9_ExtendedRun4D110 \
--input-sample TTBar-PU200 \
phase2_HLT_Timing.py 
```
where the final parameter is the HLT python configuration file.
Once a job has been submitted, you can check its status and the results on the [web dashboard](https://cmshlttiming.app.cern.ch/)

The complete timing server documentation can be found in the [HLT Upgrade docs](https://cmshltupgrade.docs.cern.ch/RunningInstructions/#timing) and in the [timing server twiki](https://twiki.cern.ch/twiki/bin/viewauth/CMS/TriggerStudiesTiming).



## Manual timing (requires access to a benchmark machine)
The Bergamo machines currently used for timing are equipped with 256 physical cores and 4 NVIDIA L4 GPUs. In order to fully utilize the capabilities of these machines, benchmarks are run using the [patatrack-scripts](https://github.com/cms-patatrack/patatrack-scripts) utility. In particular, the current recommended settings for Phase 2 benchmarking are 64 jobs each with 8 threads and 8 streams (this covers all the 512 logical threads of a Bergamo machine). 
You can find an example of a timing script used to measure the HTL Muon reconstruction performance here

<details>
<summary> Timing script </summary>

```bash
#! /bin/bash

hlt_config_names=("current" "IO_first" "OI_first")

echo "Starting HLT benchmark for configurations ${hlt_config_names[@]}"

for config_name in "${hlt_config_names[@]}"; do
  echo "Configuration: $config_name"
  jobs=64
  threads=8
  streams=8
  events=1000
  logdir="logs.$config_name.${jobs}j.${threads}t.${streams}s"
  cfg="phase2_HLT_${config_name}.py"

  # Download patatrack-scripts, if they are not there already.
  if [ ! -d 'patatrack-scripts' ]; then
    echo "Cloning patatrack-scripts repository"
    git clone https://github.com/cms-patatrack/patatrack-scripts --depth 1
  fi

  if [ ! -d "${logdir}" ]; then
    echo "Creating directory for the logs at ${logdir}"
    mkdir -p ${logdir}
  fi

  patatrack-scripts/benchmark -j ${jobs} -t ${threads} -s ${streams} -e ${events} --run-io-benchmark \
    -k Phase2Timing_resources.json --event-skip 100 --event-resolution 25 --wait 30 \
    --logdir ${logdir} -- ${cfg} | tee ${logdir}/output.log
  mergeResourcesJson.py ${logdir}/step*/pid*/Phase2Timing_resources.json > Phase2Timing_resources.json

  script_dir=$(dirname -- "$0")
  if [ -f "$script_dir/augmentResources.py" ]; then
    echo "Running augmentResources.py"
    python3 $script_dir/augmentResources.py
  fi

  cp -p Phase2Timing_resources.json ${logdir}
  cp -p Phase2Timing_resources_abs.json ${logdir}
  cp -p ${cfg} ${logdir}

done

echo "All HLT configurations have been processed successfully."
```

</details>

In this case, I have produced three HLT configuration files:

- `phase2_HLT_current.py`: baseline to compare against
- `phase2_HLT_IO_first.py`: HLT configuration including changes to the Standalone Muon seeding and executing L3 Tracker Muon reconstruction Inside-Out first
- `phase2_HLT_OI_first.py`: HLT configuration including changes to the Standalone Muon seeding and executing L3 Tracker Muon reconstruction Outside-In first

The script benchmarks each of these configuration running it three times (after warming up and measuring I/O only). Timing logs from the CMSSW Timing service are also produced (one folder per configuration). These can be visualized as usual with the circles utility running on a web server.

## Producing input skims
// re-run L1 on top of Spring24 samples //
```bash
cmsDriver.py Phase2 -s L1,L1TrackTrigger \
--processName=SKIM \
--conditions auto:phase2_realistic_T33 \
--geometry ExtendedRun4D110 \
--era Phase2C17I13M9 \
--eventcontent FEVTDEBUGHLT \
--datatier GEN-SIM-DIGI-RAW-MINIAOD \
--customise SLHCUpgradeSimulations/Configuration/aging.customise_aging_1000,Configuration/DataProcessing/Utils.addMonitoring,L1Trigger/Configuration/customisePhase2FEVTDEBUGHLT.customisePhase2FEVTDEBUGHLT,L1Trigger/Configuration/customisePhase2TTOn110.customisePhase2TTOn110 \
--filein root://cmsxrootd.fnal.gov///store/mc/Phase2Spring24DIGIRECOMiniAOD/TT_TuneCP5_14TeV-powheg-pythia8/GEN-SIM-DIGI-RAW-MINIAOD/PU200_AllTP_140X_mcRun4_realistic_v4-v1/2560000/11d1f6f0-5f03-421e-90c7-b5815197fc85.root \
--fileout file:output_Phase2_L1T.root \
--python_filename rerunL1_cfg.py \
--inputCommands="keep *, drop l1tPFJets_*_*_*, drop l1tTrackerMuons_l1tTkMuonsGmt*_*_HLT" \
--outputCommands="drop l1tTrackerMuons_l1tTkMuonsGmt*_*_HLT" \
--mc \
-n -1 \
--no_exec
```