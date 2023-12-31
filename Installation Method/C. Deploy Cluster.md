
# Deploy OpenShift

1. Power on the ocp-bootstrap host and ocp-cp-\# hosts and select 'Tab' to enter boot configuration. Enter the following configuration:

       [root@bastion ~]# sudo coreos-installer install /dev/sda --copy-network --ignition-url http://10.0.0.200/ocp4/bootstrap.ign --insecure --insecure-ignition 
       
2. Power on the master-\# hosts and select 'Tab' to enter boot configuration. Enter the following configuration:
  
If you waited for it boot, use the following command then just reboot after it finishes and make sure you remove the attached .iso

       [root@bastion ~]# sudo coreos-installer install /dev/sda --copy-network --ignition-url http://10.0.0.200/ocp4/master.ign --insecure --insecure-ignition 
     
3. Power on the worker-\# hosts and select 'Tab' to enter boot configuration. Enter the following configuration:

Or if you waited for it boot, use the following command then just reboot after it finishes and make sure you remove the attached .iso

       [root@bastion ~]# sudo coreos-installer install /dev/sda --copy-network --ignition-url http://10.0.0.200/ocp4/worker.ign --insecure --insecure-ignition
  

#### Monitor the Bootstrap Process

You can monitor the bootstrap process from the ocp-svc host at different log levels (debug, error, info)

       [root@bastion ~]# ./openshift-install --dir ~/ocp-install wait-for bootstrap-complete --log-level=debug
    

Once bootstrapping is complete the ocp-boostrap node [can be removed](#remove-the-bootstrap-node)

#### Remove the Bootstrap Node

Remove all references to the `bootstrap` host from the `/etc/haproxy/haproxy.cfg` file

       [root@bastion ~]# vim /etc/haproxy/haproxy.cfg

Restart HAProxy - If you are still watching HAProxy stats console you will see that the boostrap host has been removed from the backends.

       [root@bastion ~]# systemctl reload haproxy

The ocp-bootstrap host can now be safely shutdown and deleted from the VMware ESXi Console, the host is no longer required

#### Wait for installation to complete

> IMPORTANT: if you set mastersSchedulable to false the [worker nodes will need to be joined to the cluster](#join-worker-nodes) to complete the installation. This is because the OpenShift Router will need to be scheduled on the worker nodes and it is a dependency for cluster operators such as ingress, console and authentication.

Collect the OpenShift Console address and kubeadmin credentials from the output of the install-complete event

       [root@bastion ~]# ./openshift-install --dir ~/ocp-install wait-for install-complete
       [root@bastion ~]# ssh core@bootstrap
    

#### Configure NTP Synchronize for Master Node

Create a chrony configuration. 

       [root@bastion ~]# cat > chrony.conf
       server 10.0.0.200 iburst
       driftfile /var/lib/chrony/drift
       makestep 1.0 3
       rtcsync
       logdir /var/log/chrony

Encode base64.

       [root@bastion ~]# cat chrony.conf | base64 -w0
       c2VydmVyIDEwLjAuMC4yMDAgaWJ1cnN0CmRyaWZ0ZmlsZSAvdmFyL2xpYi9jaHJvbnkvZHJpZnQKbWFrZXN0ZXAgMS4wIDMKcnRjc3luYwpsb2dkaXIgL3Zhci9sb2cvY2hyb255Cg==

Getting machine config. 

       [root@bastion ~]# oc get mc 
       NAME                                               GENERATEDBYCONTROLLER                      IGNITIONVERSION   AGE
       00-master                                          7101fb0720d05771bdc174af918b64deb4efa604   3.2.0             17h
       00-worker                                          7101fb0720d05771bdc174af918b64deb4efa604   3.2.0             17h
       01-master-container-runtime                        7101fb0720d05771bdc174af918b64deb4efa604   3.2.0             17h
       01-master-kubelet                                  7101fb0720d05771bdc174af918b64deb4efa604   3.2.0             17h
       01-worker-container-runtime                        7101fb0720d05771bdc174af918b64deb4efa604   3.2.0             17h
       01-worker-kubelet                                  7101fb0720d05771bdc174af918b64deb4efa604   3.2.0             17h
       99-master-generated-registries                     7101fb0720d05771bdc174af918b64deb4efa604   3.2.0             17h
       99-master-ssh                                                                                 3.2.0             17h
       99-worker-generated-registries                     7101fb0720d05771bdc174af918b64deb4efa604   3.2.0             17h
       99-worker-ssh                                                                                 3.2.0             17h
       rendered-master-41e3ae91fdb6fb27a780cdd34021b796   7101fb0720d05771bdc174af918b64deb4efa604   3.2.0             17h
       rendered-master-519f25febe4b0f69586d664d30120bb0   7101fb0720d05771bdc174af918b64deb4efa604   3.2.0             9h
       rendered-worker-44a8cca273b3bee3276e36a806fd97c9   7101fb0720d05771bdc174af918b64deb4efa604   3.2.0             9h
       rendered-worker-6fc2fa6e72356a18924bbeb7afa6fc80   7101fb0720d05771bdc174af918b64deb4efa604   3.2.0             17h

Apply master configuration. 

       [root@bastion ~]# oc apply -f 99-master-chrony.conf 
       machineconfig.machineconfiguration.openshift.io/99-master-chrony created

After getting applied configuration. 

       [root@bastion ~]# oc get mc 99-master-chrony
       NAME               GENERATEDBYCONTROLLER   IGNITIONVERSION   AGE
       99-master-chrony                           3.1.0             9s

