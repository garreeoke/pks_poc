# General #

For use in setting up automated deployment for a PKS/NSX-T POC.  

* For use in a guided POC with a VMware PKS Engineer
* These instructions are not supported by VMware or Pivotal
* If you have questions or requests, feel free to post an issue.

## PKS-Client VM ##

1. Download & setup Ubuntu
   * Download Ubuntu 16.04: http://cdimage.ubuntu.com/releases/16.04/release/
   * Create a VM and install ubuntu using all defaults
      * Set VM name to pks-client
      * Create VM with two nics, the second one can be initially disconnected
      * Concourse could take up some space.  100-200GB of disk should be suffice.
   * Login to VM with credentials you created
      * May have to use "sudo" for some commands below.
      * If you are able to unlock root: https://askubuntu.com/questions/44418/how-to-enable-root-login


## Setup Fly Command (CLI for concourse) ##

1. Install fly cli (command line to interact with concourse)
   * $ curl -LO https://github.com/concourse/concourse/releases/download/v4.0.0/fly_linux_amd64
   * $ chmod +x fly_linux_amd64
   * $ mv fly_linux_amd64 /usr/local/bin/fly
   * $ fly --version
       (4.0.0)
   * $ fly --target nsx-concourse login --concourse-url http://localhost:8080 -n main

## Setup NSX-T Pipeline ##
1. Modify nsx_pipeline_config.yml (either from this repo or one sent to you)
   * Search for CHANGE_ME and modify the file for your environment
   * Place file in /home/concourse directory on the pks-client VM
2. Follow instructions at: https://github.com/vmware/nsx-t-datacenter-ci-pipelines/wiki/Deploying-Concourse-using-the-Docker-Imag

## Verify NSX-T ##

### TEP to TEP Communication ###
1. ssh in to two esx hosts that are prepped for NSX
2. On each host:
   * List vmknics: $ esxcfg-vmknic -l
   * Look for IP address of vmk10
   * Ping vmk10 of other host: $ vmkping ++netstack=vxlan -I vmk10 x.x.x.x
   * Ping vmk10 of other host with MTU 1600: $ vmkping ++netstack=vxlan -d -s 1572 -I vmk10 x.x.x.x 

### Within NSX Manager ###
1. Login, go to Fabric from the left menu
   * Verify all hosts have deployment status of NSX Installed
2. Select Edges from the top menu
   * Deployment, Controller, and Manager status should be green
3. Select Transport Nodes from top menu
   * Configuration should be Succes and status should be up
4. Select Networking/Routers from left menu
   * Verify routers creation of T0 and T1 routers (one T0 and two T1s)

### Test ingress/egress ###
1. On a VM or ssh session not deployed on NSX logical switch (your desktop)
   * Ping the logical switch gateways.  In the nsx_pipeline_config.yml file, search for logical_switch_gw. Try pinging each one (Should be two).
2. On the pks-client VM
   * Connect the unused nic to the pks-mgmt logical switch (Use last available IP in the PKS-MGMT block)
   * Ping logical switch gateways (see above)
   * Ping edge gateway vip (search for tier0_ha_vip: in params file)
   * Ping physical switch gateway

# Post NSX-T install and verification #

## PKS Pipeline ##
1. Modify pks-params.yml
2. Download pks pipeline from github
  i. cd /home/concourse
  ii. copy pks-param.yml to /home/concourse
  iii. git clone https://github.com/nvpnathan/nsx-t-ci-pipeline.git
3. Register pipeline
  * fly --target nsx-concourse login --concourse-url http://localhost:8080 -n main
  * fly -t nsx-concourse set-pipeline -p pks-install -c nsx-t-ci-pipeline/pipelines -l ./pks-params.yml
  * fly -t nsx-concourse unpause-pipeline -p pks-install
4. Login to concourse, select the pks-install pipeline from the left menu
5. Run the pipeline
  * First time you run this, each box will run automatically.  Subsequent runs will require clicking on each box individually


