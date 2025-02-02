## Bonus Lab: Deploy VM using Hostpath Storage

Execute `oc whoami` here just makes sure you're in the right place:

```execute-1
oc whoami
```

And see below:

~~~bash
system:serviceaccount:workbook:cnv
~~~

Now, how is this plugged on the underlying host? 

The key to this is the **"net1@if29"** device (it may be slightly different in your environment); this is one half of a **"veth pair"** that allows network traffic to be bridged between network namespaces, which is exactly how containers segregate their network traffic between each other on a container host. In this example the **"cnv-bridge"** is being used to connect the bridge for the virtual machine (**"k6t-net1"**) out to the bridge on the underlying host (**"br1"**), via a veth pair. The other side of the veth pair can be discovered as follows. First find the host of our virtual machine:

```execute-1
oc get vmi
```

This is your host:

~~~bash
NAME               AGE   PHASE     IP               NODENAME                       READY
rhel8-server-ocs   30h   Running   192.168.123.64   ocp4-worker3.%node-network-domain%   True
~~~

Then connect to it and track back the link - here you'll need to adjust the commands below - if your veth pair on the pod side was **"net1@if29"** then the **ifindex** in the command below will be **"29"**, if it was **"net1@if5"** then **"ifindex"** will be **"5"** and so on...

To do this we need to get to the worker running our virtual machine, and we can use the `oc debug node` function to do this (adjust to suit the host of your virtual machine from the previous command):

```execute-1
oc debug node/ocp4-worker3.%node-network-domain%
```

This will open a debug pod:

~~~bash
Starting pod/ocp4-worker3aioexamplecom-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.123.106
If you don't see a command prompt, try pressing enter.
~~~

```execute-1
chroot /host
```

Then execute

```execute-1
export ifindex=29
```

After that check netns


```execute-1
ip -o link | grep ^$ifindex: | sed -n -e 's/.*\(veth[[:alnum:]]*@if[[:digit:]]*\).*/\1/p'
```

Now you should get an error as followŞ

~~~bash
Error: Peer netns reference is invalid.
veth9d567769@if4
~~~

Therefore, the other side of the link, in the example above is **"veth9d567769@if4"**. 

```copy
ip link show veth9d567769  
```

You can then see that this is attached to **"br1"** as follows:

~~~bash
29: veth9d567769@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br1 state UP mode DEFAULT group default 
Error: Peer netns reference is invalid.
    link/ether 16:6b:ba:90:da:87 brd ff:ff:ff:ff:ff:ff link-netns c3dde5db-32f3-47a7-b7a8-3705dd17f8e1
~~~

Note the "**master br1**" in the above output.

Or visually represented:

<img src="img/veth-pair.png" />

Exit the debug shell(s) before proceeding:

Exit the shell before proceeding:

```execute-1
exit
```

```execute-1
exit
```

Execute `oc whoami` here just makes sure you're in the right place:

```execute-1
oc whoami
```


Now ensure you are in the default project:

```execute-1
oc project default
```

~~~ bash
Already on project "default" on server "https://172.30.0.1:443".
~~~

Now that we have the OCS instance running, let's do the same for the **hostpath** setup we created. Let's leverage the hostpath PVC that we created in a previous step - this is essentially the same as our OCS-based VM instance, except we reference the `rhel8-hostpath` PVC instead:


