


My Lab setup contains two servers. One control plane machine and one node to be used for running containerized workloads.

|Server Type | Server Hostname | Specs|
|------------|-----------------|-------|
|Bastion  |bastion.lab.example.com  | 16GB Ram, 8vcpus|
|Haproxy |haproxy.lab.example.com | 4GB Ram, 2vcpus|
|bootstrap |bootstrap.lab.example.com | 16GB Ram, 8vcpus|
|Master1  |master1.lab.example.com  | 32GB Ram, 16vcpus|
|Master2 |master2.example.com | 32GB Ram, 16vcpus|
|Master3 |master3.example.com | 32GB Ram, 16vcpus|

#### Step 1: Bastion Node Configuration

Operating System Details :

    [root@bastion ~]# cat /etc/redhat-release
    Red Hat Enterprise Linux release 8.6 (Ootpa)

Disk mount details : 

    [root@bastion ~]# lsblk
    NAME          MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sda             8:0    0  100G  0 disk
    ├─sda1          8:1    0  500M  0 part /boot/efi
    ├─sda2          8:2    0    1G  0 part /boot
    └─sda3          8:3    0 98.5G  0 part
      └─rhel-root 253:0    0 98.5G  0 lvm  /
    sdb             8:16   0 1020G  0 disk /ocpregistry
    sr0            11:0    1 1024M  0 rom
     
IP Details : 

    [root@bastion ~]# ip address 
    [root@bastion ~]# ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
        link/ether 00:0c:29:e8:2b:27 brd ff:ff:ff:ff:ff:ff
        altname enp11s0
        inet 10.0.0.200/24 brd 192.168.31.255 scope global noprefixroute ens192
           valid_lft forever preferred_lft forever

Configure YUM Repository Server : 
    
    [root@haproxy ~]# mount /dev/cdrom /mnt
    mount: /mnt: WARNING: device write-protected, mounted read-only.
    
    [root@haproxy ~]# cat /etc/yum.repos.d/abc.repo
    [AppStream]
    enabled = 0
    gpgcheck = 0
    baseurl = file:///mnt/AppStream
    name = app stream  
    [baseos]
    enabled = 0
    gpgcheck = 0
    baseurl = file:///mnt/BaseOS
    name = base

Installation Pre-requisites Packages :-

    [root@bastion ~]# yum install wget curl tree tmux vim git -y
    [root@bastion ~]# yum install nfs-utils -y

/ocpregistry directory will be shared by nfs server
    
    [root@bastion ~]# df -h /ocpregistry
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/sdb       1020G  7.2G 1013G   1% /ocpregistry
    
Make Entry in exports file.

    [root@bastion ~]# echo "/ocpregistry  *(rw,sync)" >> /etc/exports
    
    [root@bastion ~]# exportfs -arv
    exporting *:/ocpregistry
    
Start and Enable nfs service. 

    [root@bastion ~]# systemctl enable --now nfs-server
    Created symlink /etc/systemd/system/multi-user.target.wants/nfs-server.service → /usr/lib/systemd/system/nfs-server.service.

Check nfs share point.
    
    [root@bastion ~]# showmount -e
    Export list for bastion.lab.example.com:
    /ocpregistry *

change ownership. 
    
    [root@bastion ~]# chown nobody:nobody /ocpregistry/
       
#### Step 2: Download OpenShift Binaries

- https://developers.redhat.com/content-gateway/file/pub/openshift-v4/clients/mirror-registry/1.3.6/mirror-registry.tar.gz
- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.12.36/oc-mirror.tar.gz
- https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.12.36/openshift-client-linux-4.12.36.tar.gz
- https://mirror.openshift.com/pub/openshift-v4/clients/pipeline/latest/tkn-linux-amd64.tar.gz
- https://mirror.openshift.com/pub/openshift-v4/clients/operator-sdk/4.12.36/operator-sdk-v1.22.2-ocp-linux-x86_64.tar.gz
- https://developers.redhat.com/content-gateway/file/pub/openshift-v4/clients/helm/3.11.1/helm-linux-amd64
- https://console.redhat.com/openshift/downloads download pullsecret

