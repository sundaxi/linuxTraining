# Azure_linux_waagent_session.md

Table of Contents
=================
      * [Azure Waagent Session](#azure-waagent-session)
         * [Introduction](#introduction)
         * [Code Logics](#code-logics)
            * [Daemon Initialization](#daemon-initialization)
            * [Provioning Hanlder](#provioning-hanlder)
            * [Provisioning Starting](#provisioning-starting)
            * [Update Handler Run Latest](#update-handler-run-latest)
            * [Update Handler Run(subprocess start working)](#update-handler-runsubprocess-start-working)
            * [exthandler Run](#exthandler-run)
            * [supplementary](#supplementary)


### Introduction

This document is to help you understand better how Azure WaLinuxagent works. This document is also used for my Linux training session purpose. 

### Code Logics 

#### Daemon Initialization

- **Get Azure Linux Agent versioon: 2.2.14. [hard coded]**
- **Get OS version: redhat 7.3.**  
  Get via python Platform module. 
- **Python: 2.7.5. Get the current running python version.**   
  Get via python sys module
- **Check current pid and write it to /var/run/waagent.pid**  
- **Check if openssl_fips enabled **   
  If yes, set system environment OPENSSL_FIPS_ENVIRONMENT=1
- **Create waagent lib dirctory**   
  mkdir /var/lib/waagent
- **Check if scvmm enabled(VMM agent)**
- **Activate resource disk and format filesystem for resource disk** 
  The temp drive has partition by default, according to waagent    configurartion. Mount the disk to target mountpoint and use mkswap to mount the swap.   
  *how to find out the temp disk* ?   
  Traverse device under /sys/bus/vmbus/device and find out the device with guid start with '00000000-0001'  
  ex: `/sys/bus/vmbus/devices/00000000-0001-8899-0000-000000000000/host3/target3:0:1/3:0:1:0/block/sdb `
- **Clean protocol** 
  Remove /var/lib/waagent/procotol 

#### Provioning Hanlder

- **Check if provision enabled in /etc/waagent.conf**
  If disabled, skip provisioning `cat /sys/class/dmi/id/product_uuid > /var/lib/waagent/provisioned`  
  Report health "{'Provisioning': 'Ready'}"  

- **Check if the system was already provisioned** 
  check if `/var/lib/waagent/provisioned` exsits. Exit provisioning hanlder directly if file exsits

- **Check if cloud-init is running.** 
  Once provisioning is enabled on Waagent, it doesn't allow cloud-init running. 

- **Copy OVF ENV** 

  - **Mount DVD**
    Loaded /lib/modules/2.6.32-696.23.1.el6.x86_64/kernel/drivers/ata/ata_piix.ko driver for ATAPI CD-ROM  
    make sure module udf or iso9660 were not blacklist 
  - **copy the ovf-env.xml**
    /var/lib/waagent/ovf-env.xml. The password will be erased after copy.
    Note: on-prem has tag file. Copy if tag file exists
  - **umount dvd and eject DVD**
    eject /dev/sr0     

- **Detect protocol by File** 
  Check if /var/lib/waagent/Procotol exists.   
  If exsits, get the procotol from the Protocol file   
  If not, check if tag file exists, if not, WireProtocol.    

- **Detect endpoint**

  - **Determine endpoint via DHCP handler** 
    note that DHCP process was already running.  
  - **Check route table**
    check if wireserver was in `/proc/net/route`
  - **Alternatively, check DHCP release** 
    check option DNS cache in /var/lib/dhclient/dhclient.leases. Use DNS server as Wire Endpoint(This should be fixed https://github.com/Azure/WALinuxAgent/issues/1176 )
  - **(optional) Build DHCP request**
    If above failed, build DHCP request

- **Detect Procotol** 

  - **Check Wire API version**
    currently 2012-11-30 

    ```bash
    curl -H 'User-Agent: WALinuxAgent/FlexibleVersion ('2.2.14', '.', ('alpha', 'beta', 'rc'))' http://168.63.129.16/?comp=versions
    ```

  - **Generate Transport self-sign certificate**
    cert: TransportCert.pem private key: TransportPrivate.pem

  - **Forcely update goal state**
    It's the first time to update the goal-state   
    Difference between update goal state and forcely update goal state. For update goal state, if Incarnation value doesn't change, no additional action accutally.  

- **Update Goal state** 

  - **Get goal state xml (max retry 3)**
    cache on /var/lib/waagent/GoalState.1.xml

    ```bash
    curl -H 'User-Agent: WALinuxAgent/FlexibleVersion ('2.2.14', '.', ('alpha', 'beta', 'rc'))' -H 'x-ms-version:2012-11-30' -H 'x-ms-agent-name:WALinuxAgent' http://168.63.129.16/machine/?comp=goalstate
    ```

    Obj goal state(troubleshooting from python)

    ```python
    from azurelinuxagent.common.protocol.wire import GoalState
    import azurelinuxagent.common.utils.fileutil as fileutil
    goalstate = GoalState(fileutil.read_file('/var/lib/waagent/GoalState.1.xml'))
    ```

  - **Get HostingEnvironmentConfig** 

    ```bash
    curl -H 'User-Agent: WALinuxAgent/FlexibleVersion ('2.2.14', '.', ('alpha', 'beta', 'rc'))' -H 'x-ms-version:2012-11-30' -H 'x-ms-agent-name:WALinuxAgent' 'http://168.63.129.16:80/machine/2f196a05-4296-4a33-9bbc-8d15d87cbfa0/534e4d2c%2D4f05%2D40b0%2D9457%2Dbfa64b6ff8c0.%5Fyingrhel74?comp=config&type=hostingEnvironmentConfig&incarnation=3'
    ```

  - **Get Shared Config** 
    same as HostingEnvironmentConfig

  - **Update certificates**  
    /var/lib/waagent/certificates.pem(signed by CRP). Also for public key authentication 

  - **Get Extension Config**  

    ```bash
    curl -H 'User-Agent: WALinuxAgent/FlexibleVersion ('2.2.14', '.', ('alpha', 'beta', 'rc'))' -H 'x-ms-version:2012-11-30' -H 'x-ms-agent-name:WALinuxAgent' 'http://168.63.129.16:80/machine/2f196a05-4296-4a33-9bbc-8d15d87cbfa0/534e4d2c%2D4f05%2D40b0%2D9457%2Dbfa64b6ff8c0.%5Fyingrhel74?comp=config&type=extensionsConfig&incarnation=3'
    ```

  - **Update HostPlugin(Optional)** 
    if hostplugin exisits, assign container id/role config name to hostplugin object

- **Write Endpoint and Save Protocol** 
  /var/lib/waagent/WireServerEndpoint and /var/lib/waagent/Protocol 

#### Provisioning Starting

- **Report Provisioning State** 
  Provisioning: Starting 

- **Set Hostname** 
  set hostname to /etc/hostname and run hostname [hostname]

- **Publish Hostname** 
  configure "send host-name" to publish the hostname to DHCP server
  /var/lib/waagent/published_hostname
  note: the network interface will be restart after hostname change...

- **Config User Account** 

  - **Add user and modify the password**
    system user doesn't allow to change password
  - **Configure Sudoers** 
    /etc/sudoers.d/waagent. PubKey: NOPASSWD
  - **Disable Password Login if Pub key used** 
    client interval timeout 180s
  - **Deploy Public Key**
    The public key was generated from Certificate.crt when provisioning... 

- **Save Custom Data**
  decode the base64 code and save it to /var/lib/waagent/CustomData and then run the script

- **Delete Root Password**

- **Register ssh host key**

  ```bash
  ssh-keygen -A
  ssh-keygen -N '' -t rsa -f /etc/ssh/ssh_host_rsa_key
  Check fingerprint 
  ssh-keygen -lf /etc/ssh/ssh_host_rsa_key.pub
  ```

- **Restart sshd service** 

- **Report Provisioning Succeed/Complete**

  ```bash
  cat /sys/class/dmi/id/product_uuid > /var/lib/waagent/provisioned
  ```

  Thumbprint will be used as unque ID with Contianer ID/RoleInstance ID and report to Wireserver   

- **Enable RDMA if enabled**   
  setup_rdma_device  

#### Update Handler Run Latest

- **Check upgrade avaiable**  
  Check update every 1h. 

- **Get Latest Agent**  
  check /var/lib/waagent/WALinuxAgent-x.x.xx folder.     

- **Parse ExtensionConfig.1.xml file and get the agent's manifest.**
  The interesting thing is agent though itself as an extension.  Download the Waagent based on Prod.[Incarnation].manifest.xml.  Agent list could be found in Prod.[Incarnation].agentsManifest   

- **Also get the statusblob from ExtensionConfig.1.xml file**

  ```bash
  echo "cat //StatusUploadBlob/text()"|xmllint --shell /var/lib/waagent/ExtensionsConfig.1.xml|sed 's/\$system.*//g'|sed '1d;$d'
  ```

- **Download Agent egg via HostPlugin** 
  hostplugin: 168.63.129.16:32526

![hostPlugin.png](https://github.com/sundaxi/materials/blob/master/pics/waagent/hostPlugin.png?raw=true) 

- **Run subprocess(child process)**
  `python -u bin/WALinuxAgent-2.2.25-py2.7.egg -run-exthandlers`   
  When the agent first start up, there is no Latest Agent avaiable (Codes Logic), so agent will start up the subprocess by main process within the same version.   
  The default poll time is 60s, but it's 1s when it was first time provisioned. Father process check the child process every 1s. Terminate the old version child process and start up the latest subprocess.   

![Agent_subprocess_1.png](https://github.com/sundaxi/materials/blob/master/pics/waagent/Agent_subprocess_1.png?raw=true) 

![Agent_subprocess_2.png](https://github.com/sundaxi/materials/blob/master/pics/waagent/Agent_subprocess_2.png?raw=true) 

- **At this moment, main process do nothing else.** 
  ret = self.child_process.wait()  

#### Update Handler Run(subprocess start working)

`python -u bin/WALinuxAgent-2.2.25-py2.7.egg -run-exthandlers`

- **Run monitor handler thread** 
  Collect basic system information and append to Telemetry log.
  Check the event logs every 1 minute, collect and send event 
  Heartbeat is 30mins.   

  - Report Event heatbeat with firewall dropped packets 
  - Collect IOerror of wireserver/hostplugin/other statistics

  also report vminfo to wireserver Telemetry URI     

  ```python
  python -c 'import pprint; from azurelinuxagent.common.protocol import get_protocol_util;protocol_util = get_protocol_util();protocol = protocol_util.get_protocol();vminfo = protocol.get_vminfo();pprint.pprint(vars(vminfo))'
  ```

  WALinuxAgent collects usage data and sends it to Microsoft to help improve our products and services. The data collected is used to track service health and assist with Azure support requests.  WALinuxAgent does not support disabling telemetry at this time. WALinuxAgent must be removed to disable telemetry collection. If you need this feature, please open an issue in GitHub and explain your requirement.  

- **Run Env Handler thread**
  Check every 5 sec

  - **If use built-in DHCP, network configuration will be set by Env handler**

  - **Monitor the hostname change**
    it will publish the hostname change automatically (Can disable in agent configuration)  

  - **Remove the net-rule files**   
    /lib/udev/rules.d/75-persistent-net-generator.rules and /etc/udev/rules.d/70-persistent-net.rules 

  - **Insert Firewall rules for WireServer**   
    If firewall was enabled by waagent.conf(OS.EnableFirewall=y)  

    ```bash 
    # iptables -L -n -t security
    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination
    OUTPUT_direct  all  --  0.0.0.0/0            0.0.0.0/0
    ACCEPT     tcp  --  0.0.0.0/0            168.63.129.16        owner UID match 0
    ACCEPT     tcp  --  0.0.0.0/0            168.63.129.16        ctstate INVALID,NEW
    
    ```

  - **Set ISCSI device timeout, default 300s**

  - **Check DHCP is running.**    
    It's only monitor and only check the dhcp service restart(Code logic is incorrect for my understanding... but anyway it's additional functionality which cx is not aware of)  

  - **Purge cache files Maxisum 50**   
    Purge interval is 24h   

  ```bash
  # check the threads 
  ps aux |grep exthandlers
  root       9677  8.0  1.4 394196 24416 ?        Sl   03:21   1:38 python -u bin/WALinuxAgent-2.2.26-py2.7.egg -run-exthandlers 
  
  ps -To pid,tid,tgid,tty,time,comm  -p 9677
   PID    TID   TGID TT           TIME COMMAND
  9677   9677   9677 ?        00:01:38 python
  9677   9697   9677 ?        00:00:00 python
  9677   9701   9677 ?        00:00:01 python
  ```

- **migrate handler state**(deprecated in 2.2.x)

- **Ensure no orphans**   
  make sure no orphan process of waagent under /var/run 

- **Emit restart event**    
  Generate /var/lib/waagent/current_version 

- **Ensure partition file exsits**   
  Generate /var/lib/waagent/partition. Current time take remainder of 100, the value is (0-99)

- **Ensure all the files under /var/lib/waagent/ are ReadOnly**

- **Check Upgrade avaiable Every one hour** 

#### exthandler Run

Loop execution(every 3s).  To handle all the extensions  and report status to Status blob

- **Extension handle install in sequence** ( will enable in parallel)   

- **Get Extension information from ExtensionConfig.[Incarnation].xml ** 

  Plugins > Plugin.location  --- > /var/lib/waagent/[ExtensionName].[Incarnation].manifest.xml

  Plugins > Plugin.state--- > Decide if that extension needs to be enabled   
  Create extension folder for extension logs /var/log/waagent/[extension]

- **Decide Extension Version**  
  Based on the Extension Manifest file, decide the latest version  
  Currently, no downgrade supported. 

- **Check If Handle state file exsits**  
  /var/lib/waagent/[Extensionfolder]/config/HandlerState  
  If exsits, Extension was already installed if not, Download and update settings.   
  The interesting thing is all the extensions will be enabled again once agent restarted. 

- **Download Extension and Update Settings**    
  Update settings writes the running Settings to /var/lib/waagent/[ExtensionFolder]/config/[seq].settings 
  Skip it if extension already installed. 

- **Install Extension**   
  If first time to install the extension.   
  Get the installation command from /var/lib/waagent/[extension]/HandlerManifest.json   
  eg. diagnostic.py -install. Run install command default timeout 900s 

- **Upgrate Extension**  
  If upgrade is avaiable during Decide Extension Version phase   

  - Disable older extension   
  - Copy status file from old folder to new folder 
  - Run Extension Update command on new Extension 
  - Run Extension uninstall command on Older Extension
  - Remove older extension folder 
  - If 'UpdateMode' is set as 'updatewithinstall', then run install command of newer extension again. 

- **Enable Extension**   
  Run Extension enable command 

- **Report Extension status**   
  Report all the extension status after extension handling in sequence 

  - Check extension status by checking config/HandlerStatus  
    use this long command to get the report value 

    ```python
    echo 'from azurelinuxagent.common.protocol import get_protocol_util
    from azurelinuxagent.common.protocol.wire import WireClient
    from azurelinuxagent.common.protocol.restapi import VMStatus
    protocol_util = get_protocol_util()
    protocol = protocol_util.get_protocol()
    client = WireClient("168.63.129.16")
    ext_handlers = protocol.get_ext_handlers()[0]
    from azurelinuxagent.ga.exthandlers import ExtHandlerInstance
    vm_status = VMStatus(status="Ready", message="Guest Agent is running")
    for ext_handler in ext_handlers.extHandlers:
        ext_handler_i = ExtHandlerInstance(ext_handler, protocol)
        handler_status = ext_handler_i.get_handler_status()
        vm_status.vmAgent.extensionHandlers.append(handler_status)
        
    client.status_blob.set_vm_status(vm_status)
    client.status_blob.prepare("PageBlob")
    print(client.status_blob.data)'>test.py;python test.py|jq;rm -rf test.py
    ```

     Modify the value and trick ASC 

    ![trick_ASC.png](https://github.com/sundaxi/materials/blob/master/pics/waagent/trick_ASC.png?raw=true) 
    

- **Clean up outdated Extension** 

#### supplementary

- **InVMArtifactsProfileBlob in Extension.[Incarnation].xml ** 

  ```bash
  curl "$(echo "cat //InVMArtifactsProfileBlob/text()"|xmllint --shell /var/lib/waagent/ExtensionsConfig.$(cat /var/lib/waagent/Incarnation).xml|sed '1d;$d'|sed 's/amp\;//g')" --output -
  ```

- **Check all the waagent active configuration**   

  ```bash
  waagent show-configuration  
  ```

  