```execute-1
cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  annotations:
    name.os.template.kubevirt.io/rhel8: Red Hat Enterprise Linux 8.0
  labels:
    app: rhel8-server-hostpath
    kubevirt.io/os: rhel8
    os.template.kubevirt.io/rhel8: 'true'
    template.kubevirt.ui: openshift_rhel8-generic-small
    vm.kubevirt.io/template: openshift_rhel8-generic-small
    workload.template.kubevirt.io/generic: 'true'
  name: rhel8-server-hostpath
spec:
  running: true
  template:
    metadata:
      labels:
        vm.kubevirt.io/name: rhel8-server-hostpath
    spec:
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          disks:
          - disk:
              bus: sata
            name: rhel8-hostpath
          interfaces:
          - bridge: {}
            model: e1000
            name: tuning-bridge-fixed
        firmware:
          uuid: 5d307ca9-b3ef-428c-8861-06e72d69f223
        machine:
          type: q35
        resources:
          requests:
            memory: 512M
      evictionStrategy: LiveMigrate
      networks:
        - multus:
            networkName: tuning-bridge-fixed
          name: tuning-bridge-fixed
      terminationGracePeriodSeconds: 0
      volumes:
      - name: rhel8-hostpath
        persistentVolumeClaim:
          claimName: rhel8-hostpath
EOF
```

Check the VirtualMachine object is created:

~~~bash
virtualmachine.kubevirt.io/rhel8-server-hostpath created 
~~~

As before we can see the launcher pod built and run:

```execute-1
oc get pods
```

You will see:

~~~bash
NAME                                        READY   STATUS    RESTARTS   AGE
virt-launcher-rhel8-server-hostpath-ddmjm   0/1     Pending   0          4s
virt-launcher-rhel8-server-ocs-z5rmr        1/1     Running   0          30h
~~~

Now check the VMIs:

```execute-1
oc get vmi
```

And after a few seconds, this should launch...
~~~bash
NAME                    AGE   PHASE     IP               NODENAME                       READY
rhel8-server-hostpath   71s   Running   192.168.123.65   ocp4-worker2.%node-network-domain%   True
rhel8-server-ocs        30h   Running   192.168.123.64   ocp4-worker3.%node-network-domain%   True
~~~

Now look deeper:

```copy
oc describe pod/virt-launcher-rhel8-server-hostpath-9sqxm
```

And looking deeper we can see the hostpath claim we explored earlier being utilised, note the `Mounts` section for where, inside the pod, the `rhel8-hostpath` PVC is attached, and then below the PVC name:


~~~yaml
Name:         virt-launcher-rhel8-server-hostpath-9sqxm
Namespace:    default
Priority:     0
Node:         ocp4-worker2.%node-network-domain%/192.168.123.105
Start Time:   Tue, 09 Nov 2021 20:33:22 +0000
Labels:       kubevirt.io=virt-launcher
              kubevirt.io/created-by=7a19cb48-18bb-4d16-a7eb-4f7811675c17
              vm.kubevirt.io/name=rhel8-server-hostpath
(...)    
    Mounts:
      /var/run/kubevirt from public (rw)
      /var/run/kubevirt-ephemeral-disks from ephemeral-disks (rw)
      /var/run/kubevirt-private from private (rw)
      /var/run/kubevirt-private/vmi-disks/rhel8-hostpath from rhel8-hostpath (rw)
      /var/run/kubevirt/container-disks from container-disks (rw)
      /var/run/kubevirt/hotplug-disks from hotplug-disks (rw)
      /var/run/kubevirt/sockets from sockets (rw)
      /var/run/libvirt from libvirt-runtime (rw)
(...)
Volumes:
  rhel8-hostpath:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  rhel8-hostpath
    ReadOnly:   false
(...)
~~~

If we take a peek inside of that pod we can dig down further, you'll notice that it's largely the same output as the OCS step above.

Open a bash shell in the pod:

```copy
oc exec -it virt-launcher-rhel8-server-hostpath-9sqxm bash
```

Then execute:


```execute-1
virsh domblklist default_rhel8-server-hostpath
```

But the mount point is obviously not over OCS:

~~~bash
 Target   Source
-----------------------------------------------------------------------
 sda      /var/run/kubevirt-private/vmi-disks/rhel8-hostpath/disk.img
~~~

The problem here is that if you try and run the `mount` command to see how this is attached:

```execute-1
mount | grep rhel8
```
It will only show the whole filesystem being mounted into the pod:

