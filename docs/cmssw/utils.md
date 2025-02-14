# General configuration tips
This section includes a collection of configuration tips and tricks to set up the coding environment and the access to CERN resources 

## Visual Studio Code and Kerberos
This is how I configured Visual Studio code to connect to CERN machines using a kerberos ticket from Windows.

**One-time configuration**

- Install openSSH (available from additiona features in settings)

- Setup the windows ticket manager:

    - Add key (will prompt for password)
    ```cmd
    cmdkey /add:*.cern.ch /user:user@CERN.CH /pass
    ```

    - Register key as kerberos token (run as administrator)
    ```cmd
    ksetup /addrealmflags CERN.CH TcpSupported
    ```

- Edit (or create) the ssh configuration file in `C:\Users\user\.ssh\config`

The ssh configuration file should contain entries similar to these:
```
Host lxplus
  HostName lxplus.cern.ch
  User user
  GSSAPIAuthentication yes
  GSSAPIDelegateCredentials yes

Host CERN_MACHINE
  HostName someMachine.cern.ch
  User user
  GSSAPIAuthentication yes
  GSSAPIDelegateCredentials yes
  ProxyJump lxplus
```

Finally, set the Remote-ssh vscode extension to use the correct configuration file (`C:\Users\user\.ssh\config`) and follow the [guide on Service-Now](https://cern.service-now.com/service-portal?id=kb_article&n=KB0008901). Now all vscode connections through lxplus should work without requiring a password.

<div class="warning" style='background-color:#E9D8FD; color: #69337A; border-left: solid #805AD5 4px; border-radius: 4px; padding:0.7em;'>
<span>
<p style='margin-top:1em; text-align:center'>
<b>NOTE</b></p>
<p style='margin-left:1em;'>
When using the integrated terminal in vscode it might sometimes happen that the kerberos ticket is not refreshed correctly. 
<br>
This results in getting a permission denied error when trying to access the .bashrc
<br>
To solve this issue one can simply execute the kinit and aklog commands to manually refresh the kerberos ticket. Having done so, simply start a new bash session and everything should be back to normal.
</span>
</div>

## CERN Grid certificate
To request a grid certificate vist the [CERN Certificate Authority (CA) website](https://ca.cern.ch/ca/). If this is your first time requesting a certificate follow the instructions in the [help](https://ca.cern.ch/ca/Help/) section to trust CERN CA and install the certificate in your browser.
Otherwise, if you are renewing a certificate that is expiring you can reuse the same password both for the request and to install the new certificate.

Install the new `.p12` certificate file in the browser:
- Settings -> Certificates -> View certificates
- Import 
- Same password as the one used to create the certificate

To install on lxplus / CERN machines copy the `.p12` certificate to lxplus in the `.globus` directory, then:
```bash
openssl pkcs12 -in myCertificate.p12 -clcerts -nokeys -out usercert.pem
openssl pkcs12 -in myCertificate.p12 -nocerts -out userkey.pem
chmod 400 userkey.pem
chmod 444 usercert.pem
```

## Setup a `CMSSW` environment
The required configuration to access `CMSSW` is already present on most machines, so that you can simply 
```bash
scram list CMSSW
```
to get a list of all available `CMSSW` releases. 
However, in some cases the required release is not installed or the machine is not configured for `CMSSW`. In such cases `CMSSW` can be accessed through `cvmfs`. 
First you need to export the correct architecture for scram to produce an executable that can be used on the machine, then source the relevan environment:
```bash
export SCRAM_ARCH=el9_amd64_gcc12
source /cvmfs/cms.cern.ch/cmsset_default.sh
```
If you are unsure on which architecture should be used for a specific machine, you can query information about the operating system and the processor installed by using the commands `hostnamectl` and `lscpu`, respectively.

At this point, 
```bash
scram list CMSSW 
```
produces the list of all available CMSSW releases (can be filtered using grep if you are looking for a specific release).

Create a working area with:
```bash
cmsrel CMSSW_15_0_0_pre2
cd CMSSW_15_0_0_pre2/src
cmsenv
git cms-init
```
If you are starting a fresh development, start by adding the package(s) you want to modify using `git cms-addpkg Package/Module`. You can also recreate working areas, merge them together or rebse them on new releases using the full suit of `git cms` commands (`checkout-topic`, `merge-topic`, `rebase-topic`, ...), fully documented [here](https://cms-sw.github.io/tutorial-resolve-conflicts.html).
If this is your first time developing code for `CMSSW` checkout [this tutorial](https://codimd.web.cern.ch/s/glLKn0Nb-#CMSSW-Tutorial-for-Newcomers) to learn all the basics.

## Compiling with scram
Once the environment is properly set up (`cmsenv`) the project can be compiled with 
```bash
scram b -j `nprocs`
```
`-j N` specifies the number of threads to use during the compilation (0 tries to use all the available threads).

SCRAM also offers code checks and clang-format for all checked-out modules
```bash
scram b code-checks
scram b code-format
```
any change suggested by these commands is necessary to be able to merge a PR.

To activate debug messages compile with the added option
```bash
USER_CXXFLAGS="-DEDM_ML_DEBUG" scram b -j `nproc`
```
or export the flag and compile normally. Then the message logger must be loaded and used in the python config file 
```python
process.load("FWCore.MessageService.MessageLogger_cfi")
process.MessageLogger.cerr.threshold = "DEBUG"
process.MessageLogger.debugModules = ["myModule"]  # or ["*"] for all modules (VERBOSE)
```
Scram uses gcc as the default compiler, but clang is also available using the command 
```bash
scram b COMPILER=llvm
```

## RunTheMatrix, cmsDriver, and cmsRun
runTheMatrix allows to pick and choose a workflow using its designated number. The workflow includes a series of conditions (e.g. Phase-2, geometry, era, PU, ecc.). For workflows whose number does not end with .0, process modifiers are also included. For example, the Phase-2 changes to HLT muon reconstruction are found in all workflows ending with .777 (Inside-Out reconstruction first) and .778 (Outside-In reconstruction first) (SingleMuon, TTbar, and ZMM).
The basic approach to runTheMatrix is listing all available workflows
```bash
runTheMatrix.py -w upgrade -n | grep <workflow>
```
(Some) options
- `-w` tells the matrix which set of workflows to look through (here, Phase-2)
- `-n` only lists the workflows without executing
- `-e` show the detailed cmsDriver commands for the specified workflow
- `-l` comma separated list of the workflows to run or show
- `-t NTHREADS` specifies the number of threads to be used per process in cmsRun
- `-j 0` only creates the python configuration files without executing them
- `--maxSteps N` executes the workflow only up to the specified step (e.g. `--maxSteps 2` will run step1 and step2, meaning GEN-SIM and L1-HLT reconstruction, but no offline reconstruction or harvesting). This option also saves the `cmsDriver` commands for the following steps in a `wf_steps.txt` file

For example
```bash
runTheMatrix.py -w upgrade -n -e -l 29850.0 
```
lists all the steps for workflow `29850.0` without executing them.
One can execute them by simply copy-pasting the cmsDriver commands listed one after the other or by using `runTheMatrix` without the `-n` option.

`runTheMatrix` generates (and by default runs) a series of python configuration files. Each of them is specifically generated by a `cmsDriver.py` command which lists all the conditions and modifiers to apply.
Take the workflow `29850.777` as an example. It corresponds to a Phase-2 ZMM sample with 200 PU, the `.777` modifier introduces the improved L2 Muon seeding from L1 Tracker Muons and attempts L3 Tracker Muon reconstruction Inside-Out first.

The full name of the workflow is: `29850.777 ZMM_14TeV_TuneCP5_2026D110PU_GenSimHLBeamSpot14+DigiTriggerPU_phase2L2AndL3MuonsIOFirst_2026D110PU+RecoGlobalPU_phase2L2AndL3MuonsIOFirst_2026D110PU+HARVESTGlobalPU_phase2L2AndL3MuonsIOFirst_2026D110PU`

```bash
runTheMatrix.py -w upgrade -n -e -l 29850.777
```
produces the following output (note the `-n` to only generate the configuration files and not run them)
<details>
<summary> output </summary>

```bash 
[1]: cmsDriver.py
ZMM_14TeV_TuneCP5_cfi
-s GEN,SIM
-n 10
--conditions auto:phase2_realistic_T33
--beamspot DBrealisticHLLHC
--datatier GEN-SIM
--eventcontent FEVTDEBUG
--geometry Extended2026D110
--era Phase2C17I13M9
--relval 18000,100 

[2]: cmsDriver.py step2
-s DIGI:pdigi_valid,L1TrackTrigger,L1,L1P2GT,DIGI2RAW,HLT:@relval2026
--conditions auto:phase2_realistic_T33
--datatier GEN-SIM-DIGI-RAW
-n 10
--eventcontent FEVTDEBUGHLT
--geometry Extended2026D110
--era Phase2C17I13M9
--procModifiers phase2L2AndL3Muons
--pileup AVE_200_BX_25ns
--pileup_input das:/RelValMinBias_14TeV/CMSSW_14_1_0_pre5-140X_mcRun4_realistic_v4_RegeneratedGS_2026D110_noPU-v1/GEN-SIM

[3]: cmsDriver.py step3  
-s RAW2DIGI,RECO,RECOSIM,PAT,VALIDATION:@phase2Validation+@miniAODValidation,DQM:@phase2+@miniAODDQM
--conditions auto:phase2_realistic_T33
--datatier GEN-SIM-RECO,MINIAODSIM,DQMIO
-n 10
--eventcontent FEVTDEBUGHLT,MINIAODSIM,DQM
--geometry Extended2026D110
--era Phase2C17I13M9
--procModifiers phase2L2AndL3Muons
--pileup AVE_200_BX_25ns
--pileup_input das:/RelValMinBias_14TeV/CMSSW_14_1_0_pre5-140X_mcRun4_realistic_v4_RegeneratedGS_2026D110_noPU-v1/GEN-SIM

[4]: cmsDriver.py step4 
-s HARVESTING:@phase2Validation+@phase2+@miniAODValidation+@miniAODDQM
--conditions auto:phase2_realistic_T33 
--mc 
--geometry Extended2026D110 
--scenario pp 
--filetype DQM 
--era Phase2C17I13M9 
--procModifiers phase2L2AndL3Muons 
-n 10 
--pileup AVE_200_BX_25ns 
--pileup_input das:/RelValMinBias_14TeV/CMSSW_14_1_0_pre5-140X_mcRun4_realistic_v4_RegeneratedGS_2026D110_noPU-v1/GEN-SIM
```

</details>

Some workflows might require access to the GRID (e.g. to retrieve pileup input to mix with signal). Initialise a grid proxy with the command

```bash
voms-proxy-init --rfc --voms cms -valid 192:00
```
[Full documentation to use grid certificates](https://twiki.cern.ch/twiki/bin/view/CMSPublic/WorkBookStartingGrid#BasicGrid)

## Some useful commands

### EDM and ROOT
CMSSW output files are generally in `EDM` format, meaning that they cannot be opened using plain `ROOT` (excluding the results of any harvesting step).
However, in the `CMSSW` environment you have access to some commands to dump the file content and its size:
```bash
# Dump the event content of any root file
edmDumpEventContent <root_file>
# Dump event size
edmEventSize -v <root_file>
# flag to compile root code with g++
`root-config --cflags --glibs`
```

### Integration tests
The standard battery of tests executed when opening a PR can be executed locally by running:
```bash
runTheMatrix.py -l limited -i all --ibeos
```

Another set of standard test can be run with
```bash
addOnTests.py --nproc 2
```

A set of common integration tests can be performed using the command 
```bash
hltPhase2UpgradeIntegrationTests
```
this supports some of the same options as `runTheMatryx`, in particular:
- `--procModifiers` allows to specify any process modifiers to be passed to cmsDriver
- `--threads` sets the number of threads used to run cmsDriver
