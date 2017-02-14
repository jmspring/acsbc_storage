# RethinkDB Storage Demo on DCOS

This demo will demonstrate running RethinkDB on a DCOS cluster on ACS.  The demo is fairly basic because DCOS does not yet have the level of Persistent Volume support that Kubernetes does.  There are other solutions including network filesystems like GlusterFS or potentially using Flocker and the Azure Flocker Driver.  That said, ClusterHQ, the maintainers of Flocker, have ceased being in business.  The Mesosphere preferred solution is to use the RexRay driver framework.  No RexRay driver exists yet for Azure.

In this demo we will:

- Deploy an DCOS cluster using `acs-engine`
- Make use of Marathon Attributes to specify the agent that RethinkDB will run on
- Deploy RethinkDB to the specific agent
- Interact with RethinkDB
- Kill the agent and wait and see RethinkDB get redeployed to the same agent
- Show that data in RethinkDB was preserved

## Deploying the DCOS cluster

For this demo, the cluster is deployed using `acs-engine` and will come with pre-attached managed disks.  `acs-engine` takes a template and generates the approriate ARM templates to deploy the desired configuration.

The template for this deployment is as follows:

```bash
{
  "apiVersion": "vlabs",
  "properties": {
    "orchestratorProfile": {
      "orchestratorType": "DCOS"
    },
    "masterProfile": {
      "count": 1,
      "dnsPrefix": "jmsbcdcos",
      "vmSize": "Standard_D2_v2"
    },
    "agentPoolProfiles": [
      {
        "name": "agent128",
        "count": 3,
        "vmSize": "Standard_D3_v2",
        "availabilityProfile": "AvailabilitySet",
        "storageProfile": "ManagedDisks",
        "diskSizesGB": [20, 20]
      },
      {
        "name": "agent1public",
        "count": 1,
        "vmSize": "Standard_D2_v2",
        "dnsPrefix": "jmsbcdcospub",
        "availabilityProfile": "AvailabilitySet",
        "storageProfile": "ManagedDisks",
        "diskSizesGB": [1],
        "ports": [
          80,
          443,
          8080
        ]
      }
    ],
    "linuxProfile": {
      "adminUsername": "sadmin",
      "ssh": {
        "publicKeys": [
          {
            "keyData": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCkyxU5ERFK7Z8SujvxxOLHeR6pRi9LJy7+ju5lnxa+0xMDBIvvwCM4dgY1ccCinfmRV3ha5riGhLEFbS8mrFXWC+0umF+iS/9+swzvHXqUnpXiqy8E04BXxYNVrOjNw7XR6aaPuq4M2/IM8UW6ZBxOtd5qhY/93oS+g9pqU4mI5ZoyofPXB6WNN4QTLjfMiBqe/r4adCv/3oNkMOVYHXp7p1zQ3H0taZEFD3Koigh6Ggb1JyAtp692r2ZlDRBtJU/VSBjFudBQlQcn4qohUmh8FnL4N2t6AWffb0BGoH4T/SVZq7ME+F6cfUNxAYZhlXoulb5gBuAb882p+1Sq703S1l3WZagyQ45tcmOsM17gWIyQdV8IRSkodVpbhSIl/9SBHrBejeVK2EnYA+WGlPKxrPSHsH1cImkArQxbpj9qBGiBrTFQCOu1sDZeRTkP3RRzaZOquN8iB+i9gCwC1Olg3OBeZq8jQcxRT05kCd/iAlNIbFTHmB2zD5YwAnY8hdCMMZwpMVU9hv7LmSNRLE/QWyVfl/VLNxox1uMJ1F9VB7oPc8Y4TyzflFTWgnaARnIZRtp3mJ0aYL/RTXf/59hMa7gnY4ShVOFo/F+JRSWT3+0eceGxl1I+ZUBh4l5MmMTSCJTdYFrreIIWXlNiKwgxs/N4EZn6mM/NeNabgFnkPw== jims@azhat"
          }
        ]
      }
    }
  }
}
```

Basically, each agent node is configured with two pre-attached disks of 20Gig each.  The rest is pretty straight forward.

To generate the templates:

```bash
jims@azhat:~/src/github/acsbc_storage/dcos/setup$ acs-engine -artifacts out/ dcos-template.json 
wrote out/apimodel.json
wrote out/azuredeploy.json
wrote out/azuredeploy.parameters.json
acsengine took 22.051761ms
```

And to deploy the generated ARM templates:

