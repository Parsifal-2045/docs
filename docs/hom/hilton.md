# How to test new menus on Hilton
First things first, connect to one of the `hilton` nodes, login as `hltpro` and refresh the kerberos ticket for `/eos` access
```bash
ssh cmsusr
```
```bash
ssh -Y $(logname)@hilton-c2b01-44-01
```
```bash
sudo -u hltpro -i
cd ${HOSTNAME//-/_}/hilton
sudo kdestroy
kinit $(logname)@CERN.CH
```
This is a great time to setup git and check if the [Hilton repository](https://gitlab.cern.ch/cms-tsg/fog/hilton/-/tree/master) was updated (you'll be prompted to enter your credentials both times)
```bash
git fetch cms-tsg-fog
```
```bash
git pull cms-tsg-fog master
```
If the fetch fails the proxy through cmsusr went down, simply activate it via 
```bash
ssh -f -N $(logname)@cmsusr.cms
```
making sure that the ssh config file on your home directory on cmsusr contains something like 
```bash
Host cmsusr cmsusr.cms
  DynamicForward            18080
```
Now check that the submodules used by Hilton are up to date
```bash
git submodule sync --recursive
git submodule update --init
```
To commit changes some git setup is required. Firstly you should have a fork of the main Hilton repository under your CERN account on gitlab. 

- Add it to the remotes
```bash
git remote add $(logname) https://gitlab.cern.ch/$(logname)/hilton.git
``` 

- Modify the author/committer details as well as the default text editor for commit messages:
```bash
export GIT_AUTHOR_NAME='Luca Ferragina'
export GIT_AUTHOR_EMAIL='luca.ferragina@cern.ch'
export GIT_COMMITTER_NAME=${GIT_AUTHOR_NAME}
export GIT_COMMITTER_EMAIL=${GIT_AUTHOR_EMAIL}
export GIT_EDITOR=vim
```

Now changes will be committed using this information and you can simply push them to your fork (and then open a Merge Request to the main repository).

General menu testing follows 4 steps:

- setup the CMSSW release (**make sure to change it when necessary**)
```bash
source setup.sh -r CMSSW_15_0_5
```

- Bonus: make sure that the parameters in `fffParameters.jsn` are updated. In this case
```bash
{
   "SCRAM_ARCH" : "el8_amd64_gcc12",
   "CMSSW_VERSION" : "CMSSW_15_0_5",
   "TRANSFER_MODE" : "TIER0_TRANSFER_ON"
}
```

- check HLT compatibility with L1 menu
```bash
./newHiltonMenu.py <path to HLT menu> --run <run number>
```
HLT menus will usually be under `/cdaq/test/username/...`. The run number will also specify which data to use, check the file `inputFilesRAW.py` to verify which runs are available. You can also add more data checking the storage location on `/eos`. Recent run data will be under `/eos/cms/tier0/store/data/Run2025A/HLTPhysics/RAW/v1/000` followed by the run number (check [OMS](https://cmsoms.cern.ch/cms/run_3/commissioning_2025) for recent runs) and can be added to inputFilesRAW.py by simply adding `root://eoscms.cern.ch/` before the `/eos` path found on `lxplus`.
**Make sure that the input data does not have lumisections with 0 events**, because that will silently fail some jobs and results will look weird.

- having selected a suitable run, execute the test
```bash
./cleanGenerateAndRun.sh --run <run number> (optional --maxEvents n)
```
Optionally you can also force the configuration to re-run the L1 in cases where you want to test the compatibility with a new L1 key using the options `--l1 L1Menu_Collisions2025_v1_0_0_xml --l1-emulator uGT` (where the `xml` should in general be provided by the L1 DOC).

- check the results
```bash
./monitorRatesMultiLumi.py /fff/BU0/output/run<run number>/streamHLTRates/data/run<run number>_ls*.jsndata
```
You should see that the first two colums of specific paths (like `HLTriggerFirstPath` or `Status_OnGPU`) have the same counts as the number of events. Sample [e-log](https://cmsonline.cern.ch/webcenter/portal/cmsonline/pages_common/elog?__adfpwp_action_portlet=683379043&__adfpwp_backurl=https%3A%2F%2Fcmsonline.cern.ch%3A443%2Fwebcenter%2Fportal%2Fcmsonline%2Fpages_common%2Felog%3F__adfpwp_mode.683379043%3D1&_piref683379043.strutsAction=%2FviewMessageDetails.do%3FmsgId%3D1258051) with test results