## PKS Client Software ##

1. Downlowd & install cli binaris from Pivotal, they will need to be copied to the Ubuntu VM
   * https://network.pivotal.io/products/pivotal-container-service/
     * Download pks cli and kubectl for your linux
   * _PKS_
    ** $ chmod +x pks-linux-amd64-x.x.x-build.x
    ** $ mv pks-linux-amd64-x.x.x-build.x /usr/local/bin/pks
    ** $ pks --version
   *_Kubectl_
    ** $ chmod +x kubectl-linux-amd64-vx.x.x
    ** $ mv kubectl-linux-amd64-vx.x.x /usr/local/bin/kubectl
    ** $ kubectl version
2. Setup Uaac
   * $ apt update
   * $ apt upgrade
   * $ apt install ruby
   * $ apt install ruby-dev
   * $ apt install gcc
   * $ apt-get install build-essential g++
   * $ gem install cf-uaac
   * $ uaac version
   * Add /etc/hosts entry
      * 10.x.x.x pks.your_domain.com
   * $ uaac target https://uaa.mylab.com:8443 --skip-ssl-validation
   * Get the secret 
    * OpsMan->Pivotal Tile->Credentials->Pks Uaa Management Admin Client->Link to Credential
    * Copy the secret value
   * $ uaac token client get admin -s copied_secret_value

__Steps 3 & 4 will not work until after PKS Pipeline is run__
3. Login to PKS
   * $ pks login -a pks.your_domain.com -u username -p password -k
4. Create cluster, get kubernetes credentials, and use kubectl
   * $ pks create-cluster k8s-1 --external-hostname k8s-1 --plan small --num-nodes 3
   * $ pks cluster k8s-1
   * Copy ip addres of Kubernetes Master IP
   * Add entry for k8s-1 in to /etc/hosts
   * $ pks get-credentials k8s-1
   * $kubectl get nodes

## Harbor Cert setup to enable pushing images to Harbor ##

1. Setup docker cert to enable pushing images to harbor
   * $ cd /etc/docker
   * $ mkdir certs.d
   * $ cd certs.d
   * $ mkdir harbor.your_domain.com
   * $ cd harbor.your_domain.com

   __COMPLETE AFTER PKS PIPELINE__
   * Login to opsman and download cert
    * OpsMan->Settings(click on user name)->Advanced->Download root CA
    * Copy contents of downloaded file
    * past copied content and save file ca.crt
   * $ systemctl stop docker
   * $ systemctl start docker
   * Add entry in /etc/hosts for harbor
      * 10.x.x.x harbor.your_domain.com
   * Verify
    * $ docker login harbor.your_domain.com

## Optional Bosh Setup (used for trouble-shooting bosh) ##

1. Install bosh
  * $ cd /pks_install/binaries
  * $ wget https://s3.amazonaws.com/bosh-cli-artifacts/bosh-cli-2.0.48-linux-amd64
  * $ chmod +x bosh-cli-x.x.x-linux-amd64
  * $ mv bosh-cli-x.x.x-linux-amd64 /usr/local/bin
  * $ bosh -v
  * $ cd /pks_install

  __Complete after PKS Pipeline__
  * Get bosh secret
    OpsMan->VMware Vsphere Tile->Credentials->Bosh Commandline Credentials->Link to Credential
    Copy secret value
  * Login to opsman and download cert (If haven't done already)
    OpsMan->Settings(click on user name)->Advanced->Download root CA
    Copy contents of downloaded file
    Save cert to file on linux box
  * Create bosh env file
    vi bosh.env
      export BOSH_ENVIRONMENT=ip_address_for_bosh
      export BOSH_CLIENT=ops_manager
      export BOSH_CLIENT_SECRET= secret_value_from_above
      export BOSH_CA_CERT=/etc/docker/certs.d/harbor.your_domain.com/ or other file where cert is saved
  * source bosh.env
  * Commands to verify setup
     * $ bosh env
     * $ bosh vms