Placing all binaries to /usr/bin directory. 

    [root@bastion ~]# ls
    anaconda-ks.cfg  kubectl                 oc         oc-mirror.tar.gz  openshift-client-linux-4.12.36.tar.gz  tkn-linux-amd64.tar.gz
    helm             mirror-registry.tar.gz  oc-mirror  opc               tkn                                    tkn-pac

move all binaries to the binary directory. 
    
    [root@bastion ~]# mv helm kubectl oc oc-mirror opc tkn tkn-pac  /usr/bin

Checking openshift client version. 
    
    [root@bastion ~]# oc version
    Client Version: 4.12.36
    Kustomize Version: v4.5.7
    
Checking helm version. 

    [root@bastion ~]# helm version
    version.BuildInfo{Version:"v3.11.1+6.el8", GitCommit:"66bfc44f827aea6eb8e001150914170ac0d49e2d", GitTreeState:"clean", GoVersion:"go1.18.9"}

oc-mirror version. 
    
    [root@bastion ~]# oc-mirror version
    Client Version: version.Info{Major:"", Minor:"", GitVersion:"4.12.0-202309181625.p0.g3ac49d9.assembly.stream-3ac49d9", GitCommit:"3ac49d9bd7c2193ede794e328dfa1142d7735f2e", GitTreeState:"clean", BuildDate:"2023-09-18T18:56:58Z", GoVersion:"go1.19.10 X:strictfipsruntime", Compiler:"gc", Platform:"linux/amd64"}
    
Creating openshift variables : 

    [root@bastion ~]# cat openshift-vars.sh
    export OCP_RELEASE=4.12.36
    export LOCAL_REGISTRY=bastion.lab.example.com:8443
    export LOCAL_REPOSITORY=ocp4/openshift4
    export PRODUCT_REPO=openshift-release-dev
    export RELEASE_NAME=ocp-release
    export ARCHITECTURE=x86_64
    export GODEBUG=x509ignoreCN=0
    export REMOVABLE_MEDIA_PATH=/ocpregistry/ocp-base-images


#### Step 3: Chrony NTP Configuration 

    [root@bastion ~]# yum install chrony -y
    [root@bastion ~]# cat /etc/chrony.conf
    # Use public servers from the pool.ntp.org project.
    # Please consider joining the pool (http://www.pool.ntp.org/join.html).
    server 10.0.0.200 iburst
    
    # Allow NTP client access from local network.
    #allow 192.168.0.0/16
    allow 10.0.0.0/24
    
    [root@bastion ~]# systemctl restart chronyd
    [root@bastion ~]# systemctl enable --now chronyd

    [root@bastion ~]# timedatectl
                   Local time: Fri 2023-10-13 09:45:18 IST
               Universal time: Fri 2023-10-13 04:15:18 UTC
                     RTC time: Fri 2023-10-13 04:15:18
                    Time zone: Asia/Kolkata (IST, +0530)
    System clock synchronized: yes
                  NTP service: active
              RTC in local TZ: no

