# How to connect to confDB
As of end of April 2025, I've found two ways to successfully connect to both the DAQ and Run3 Offline databases, I'll detail both of them, since it looks like the conifguration itself is quite finnicky and might not always work the same (e.g. the instructions shown at the tutorial work out of the box only from inside the CERN network).

## Tunnel from local machine to `cmsusr`
The setup shown in this section allows to connect to both `Run3 Offline` and `DAQ` databases using a single ssh command, while running the GUI locally. I've tested this setup only on Fedora 42 but any linux system (and probably wsl too) should work.
Make sure that your ssh config file has these entries for connecting to `lxplus` (CERN network) and `cmsusr` (CMS network) **remember to change the `User` field in both to your CERN computing account username**
```bash
Host lxplus  
  Hostname lxplus.cern.ch
  User lferragi
  GSSAPIAuthentication yes
  GSSAPIDelegateCredentials yes
  ForwardAgent yes
  ForwardX11 yes
  ForwardX11Trusted no
  CheckHostIP no
  ProxyJump none

Host cmsusr 
  HostName cmsusr.cern.ch
  User lferragi
  GSSAPIAuthentication yes
  GSSAPIDelegateCredentials yes
  # For GPU machines
  DynamicForward 18080
  # For confDB
  DynamicForward 1080
  # For web proxy
  DynamicForward 1081
  ForwardAgent yes
  ForwardX11 yes
  ForwardX11Trusted no
  ProxyCommand ssh lxplus nc %h 22
  LocalForward 10121 cmsonr1-v.cms:10121
  #LocalForward 10122 cmsonr2-v.cms:10121
  #LocalForward 10123 cmsonr3-v.cms:10121
  #LocalForward 10124 cmsonr4-v.cms:10121
  #LocalForward 10125 cmsintr1-v.cms:10121
  #LocalForward 10126 cmsintr2-v.cms:10121
  #LocalForward 45679 cmsdaqweb.cms:45679
```

On your local machine, open a tunnel to `cmsusr` via the command
```bash
ssh -f -N cmsusr
```
If successfull you should be prompted for both your `lxplus` password and your `cmsusr` one (or only the latter if you have a valid kerberos ticket) and the tunnel should stay up in the background. You can check the status of the connection using 
```bash
ps aux | grep ssh
```
and if anything is wrong with the tunnel, it can simply be killed (`kill <PID>`) and re-opened with the command shown above.
Once the tunnel is open you can start [confDB](https://confdb.web.cern.ch/confdb/v3/gui/)
```bash
git clone https://github.com/cms-sw/hlt-confdb.git
cd hlt-confdb
ant clean
./start
```
At this point both the Run3 Offline and DAQ databases should be accessible:
 
 - `Run3 Offline `: Select the `Run3 Offline (socks tunnel)` setup, default parameters should work
 - `DAQ`: Select the `DAQ (direct tunnel)` setup with default parameters

<div class="warning" style='background-color:#E9D8FD; color: #69337A; border-left: solid #805AD5 4px; border-radius: 4px; padding:0.7em;'>
<span>
<p style='margin-top:1em; text-align:center'>
<b>NOTE</b></p>
<p style='margin-left:1em;'>
As of April 2025, if you're trying to connect from outside the CERN network, the socks tunnel setup will not work. To avoid the issue, the names in the <code>Host</code> field should be replaced with their IP addresses: <code> 10.116.96.91, 10.116.96.109, 10.116.96.141 </code>. A more definitive version of this change can be achieved by changing directly the contents of <code>hlt-confdb/src/conf/confdb.properties</code> as in the patch attached below
</span>
</div>

<details>
<summary> confDB patch </summary>

```bash
diff --git a/src/conf/confdb.properties b/src/conf/confdb.properties
index 84bcc676..9f41c22d 100644
--- a/src/conf/confdb.properties
+++ b/src/conf/confdb.properties
@@ -16,7 +16,7 @@ confdb0.proxy=false
 
 confdb1.dbSetup=Run3 Offline (socks tunnel)
 confdb1.dbType=oracle
-confdb1.dbHost=cmsr1-v.cern.ch, cmsr2-v.cern.ch, cmsr3-v.cern.ch
+confdb1.dbHost=10.116.96.91, 10.116.96.109, 10.116.96.141
 confdb1.dbPort=10121
 confdb1.dbName=cms_hlt.cern.ch
 confdb1.dbUser=cms_hlt_v3_w
```
</details>

## vnc on a CERN machine