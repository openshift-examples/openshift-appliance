# OpenShift Appliance demo env.

All about OpenShift Appliance you can find [here](https://github.com/openshift/appliance/tree/master).

Important is the [user guide](https://github.com/openshift/appliance/blob/master/docs/user-guide.md)!

## High-Level Flow Overview

![Overview](https://github.com/openshift/appliance/raw/master/docs/images%2Fhl-overview.png)


## Let's start with the Lab!

*Requirements:*

* Red Hat Enterprise Linux 9 - might work with different version / operating system as well - but not tested!
* Libvirt running
* Enough CPU, RAM and Storage to host a single node openshift

Note: it works perfect on your [hetzner-ocp4](https://github.com/RedHat-EMEA-SSA-Team/hetzner-ocp4) box ;-)


Check out the git repo on a place with at least 200GB free storage!

```git clone ...```

### First, networking!

Configure libvirt network (air-gapped)

*Create openshift-appliance network:*
```
virsh net-define virsh-network.xml
virsh net-autostart openshift-appliance
virsh net-start openshift-appliance
```

*Get bridge interface name:*

```
$ virsh net-info openshift-appliance
Name:           openshift-appliance
UUID:           a83d55fa-d72a-4ad0-a38f-ebb846d863ab
Active:         yes
Persistent:     yes
Autostart:      yes
Bridge:         virbr2

```

In my case: `virbr2`


*Configure firewalld:*

```
export BRIDGE_IF=virbr2 # From command above

firewall-cmd --zone=libvirt --add-interface=${BRIDGE_IF} --permanent

firewall-cmd --zone=libvirt --add-service=dhcp --permanent
firewall-cmd --zone=libvirt --add-service=dns --permanent
firewall-cmd --zone=libvirt --add-service=ssh --permanent

firewall-cmd --reload
```

## Let's start to prepare root disc of our appliance

If you want to know all details, checkout the [user guide](https://github.com/openshift/appliance/blob/master/docs/user-guide.md#disk-image-build---lab)

Run all as root:
```bash
export APPLIANCE_IMAGE="quay.io/edge-infrastructure/openshift-appliance"
export APPLIANCE_ASSETS=$(pwd)
podman pull $APPLIANCE_IMAGE
```

Use and adjust existing [appliance-config.yaml](appliance-config.yaml) or create a new one:
```bash
podman run --rm -it --pull newer -v $APPLIANCE_ASSETS:/assets:Z $APPLIANCE_IMAGE generate-config
```

Build the os image (for more details add `--log-level debug` :
```bash
podman run --rm -it --pull newer --privileged --net=host -v $APPLIANCE_ASSETS:/assets:Z $APPLIANCE_IMAGE build

# Copy it to the vm place
cp -v appliance.raw /var/lib/libvirt/images/openshift-appliance.raw
```


## Let's create the OpenShift configuration.

If you want to know all details, checkout the [user guide](https://github.com/openshift/appliance/blob/master/docs/user-guide.md#openshift-cluster-installation-user-site)

Create new [install-config.yaml](install-config.yaml) and [agent-config.yaml](agent-config.yaml) or use existing ones:

```bash
rm -rf cluster_config/
mkdir cluster_config/
cp -v cp installer-config/* cluster_config/
```

Build OpenShift config ISO

```bash
./openshift-install agent create config-image --dir cluster_config/
# Copy it to the VM
cp -v cluster_config/agentconfig.noarch.iso /var/lib/libvirt/images/agentconfig.noarch.iso
```

## Let's start the appliance

```bash
virsh define virsh-vm.xml
virsh start openshift-appliance
virsh console openshift-appliance
```

Follow the installation via:

```bash
./openshift-install agent wait-for bootstrap-complete --dir cluster_config/
INFO Cluster is not ready for install. Check validations
WARNING Cluster validation: The cluster has hosts that are not ready to install.
WARNING Host openshift-appliance validation: Host couldn't synchronize with any NTP server
INFO Host openshift-appliance: custom discovery ignition config was applied
INFO Host openshift-appliance: updated status from insufficient to known (Host is ready to be installed)
INFO Preparing cluster for installation
INFO Cluster validation: All hosts in the cluster are ready to install.
INFO Host openshift-appliance: updated status from known to preparing-for-installation (Host finished successfully to prepare for installation)
...

```


## Optional: Configure load balancer/proxy for external access

```
cat - >/etc/systemd/system/openshift-4-loadbalancer.service <<EOF
[Unit]
Description=OpenShift 4 LoadBalancer CLUSTER
After=network.target

[Service]
Type=simple
TimeoutStartSec=5m

ExecStartPre=-/usr/bin/podman rm "openshift-4-loadbalancer"
ExecStartPre=/usr/bin/podman pull quay.io/redhat-emea-ssa-team/openshift-4-loadbalancer
ExecStart=/usr/bin/podman run --name openshift-4-loadbalancer --net host \
  -e API=appliance=192.168.13.12:6443 \
  -e API_LISTEN=*:6443,192.168.13.1:6443 \
  -e INGRESS_HTTP=appliance=192.168.13.12:80 \
  -e INGRESS_HTTP_LISTEN=*:80,192.168.13.1:80 \
  -e INGRESS_HTTPS=appliance=192.168.13.12:443 \
  -e INGRESS_HTTPS_LISTEN=*:443,192.168.13.1:443 \
  -e MACHINE_CONFIG_SERVER=appliance=192.168.13.12:22623 \
  -e MACHINE_CONFIG_SERVER_LISTEN=127.0.0.1:22623,192.168.13.1:22623 \
  -e STATS_LISTEN=127.0.0.1:1984 \
  -e STATS_ADMIN_PASSWORD=aengeo4oodoidaiP \
  -e HAPROXY_CLIENT_TIMEOUT=1m \
  -e HAPROXY_SERVER_TIMEOUT=1m \
  quay.io/redhat-emea-ssa-team/openshift-4-loadbalancer

ExecReload=-/usr/bin/podman stop "openshift-4-loadbalancer"
ExecReload=-/usr/bin/podman rm "openshift-4-loadbalancer"
ExecStop=-/usr/bin/podman stop "openshift-4-loadbalancer"
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF

```

Start the lb:
```
systemctl enable --now openshift-4-loadbalancer
```

# DONE 🎉 - Enjou your demo env



## Troubleshooting

Assisted installer API: https://access.redhat.com/documentation/en-us/assisted_installer_for_openshift_container_platform/2023/html/assisted_installer_for_openshift_container_platform/installing-with-api


On the openshift-appliance node:

```
curl http://localhost:8090/api/assisted-install/v2/clusters | jq '.[0].validations_info | fromjson'
```

```
# curl -s http://localhost:8090/api/assisted-install/v2/clusters | jq '.[0].validations_info | fromjson | .["hosts-data"]'
[
  {
    "id": "all-hosts-are-ready-to-install",
    "status": "failure",
    "message": "The cluster has hosts that are not ready to install."
  },
  {
    "id": "sufficient-masters-count",
    "status": "success",
    "message": "The cluster has the exact amount of dedicated control plane nodes."
  }
]

```

```
# curl -s http://localhost:8090/api/assisted-install/v2/clusters/a04681f0-c051-4143-85c0-a0bc4d71ea5c/hosts | jq '.[0].validations_info | fromjson | .network[] | select(.status == "failure")'
{
  "id": "ntp-synced",
  "status": "failure",
  "message": "Host couldn't synchronize with any NTP server"
}
{
  "id": "has-default-route",
  "status": "failure",
  "message": "Host has not yet been configured with a default route."
}

```