#### Step 4: DNS Configuration 


    [root@bastion ~]# cat /etc/named.conf
    ...   
    options {
            listen-on port 53 { 10.0.0.200; };
            #listen-on-v6 port 53 { ::1; };
            directory       "/var/named";
            dump-file       "/var/named/data/cache_dump.db";
            statistics-file "/var/named/data/named_stats.txt";
            memstatistics-file "/var/named/data/named_mem_stats.txt";
            secroots-file   "/var/named/data/named.secroots";
            recursing-file  "/var/named/data/named.recursing";
            allow-query     { 10.0.0.0/24; };
    
    ...
    zone "lab.prodevans.com" IN {
        type master;
        file "forward.zone";
        allow-update { none; };
        };
        
        zone "0.0.10.in-addr.arpa" IN {
                type master;
                file "reverse.zone";
                allow-update { none; };
        };

    [root@bastion ~]# cat  /var/named/forward.zone
    $TTL 1D
    @       IN SOA  bastion.lab.prodevans.com. root.lab.prodevans.com. (
                                            0       ; serial
                                            1D      ; refresh
                                            1H      ; retry
                                            1W      ; expire
                                            3H )    ; minimum
    @               IN      NS     bastion.lab.prodevans.com.
    bastion         IN      A       10.0.0.200
    
    api             IN      A       10.0.0.200
    api-int         IN      A       10.0.0.200
    
    *.apps          IN      A       10.0.0.200
    
    haproxy         IN      A       10.0.0.190
    
    bootstrap       IN      A       10.0.0.201
    master1         IN      A       10.0.0.202
    master2         IN      A       10.0.0.203
    master3         IN      A       10.0.0.204

    [root@bastion ~]# cat  /var/named/reverse.zone
    $TTL 1D
    @       IN SOA  bastion.lab.prodevans.com. root.lab.prodevans.com. (
                                            0       ; serial
                                            1D      ; refresh
                                            1H      ; retry
                                            1W      ; expire
                                            3H )    ; minimum
    @       IN      NS      bastion.lab.prodevans.com.
    @       IN      PTR      lab.prodevans.com.
    bastion         IN      A       10.0.0.200
    api             IN      A       10.0.0.200
    api-int         IN      A       10.0.0.200
    haproxy         IN      A       10.0.0.190
    bootstrap       IN      A       10.0.0.201
    master1         IN      A       10.0.0.202
    master2         IN      A       10.0.0.203
    master3         IN      A       10.0.0.204
    
    200      IN     PTR     bastion.lab.prodevans.com.
    200      IN     PTR     api.lab.prodevans.com.
    200      IN     PTR     api-int.lab.prodevans.com.
    201      IN     PTR     bootstrap.lab.prodevans.com.
    190      IN     PTR     haproxy.lab.prodevans.com.
    202      IN     PTR     master1.lab.prodevans.com.
    203      IN     PTR     master2.lab.prodevans.com.
    204      IN     PTR     master3.lab.prodevans.com.
    
    [root@bastion ~]# chown root:named /var/named/forward.zone /var/named/reverse.zone
    [root@bastion ~]# ll /var/named/forward.zone /var/named/reverse.zone
    -rw-r--r--. 1 root named  812 Oct 12 23:21 /var/named/forward.zone
    -rw-r--r--. 1 root named 1210 Oct 12 23:23 /var/named/reverse.zone
    [root@bastion ~]# systemctl enable --now named    


#### Step 5: Quay Registry Configuration

Installation podman package. 
    
    [root@bastion ~]# yum install podman* skopeo -y

Create extract directory. 
    
    [root@bastion ~]# mkdir /ocpregistry/mirror-registry

Extract mirror-registry to the /ocpregistry/mirror-registry directory.
    
    [root@bastion ~]# tar xf mirror-registry.tar.gz  -C /ocpregistry/mirror-registry
   
Start and enable podman service. 
 
    [root@bastion ~]# systemctl enable --now podman.socket podman.service
    Created symlink /etc/systemd/system/sockets.target.wants/podman.socket → /usr/lib/systemd/system/podman.socket.
    Created symlink /etc/systemd/system/default.target.wants/podman.service → /usr/lib/systemd/system/podman.service.
 
Check active status of podman. 
   
    [root@bastion ~]# systemctl status podman.service
    ● podman.service - Podman API Service
       Loaded: loaded (/usr/lib/systemd/system/podman.service; enabled; vendor preset: disabled)
       Active: active (running) since Thu 2023-10-12 18:16:55 IST; 983ms ago
         Docs: man:podman-system-service(1)
     Main PID: 21768 (podman)
        Tasks: 9 (limit: 101036)
       Memory: 19.8M
       CGroup: /system.slice/podman.service
               └─21768 /usr/bin/podman --log-level=info system service
    
    Oct 12 18:16:55 bastion.lab.example.com systemd[1]: Starting Podman API Service...
    Oct 12 18:16:55 bastion.lab.example.com systemd[1]: Started Podman API Service.
    Oct 12 18:16:55 bastion.lab.example.com podman[21768]: time="2023-10-12T18:16:55+05:30" level=info msg="/usr/bin/podman filtering a>
    Oct 12 18:16:55 bastion.lab.example.com podman[21768]: time="2023-10-12T18:16:55+05:30" level=info msg="Not using native diff for o>
    Oct 12 18:16:55 bastion.lab.example.com podman[21768]: time="2023-10-12T18:16:55+05:30" level=info msg="Setting parallel job count >
    Oct 12 18:16:55 bastion.lab.example.com podman[21768]: time="2023-10-12T18:16:55+05:30" level=info msg="Using systemd socket activa>
    Oct 12 18:16:55 bastion.lab.example.com podman[21768]: time="2023-10-12T18:16:55+05:30" level=info msg="API service listening on \">
    lines 1-17/17 (END)

make the etc host entry.
    
    [root@bastion ~]# echo "192.168.31.21 bastion.lab.example.com" >> /etc/hosts
    
Install Quay Mirror Registry for private images. 

    [root@bastion mirror-registry]# ./mirror-registry install --quayHostname bastion.lab.example.com --initUser red$$$ --initPassword $$$$$$$ --quayRoot /ocpregistry/mirror-registry
    
       __   __
      /  \ /  \     ______   _    _     __   __   __
     / /\ / /\ \   /  __  \ | |  | |   /  \  \ \ / /
    / /  / /  \ \  | |  | | | |  | |  / /\ \  \   /
    \ \  \ \  / /  | |__| | | |__| | / ____ \  | |
     \ \/ \ \/ /   \_  ___/  \____/ /_/    \_\ |_|
      \__/ \__/      \ \__
                      \___\ by Red Hat
     Build, Store, and Distribute your Containers
    
    ...
    PLAY RECAP ***************************************************************************************************************************
    root@bastion.lab.example.com : ok=50   changed=30   unreachable=0    failed=0    skipped=17   rescued=0    ignored=0
    
    INFO[2023-10-12 18:24:50] Quay installed successfully, config data is stored in /ocpregistry/mirror-registry
    INFO[2023-10-12 18:24:50] Quay is available at https://bastion.lab.example.com:8443 with credentials (red$$$, $$$$$$$)
       
Placing pem certificate. 
     
    [root@bastion ~]# cp /ocpregistry/mirror-registry/quay-rootCA/rootCA.pem /etc/pki/ca-trust/source/anchors/

reload certificate. 

    [root@bastion ~]# update-ca-trust

Podman login registry. 

    [root@bastion ~]# podman login https://bastion.lab.example.com:8443
    Username: 
    Password:
    Login Succeeded!

    [root@bastion ~]# ls
    anaconda-ks.cfg  materials  pull-secret.txt

Extract podman secret file in json format.
    
    [root@bastion ~]# podman login https://bastion.lab.example.com:8443 --authfile quay-secret.json
    Username: 
    Password:
    Login Succeeded!
    
    [root@bastion ~]# podman login https://bastion.lab.example.com:8443 --authfile pull-secret.txt
    Username: 
    Password:
    Login Succeeded!
    
    [root@bastion ~]# ls
    anaconda-ks.cfg  materials  pull-secret.txt  quay-secret.json

#### step 6: Create dockerconfiguration creds.

Generating credentials for download operators.

    [root@bastion ~]# mkdir .docker    
    [root@bastion ~]# cp pull-secret.txt .docker/config.json


#### step 7: Getting Openshift Variables

modify openshift-vars.sh. add pull secret variables. 

    [root@bastion ~]# cat openshift-vars.sh
    export OCP_RELEASE=4.12.36
    export LOCAL_REGISTRY=bastion.lab.example.com:8443
    export LOCAL_REPOSITORY=ocp4/openshift4
    export LOCAL_SECRET_JSON=/root/quay-secret.json
    export REDHAT_SECRET_JSON=/root/pullsecret.txt
    export PRODUCT_REPO=openshift-release-dev
    export RELEASE_NAME=ocp-release
    export ARCHITECTURE=x86_64
    export GODEBUG=x509ignoreCN=0
    export REMOVABLE_MEDIA_PATH=/ocpregistry/ocp-base-images

Source openshift variable. 
    
    [root@bastion ~]# source openshift-vars.sh

Download ocp base images for ocp4 images. 

    [root@bastion ~]# oc adm release mirror -a ${REDHAT_SECRET_JSON} --to-dir=${REMOVABLE_MEDIA_PATH}/mirror quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE}
  
    info: Mirroring completed in 4m4.92s (70.08MB/s)  
    Success
    Update image:  openshift/release:4.12.36-x86_64
    To upload local images to a registry, run:
    
        oc image mirror --from-dir=/ocpregistry/ocp-base-images/mirror 'file://openshift/release:4.12.36-x86_64*' REGISTRY/REPOSITORY
    
    Configmap signature file /ocpregistry/ocp-base-images/mirror/config/signature-sha256-38ccab25d5895a21.json created

Push ocp base images to the local quay registry. 

    [root@bastion ~]# oc image mirror -a ${LOCAL_SECRET_JSON} --from-dir=${REMOVABLE_MEDIA_PATH}/mirror "file://openshift/release:${OCP_RELEASE}*" ${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}

Extract openshift-install command for create manifests. 
   
    [root@bastion ~]# oc adm release extract -a ${REDHAT_SECRET_JSON} --command=openshift-install "${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}"

Checking openshift version release. 

    [root@bastion ~]# oc adm release info -a ${REDHAT_SECRET_JSON} "quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE}" | head -n 18
    Name:           4.12.36
    Digest:         sha256:38ccab25d5895a216a465a9f297541fbbebe7aa115fdaa9f2015c8d5a5d036eb
    Created:        2023-09-28T09:53:41Z
    OS/Arch:        linux/amd64
    Manifests:      648
    Metadata files: 1
    
    Pull From: quay.io/openshift-release-dev/ocp-release@sha256:38ccab25d5895a216a465a9f297541fbbebe7aa115fdaa9f2015c8d5a5d036eb
    
    Release Metadata:
      Version:  4.12.36
      Upgrades: 4.11.11, 4.11.12, 4.11.13, 4.11.14, 4.11.16, 4.11.17, 4.11.18, 4.11.19, 4.11.20, 4.11.21, 4.11.22, 4.11.23, 4.11.24, 4.11.25, 4.11.26, 4.11.27, 4.11.28, 4.11.29, 4.11.30, 4.11.31, 4.11.32, 4.11.33, 4.11.34, 4.11.35, 4.11.36, 4.11.37, 4.11.38, 4.11.39, 4.11.40, 4.11.41, 4.11.42, 4.11.43, 4.11.44, 4.11.45, 4.11.46, 4.11.47, 4.11.48, 4.11.49, 4.11.50, 4.12.0, 4.12.1, 4.12.2, 4.12.3, 4.12.4, 4.12.5, 4.12.6, 4.12.7, 4.12.8, 4.12.9, 4.12.10, 4.12.11, 4.12.12, 4.12.13, 4.12.14, 4.12.15, 4.12.16, 4.12.17, 4.12.18, 4.12.19, 4.12.20, 4.12.21, 4.12.22, 4.12.23, 4.12.24, 4.12.25, 4.12.26, 4.12.27, 4.12.28, 4.12.29, 4.12.30, 4.12.31, 4.12.32, 4.12.33, 4.12.34, 4.12.35
      Metadata:
        url: https://access.redhat.com/errata/RHSA-2023:5390
    
    Component Versions:
      kubernetes 1.25.12
      machine-os 412.86.202309200923-0 Red Hat Enterprise Linux CoreOS

RedHat Operators Download

|Operator | Resource Hub | version|
|------------|-----------------|-------|
|submariner |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable-0.15|
|mcg-operator |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable-4.12|
|kiali-ossm |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable|
|ocs-client-operator |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable-4.12|
|ocs-operator |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable-4.12|
|odf-csi-addons-operator |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable-4.12|
|quay-operator |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable-3.9|
|openshift-pipelines-operator-rh |registry.redhat.io/redhat/redhat-operator-index:v4.12  | latest|
|web-terminal |registry.redhat.io/redhat/redhat-operator-index:v4.12  | fast|
|vertical-pod-autoscaler |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable|
|file-integrity-operator |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable|
|local-storage-operator |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable|
|advanced-cluster-management |registry.redhat.io/redhat/redhat-operator-index:v4.12  | release-2.8|
|compliance-operator |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable|
|cluster-logging |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable|
|elasticsearch-operator |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable|
|rhacs-operator |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable|
|openshift-gitops-operator |registry.redhat.io/redhat/redhat-operator-index:v4.12  | latest|
|odf-operator |registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable-4.12|
|quay-bridge-operator|registry.redhat.io/redhat/redhat-operator-index:v4.12  | stable-3.9|

Now, Login to redhat registries 

    [root@bastion ocp-operators]# podman login registry.redhat.io --username username --password password
    Login Succeeded!    
    [root@bastion ocp-operators]# podman login registry.access.redhat.com --username username --password password
    Login Succeeded!

Create imageset-config.yaml file. 

    [root@bastion ocp-operators]# ls
    imageset-config.yaml
     
    [root@bastion ~]# cat imageset-config.yaml
    kind: ImageSetConfiguration
    apiVersion: mirror.openshift.io/v1alpha2
    archiveSize: 4
    storageConfig:
      local:
        path: iaf-egov
    mirror:
      platform:
        channels:
        - name: stable-4.12
          type: ocp
        graph: true
      operators:
      - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.12
        packages:
        - name: submariner
          channels:
          - name: stable-0.15
        - name: mcg-operator
          channels:
          - name: stable-4.12
        - name: kiali-ossm
          channels:
          - name: stable
        - name: ocs-client-operator
          channels:
          - name: stable-4.12
        - name: ocs-operator
          channels:
          - name: stable-4.12
        - name: odf-csi-addons-operator
          channels:
          - name: stable-4.12
        - name: quay-operator
          channels:
          - name: stable-3.9
        - name: openshift-pipelines-operator-rh
          channels:
          - name: latest
        - name: web-terminal
          channels:
          - name: fast
        - name: vertical-pod-autoscaler
          channels:
          - name: stable
        - name: file-integrity-operator
          channels:
          - name: stable
            minVersion: v1.0.0
            maxVersion: v1.3.2
        - name: local-storage-operator
          channels:
          - name: stable
        - name: advanced-cluster-management
          channels:
          - name: release-2.8
            minVersion: v2.7.8
            maxVersion: v2.8.2
        - name: compliance-operator
          channels:
          - name: stable
        - name: cluster-logging
          channels:
          - name: stable
            minVersion: v5.6.11
            maxVersion: v5.7.6
        - name: elasticsearch-operator
          channels:
          - name: stable
        - name: rhacs-operator
          channels:
          - name: stable
        - name: openshift-gitops-operator
          channels:
          - name: latest
        - name: odf-operator
          channels:
          - name: stable-4.12
        - name: quay-bridge-operator
          channels:
          - name: stable-3.9
    

Download redhat operator local Directory. 
   
    [root@bastion ocp-operators]# oc-mirror --config=imageset-config.yaml file://iaf-egov
    Creating directory: iaf-egov/oc-mirror-workspace/src/publish
    Creating directory: iaf-egov/oc-mirror-workspace/src/v2
    Creating directory: iaf-egov/oc-mirror-workspace/src/charts
    Creating directory: iaf-egov/oc-mirror-workspace/src/release-signatures
    No metadata detected, creating new workspace
    sha256:ee3e05958d0dc1972fd16811a44aac4e3aa34b527bcdeccff3bc0de0c21d523b file://rhacm2/submariner-gateway-rhel8
    sha256:d5794a736db55ccce2e4bb130ff7fbdd92f1c4d568abff6dceda151e355aeee4 file://rhacm2/submariner-gateway-rhel8
    sha256:a9fcfea9a1fe733a5feddb6bdae201f892b46239bffe6dd3d1d29d19c31afa18 file://rhacm2/submariner-gateway-rhel8
    sha256:cd51929654558ef6df73de9428896a232c2c2e68695a3f15627174e9972d23c6 file://rhacm2/submariner-gateway-rhel8:6b5f4f8f
    info: Mirroring completed in 18m46.31s (69.71MB/s)
    Creating archive /ocpregistry/ocp-operators/iaf-egov/mirror_seq1_000000.tar
    Creating archive /ocpregistry/ocp-operators/iaf-egov/mirror_seq1_000001.tar
    Creating archive /ocpregistry/ocp-operators/iaf-egov/mirror_seq1_000002.tar

    [root@bastion ocp-operators]# pwd
    /ocpregistry/ocp-operators

Tree view structure
        
    [root@bastion ocp-operators]# tree iaf-egov/
    iaf-egov/
    ├── mirror_seq1_000000.tar
    ├── mirror_seq1_000001.tar
    ├── mirror_seq1_000002.tar
    ├── mirror_seq1_000003.tar
    ├── mirror_seq1_000004.tar
    ├── mirror_seq1_000005.tar
    ├── mirror_seq1_000006.tar
    ├── mirror_seq1_000007.tar
    ├── mirror_seq1_000008.tar
    ├── mirror_seq1_000009.tar
    ├── mirror_seq1_000010.tar
    ├── mirror_seq1_000011.tar
    ├── mirror_seq1_000012.tar
    ├── mirror_seq1_000013.tar
    ├── mirror_seq1_000014.tar
    ├── mirror_seq1_000015.tar
    ├── mirror_seq1_000016.tar
    ├── mirror_seq1_000017.tar
    ├── mirror_seq1_000018.tar
    ├── mirror_seq1_000019.tar
    ├── oc-mirror-workspace
    └── publish
    2 directories, 20 files
    
Let's push the operators to the quay mirror registry. 
    
    [root@bastion ocp-operators]# oc-mirror --from=/ocpregistry/ocp-operators/iaf-egov docker://bastion.lab.example.com:8443
    Checking push permissions for bastion.lab.example.com:8443
    Publishing image set from archive "/ocpregistry/ocp-operators/iaf-egov" to registry "bastion.lab.example.com:8443"
    ...
    info: Mirroring completed in 2.75s (34.94MB/s)
    Wrote release signatures to oc-mirror-workspace/results-1697119734
    Rendering catalog image "bastion.lab.example.com:8443/redhat/redhat-operator-index:v4.12" with file-based catalog
    Writing image mapping to oc-mirror-workspace/results-1697119734/mapping.txt
    Writing UpdateService manifests to oc-mirror-workspace/results-1697119734
    Writing CatalogSource manifests to oc-mirror-workspace/results-1697119734
    Writing ICSP manifests to oc-mirror-workspace/results-1697119734
        
    [root@bastion ocp-operators]# tree oc-mirror-workspace/
    oc-mirror-workspace/
    ├── publish
    └── results-1697119734
        ├── catalogSource-redhat-operator-index.yaml
        ├── charts
        ├── imageContentSourcePolicy.yaml
        ├── mapping.txt
        ├── release-signatures
        │   └── signature-sha256-38ccab25d5895a21.json
        └── updateService.yaml

Creating openshift installation directory. 

    [root@bastion ~]# mkdir ocp4

Create install-config.yaml for installation process. 

    [root@bastion ~]# cat install-config.yaml
    apiVersion: v1
    baseDomain: example.com
    compute:
      - hyperthreading: Enabled
        name: worker
        replicas: 0
    controlPlane:
      hyperthreading: Enabled
      name: master
      replicas: 3
    metadata:
      name: lab
    networking:
      clusterNetwork:
        - cidr: 10.128.0.0/14
          hostPrefix: 23
      networkType: OpenShiftSDN
      serviceNetwork:
        - 172.30.0.0/16
    platform:
      none: {}
    fips: false
    pullSecret: '{"auths":{"bastion.lab.example.com:8443":{"auth":"cmVka<REDACTED>AxMjM="}}}'
    sshKey: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQ<REDACTED>7fytpmWUqV2UkjdkNUkSM= root@bastion.lab.example.com"
    additionalTrustBundle: |
      -----BEGIN CERTIFICATE-----
      MIID0jCCArqgAwIBAgIURgJAqMzNcHKGd5/pjJn0vjTMYYQwDQYJKoZIhvcNAQEL
      BQAwczELMAkGA1UEBhMCVVMxCzAJBgNVBAgMAlZBMREwDwYDVQQHDAhOZXcgWW9y
      azENMAsGA1UECgwEUXVheTERMA8GA1UECwwIRGl2aXNpb24xIjAgBgNVBAMMGWJh
      <REDACTED>
      InarmfrGVCxjWs60dR/0tdkB07fodmVXBqPYEJRIsQPY4lMkksUqjz9HwIrRvuQM
      k2oQnPCteSehR9i8UHSs8fA3+2PbmNJyU0uT41GAW9dgXKlRiozrIVXG7qVnkPPF
      BPbg0mdFoxJ9OjMk2V1qNTMlqosPNQ==
      -----END CERTIFICATE-----
    imageContentSources:
    - mirrors:
      - bastion.lab.example.com:8443/ocp4/openshift4
      source: quay.io/openshift-release-dev/ocp-release
    - mirrors:
      - bastion.lab.example.com:8443/ocp4/openshift4
      source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
        
    [root@bastion ~]# cp install-config.yaml ocp4
    
    [root@bastion ~]# ./openshift-install create manifests --dir ocp
    INFO Consuming Install Config from target directory
    WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings
    INFO Manifests created in: ocp4/manifests and ocp/openshift
    
    [root@bastion ~]# ./openshift-install create ignition-configs --dir ocp4
    INFO Consuming Common Manifests from target directory
    INFO Consuming Worker Machines from target directory
    INFO Consuming Master Machines from target directory
    INFO Consuming Openshift Manifests from target directory
    INFO Consuming OpenShift Install (Manifests) from target directory
    INFO Ignition-Configs created in: ocp4 and ocp4/auth
    
    [root@bastion ~]# mkdir /var/www/html/ocp
    [root@bastion ~]# cp -rf ocp/*.ign  /var/www/html/ocp
    [root@bastion ~]# chown apache:apache -R /var/www/html/ocp
    [root@bastion ~]# chmod 777 /var/www/html/ocp -R
    
    [root@bastion ~]# systemctl restart httpd
    [root@bastion ~]# systemctl status httpd
    ● httpd.service - The Apache HTTP Server
       Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
       Active: active (running) since Fri 2023-10-13 09:35:58 IST; 1s ago
         Docs: man:httpd.service(8)
     Main PID: 22787 (httpd)
       Status: "Started, listening on: port 82"
        Tasks: 213 (limit: 100500)
       Memory: 37.3M
       CGroup: /system.slice/httpd.service
               ├─22787 /usr/sbin/httpd -DFOREGROUND
               ├─22790 /usr/sbin/httpd -DFOREGROUND
               ├─22791 /usr/sbin/httpd -DFOREGROUND
               ├─22792 /usr/sbin/httpd -DFOREGROUND
               └─22793 /usr/sbin/httpd -DFOREGROUND