```bash
jims@azhat:~/src/github/acsbc_storage/dcos/setup$ cd out/
jims@azhat:~/src/github/acsbc_storage/dcos/setup/out$ az group deployment create --resource-group=jmsacsbc --name=acsdcosdep --template-file ./azuredeploy.json --parameters @./azuredeploy.parameters.json 
{
  "id": "/subscriptions/04f7ec88-8e28-41ed-8537-5e17766001f5/resourceGroups/jmsbcdcosrg/providers/Microsoft.Resources/deployments/acsdcosdep",
  "name": "acsdcosdep",
  "properties": {
    "correlationId": "c74ed8e2-0945-4bac-b51a-9d60823dbc08",
    "debugSetting": null,
    "dependencies": [
...
    ],
    "provisioningState": "Succeeded",
    "template": null,
    "templateLink": null,
    "timestamp": "2017-02-12T18:48:05.248423+00:00"
  },
  "resourceGroup": "jmsbcdcosrg"
}
```

The whole output is truncated as it is quite long.  At this point, the DCOS cluster is deployed into resource group `jmsbcdcosrg`.

## Add Marathon Attribute to DCOS agent

In order to log onto one of the agent machines and set the appropriate attributes, we first need to gain access to the agent machines.  The best way to do this is to publish the ssh key onto the master node and then access agent machines from there.

To access the master node, we first need it's IP Address:

```bash
jims@azhat:~/src/github/acsbc_storage/dcos/setup/out$ az network public-ip list --resource-group=jmsbcdcosrg | jq -r '.[] | select(.name | contains("dcos-master")) | .ipAddress'
13.64.112.147
```

The master node has an IP Address of `13.64.112.147`.  Deployments configure SSH to start listening on port 2200 for the first master, 2201 for the second, etc.  We only have one master, so we push the SSH private key to it:

```bash
jims@azhat:~/src/github/acsbc_storage/dcos/setup/out$ scp -P2200 -i ~/.ssh/id_jmsk8s_rsa /home/jims/.ssh/id_jmsk8s_rsa sadmin@13.64.112.147:.ssh/
The authenticity of host '[13.64.112.147]:2200 ([13.64.112.147]:2200)' can't be established.
ECDSA key fingerprint is SHA256:K7YNdK4kjgPCaK3OptE7UlTMS9kGD9t4yvWAw57n4LI.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[13.64.112.147]:2200' (ECDSA) to the list of known hosts.
id_jmsk8s_rsa     
```

Next, let's log into the master and then the agent node we want to set our attribute on:

```bash
jims@azhat:~/src/github/acsbc_storage/dcos/setup/out$ ssh -p 2200 -i ~/.ssh/id_jmsk8s_rsa sadmin@13.64.112.147
Welcome to Ubuntu 16.04 LTS (GNU/Linux 4.4.0-28-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.


To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

sadmin@dcos-master-15497509-0:~$ ssh -i ~/.ssh/id_jmsk8s_rsa sadmin@10.0.0.6
ssh: /opt/mesosphere/lib/libcrypto.so.1.0.0: no version information available (required by ssh)
ssh: /opt/mesosphere/lib/libcrypto.so.1.0.0: no version information available (required by ssh)
The authenticity of host '10.0.0.6 (10.0.0.6)' can't be established.
ECDSA key fingerprint is SHA256:3n9aoTkSWABvueEKHatyIHqeWIKHJ7QDcfBAyACklKs.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.0.0.6' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 16.04 LTS (GNU/Linux 4.4.0-28-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

98 packages can be updated.
0 updates are security updates.


*** System restart required ***

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

sadmin@dcos-agent128-154975090:~$ 
```

Next, we need to add the the attribute `MESOS_ATTRIBUTES=role:db` to `/var/lib/dcos/mesos-slave-common` and reboot the agent node:

```bash
sadmin@dcos-agent128-154975090:~$ ls /var/lib/dcos/mesos-slave-common
ls: cannot access '/var/lib/dcos/mesos-slave-common': No such file or directory
sadmin@dcos-agent128-154975090:/var/lib/dcos$ sudo su
sadmin@dcos-agent128-154975090:~$ sudo echo "MESOS_ATTRIBUTES=role:db" > /var/lib/dcos/mesos-slave-common
root@dcos-agent128-154975090:/var/lib/dcos# echo "MESOS_ATTRIBUTES=role:db" > /var/lib/dcos/mesos-slave-common
root@dcos-agent128-154975090:/var/lib/dcos# reboot
```

If you look at the node in the DCOS UI, you will note that the ![screen for the node](https://github.com/jmspring/acsbc_storage/raw/master/dcos/images/dcos-agent-view.png) shows the attribute we just added.
