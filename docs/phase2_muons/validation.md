# Phase 2 Muon validation (HLT + Offline)

// Validation description //

## How to compare DQM outputs
CMSSW provides a tool to compare the validation output before and after changes. 
This is particularly useful in cases where only some (or no) changes are expected in the actual validation plots. 
The only requirements are a CMSSW environment and two DQM files (results of a harvesting step)

<div class="warning" style='background-color:#E9D8FD; color: #69337A; border-left: solid #805AD5 4px; border-radius: 4px; padding:0.7em;'>
<span>
<p style='margin-top:1em; text-align:center'>
<b>NOTE</b></p>
<p style='margin-left:1em;'>
The comparison goes bin by bin. For the results to make sense, both DQM outputs should come from the same input file (step1) and be processed sequentially. 
</span>
</div>

The baseline output file can be renamed but **be sure not to change the name of the new file you want to verify**, since the default name is used to set some parameters. 
The command to start the comparison is:

```bash
compareHistograms.py -b DQM_baseline.root -p DQM_V0001_R000000001__Global__CMSSW_X_Y_Z__RECO.root 
```

where 

 - `-b` is used to set the baseline file path (to compare against)
 - `-p` is used to set the modified file bath (to compare with baseline)

The command will output the number of histograms changed/removed/added and produce a folder containing a DQM file for the base version and one for the modified one where only the different histograms are saved.