~~~bash
/dev/vda4 on /run/kubevirt-private/vmi-disks/rhel8-hostpath type xfs (rw,relatime,seclabel,attr2,inode64,logbufs=8,logbsize=32k,prjquota)
~~~

However, we can see how this has been mapped in by Kubernetes by looking at the pod configuration on the host. 

Now exit from the pod:

```execute-1
exit
```
~~~bash
logout
~~~

First, recall which host your virtual machine is running on:

```execute-1
oc get vmi/rhel8-server-hostpath
```

~~~bash
NAME                    AGE     PHASE     IP               NODENAME                       READY
rhel8-server-hostpath   6m43s   Running   192.168.123.65   ocp4-worker2.%node-network-domain%   True
~~~

And get the name of the launcher pod:

```execute-1
oc get pods | awk '/hostpath/ {print $1;}'
```

This is the launcher pod:

~~~bash
virt-launcher-rhel8-server-hostpath-9sqxm
~~~

> **NOTE**: In your environment, your virtual machine may be running on worker1, simply adjust the following instructions to suit your configuration.

Next, we need to get the container ID from the pod:

```copy
oc describe pod virt-launcher-rhel8-server-hostpath-9sqxm | awk -F// '/Container ID/ {print $2;}'
```

You should see something similar below:

~~~bash
ba06cc69e0dbe376dfa0c8c72f0ab5513f31ab9a7803dd0102e858c94df55744
~~~

> **NOTE**: Make a note (copy it) of this container ID as we'll use it in the next step

Now we can check on the worker node itself, remembering to adjust these commands for the worker that your hostpath based VM is running on, and the container ID from the above:

```copy
oc describe pod virt-launcher-rhel8-server-hostpath-9sqxm | grep "Node:"
```

Then you should see the worker node that the pod is running on:

~~~bash
Node:         ocp4-worker2.%node-network-domain%/192.168.123.105
~~~


Open a debug pod:


This will open a debug pod:


```execute-1
oc debug node/ocp4-worker2.%node-network-domain%
```

~~~bash
Starting pod/ocp4-worker2aioexamplecom-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.123.105
If you don't see a command prompt, try pressing enter.
~~~

```execute-1
chroot /host
```

Then inspect the container by using the container ID from the previous step: 

```copy
crictl inspect ba06cc69e0dbe376dfa0c8c72f0ab5513f31ab9a7803dd0102e858c94df55744 | grep -A4 rhel8-hostpath
```

Examine the *hostPath*:

~~~json
        "containerPath": "/var/run/kubevirt-private/vmi-disks/rhel8-hostpath",
        "hostPath": "/var/hpvolumes/pvc-0f3ff50d-12b1-4ad6-8beb-be742a6e674a",
        "propagation": "PROPAGATION_PRIVATE",
        "readonly": false,
        "selinuxRelabel": false
(...)
~~~

Here you can see that the container has been configured to have a `hostPath` from `/var/hpvolumes` mapped to the expected path inside of the container where the libvirt definition is pointing to. Don't forget to exit (twice) before proceeding:

Exit the debug shell(s) before proceeding:

Exit the shell before proceeding:

```execute-1
exit
```

```execute-1
exit
```

Execute `oc whoami` here just makes sure you're in the right place:

```execute-1
oc whoami
```
~~~bash
system:serviceaccount:workbook:cnv
~~~

Then ensure you are in the default project:

```execute-1
oc project default
```

~~~bash
Already on project "default" on server "https://172.30.0.1:443".
~~~

That's it for deploying basic workloads - we've deployed a VM on-top of OCS and one on-top of hostpath.

Before moving to next lab let's remove the hostpath-based VM:

```execute-1
oc delete vm/rhel8-server-hostpath
```

And wait for VM is deleted:

~~~bash
virtualmachine.kubevirt.io "rhel8-server-hostpath" deleted
~~~

And delete PVC for hostpath:


```execute-1
oc delete pvc rhel8-hostpath
```
