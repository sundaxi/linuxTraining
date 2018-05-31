# Azure_linux_waagent_session.md

### Provisioning Log Parsing

#### Daemon Initialization

- **Azure Linux Agent versioon: 2.2.14. [hard coded]**
- **OS version: redhat 7.3.**  
  Get via python Platform module. 
- **Python: 2.7.5. Get the current running python version.** 
  Get via python sys module
- **Check current pid and write it to /var/run/waagent.pid**
- **Check if openssl_fips enabled**
  If yes, set system environment OPENSSL_FIPS_ENVIRONMENT=1
- **Create waagent lib dirctory** 
  mkdir /var/lib/waagent
- **Check if scvmm enabled(VMM agent)**
- **Activate resource disk and format filesystem for resource disk** 
  The temp drive has partition by default, according to waagent configurartion. Mount the disk to target mountpoint and use mkswap to mount the swap. 
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
    check option 245 in /var/lib/dhclient/dhclient.leases
  - **(optional) Build DHCP request**
    If above failed, build DHCP request

- **Detect Procotol** 

  - **Check Wire API version**
    currently 2012-11-30 

    ```bash
    curl -H 'User-Agent: WALinuxAgent/FlexibleVersion ('2.2.14', '.', ('alpha', 'bet http://168.63.129.16/?comp=versions
    ```

  - **Generate Transport self-sign certificate**
    cert: TransportCert.pem private key: TransportPrivate.pem

  - **Forcely update goal state**
    It's the first time to update the goal-state

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

- **Report Provisioning Succeed/Complete**

  ```bash
  cat /sys/class/dmi/id/product_uuid > /var/lib/waagent/provisioned
  ```

  Thumbprint will be used as unque ID

- **Enable RDMA if enabled** 

#### Update Handler Run Latest

- **Get Latest Agent**
  check /var/lib/waagent/WALinuxAgent-x.x.xx folder. 
- **Run subprocess(child process)**
  `python -u bin/WALinuxAgent-2.2.25-py2.7.egg -run-exthandlers`
  The default poll time is 60s, but it's 1s when it was first time provisioned. 

#### Update Handler Run

- **Run monitor handler thread** 
  Collect basic system information and append to Telemetry log.
  Check the event logs every 1 minute, collect and send event 
  Heartbeat is 30mins. Collect IOerror of wireserver/hostplugin/other statistics 

- **Run Env Handler thread**
  Check every 5 sec

  - **If use built-in DHCP, network configuration will be set by Env handler**
  - **Monitor the hostname change**
    it will publish the hostname change automatically (Can disable in agent configuration)
  - **Remove the net-rule files** 
  - **Set ISCSI device timeout, default 300s**
  - **Check DHCP is running.** 
  - **Purge cache files Maxisum 50**

- **Check upgrade avaiable** 
  Check update every 1h. 

  - **Parse ExtensionConfig.1.xml file and get the agent's manifest.**
    The interesting thing is agent though itself as an extension 

  - **Also get the statusblob from ExtensionConfig.1.xml file**

    ```bash
    echo "cat //StatusUploadBlob/text()"|xmllint --shell ExtensionsConfig.1.xml|sed 's/\$system.*//g'|sed '1d;$d'
    ```

  - **Download Agent egg via HostPlugin** 
    hostplugin: 168.63.129.16:32526

  - **Get the latest Agent again** 
    And exthandlers is running on goal state version 
