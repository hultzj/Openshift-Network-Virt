# Openshift-Network-Virt
How to guide on openshift virtualization and connecting to external networks directly to VMI.
### Prerequisites:
  - Configure the physical nodes to have a secondary network interface that is appendable with access to the subnet the user wishes to access directly.
  - Define a VLAN to append to otherwise the openshift will default to whichever vlan is on the network (normally vlan1)

## Create Openshift managed network interfaces on nodes

1. Create interface yaml
```
apiVersion: nmstate.io/v1beta1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-eno2-policy
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: br1 (desired bridge name)
        description: Linux bridge with eth1 as a port
        type: linux-bridge
        state: up
        ipv4:
          dhcp: true
          enabled: true
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: eno2 (1 or more names of the physical network interfaces)
            - name: eno3
```
2. Run oc apply -f

## Create Openshift network attachment
Openshift network attachements are project based and will bond to the physical bridge created in step 1 and allow for the VM's to be attached. This can be done either via template or under the project on the console.  

```
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: external-network
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/<bridge-interface>
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "br1",
    "plugins": [
      {
        "type": "cnv-bridge",
        "bridge": "br1",
        "vlan": xxx
      },
      {
        "type": "cnv-tuning"
      }
    ]
  }'

```
## Attaching network interface
On the console naviagate to virtualization>virtual_machines>network Interfaces and add an interface.
- Define Name, for model all linux should be virtio (windows can be e1000e), network is the new attachment and type is bridge
- VM restart is required for changes.
- This can also be done via yaml template

## VM modifications
On the VM verify the 2nd nic has been attached and then use nmtui or nmcli to create the configuration for the nic. On eth0 the default nic change the DEFROUTE to no. This is required if there exists multiple reachable subnets on the external network, because eth0 on the VM will default to the pod network otherwise and lock access to just the single subnet set in the secondary nic. 


## Documenation
- https://docs.openshift.com/container-platform/4.8/virt/about-virt.html
- Creating node interfaces: https://docs.openshift.com/container-platform/4.8/virt/node_network/virt-updating-node-network-config.html#virt-about-nmstate_virt-updating-node-network-config
- Network attachements: https://docs.openshift.com/container-platform/4.8/virt/virtual_machines/vm_networking/virt-attaching-vm-multiple-networks.html
