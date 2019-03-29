## Manual Install Guide for Integration of HPE 3PAR Containerized Plugin with RedHat OpenShift / Kubernetes (ADVANCED)

* [Introduction](#introduction)
* [Before you begin](#before)
* [Support Matrix for Kubernetes and Openshift 3.7](#support)
* [Deploying the HPE 3PAR Volume Plug-in in Kubernetes/OpenShift](#deploying)
  * [Configuring etcd](#etcd)
  * [Installing the HPE 3PAR Volume Plug-in](#installing)
* [Usage](#usage)
---

### Introduction <a name="introduction"></a>
This document details the installation steps in order to get up and running quickly with the HPE 3PAR Volume Plug-in for Docker within a Kubernetes 1.7/Openshift 3.7 environment.

**We highly recommend to use the Ansible playbooks that simplify and automate the install process before using the manual install process.**
[/ansible_3par_docker_plugin/README.md](/ansible_3par_docker_plugin/README.md)

## Before you begin <a name="before"></a>
* You need to have a basic knowledge of containers

* You should have Kubernetes or OpenShift deployed within your environment. If you want to learn how to deploy Kubernetes or OpenShift, please refer to the documents below for more details.

  * Kubernetes https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

  * OpenShift https://docs.openshift.org/3.7/install_config/install/planning.html

## Support Matrix for Kubernetes and Openshift 3.7 <a name="support"></a>

| Platforms                                               | Support for Containerized Plugin | Docker Engine Version | HPE 3PAR OS version              |
|---------------------------------------------------------|----------------------------------|-----------------------|----------------------------------|
| Kubernetes 1.6.13                                       | Yes                              | 1.12.6                | 3.2.2 MU6+ P107 & 3.3.1 MU1, MU2 |
| Kubernetes 1.7.6                                        | Yes                              | 1.12.6                | 3.2.2 MU6+ P107 & 3.3.1 MU1, MU2 |
| Kubernetes 1.8.9                                        | Yes                              | 17.06                 | 3.2.2 MU6+ P107 & 3.3.1 MU1, MU2 |
| Kubernetes 1.10.3                                       | Yes                              | 17.03                 | 3.2.2 MU6+ P107 & 3.3.1 MU1, MU2 |
| OpenShift 3.7 RPM based installation (Kubernetes 1.7.6) | Yes                              | 1.12.6                | 3.2.2 MU6+ P107 & 3.3.1 MU1, MU2   |

**NOTE**
  * Managed Plugin is not supported for Kubernetes or Openshift 3.7

  * The install of OpenShift for this paper was done on RHEL 7.4. Other versions of Linux may not be supported.

## Deploying the HPE 3PAR Volume Plug-in in Kubernetes/OpenShift <a name="deploying"></a>

Below is the order and steps that will be followed to deploy the **HPE 3PAR Volume Plug-in for Docker (Containerized Plug-in) within a Kubernetes 1.7/OpenShift 3.7** environment.

Let's get started.

For this installation process, login in as **root**:

```bash
$ sudo su -
```

### Configuring etcd <a name="etcd"></a>

**Note:** For this quick start quide, we will be creating a single-node **etcd** deployment, as shown in the example below, but for production, it is **recommended** to deploy a High Availability **etcd** cluster.

For more information on etcd and how to setup an **etcd** cluster for High Availability, please refer:
[/docs/advanced/etcd_cluster_setup.md](/docs/advanced/etcd_cluster_setup.md)

1. Export the Kubernetes/OpenShift `Master` node IP address

```
$ export HostIP="<Master node IP>"
```

2. Run the following to create the `etcd` container.


>**NOTE:** This etcd instance is separate from the **etcd** deployed by Kubernetes/OpenShift and is **required** for managing the **HPE 3PAR Volume Plug-in for Docker**. We need to modify the default ports (**2379, 4001, 2380**) of the **new etcd** instance to prevent conflicts. This allows two instances of **etcd** to safely run in the environment.`

```yaml
sudo docker run -d -v /usr/share/ca-certificates/:/etc/ssl/certs -p 40010:40010 \
-p 23800:23800 -p 23790:23790 \
--name etcd_hpe quay.io/coreos/etcd:v2.2.0 \
-name etcd0 \
-advertise-client-urls http://${HostIP}:23790,http://${HostIP}:40010 \
-listen-client-urls http://0.0.0.0:23790,http://0.0.0.0:40010 \
-initial-advertise-peer-urls http://${HostIP}:23800 \
-listen-peer-urls http://0.0.0.0:23800 \
-initial-cluster-token etcd-cluster-1 \
-initial-cluster etcd0=http://${HostIP}:23800 \
-initial-cluster-state new
```

### Installing the HPE 3PAR Volume Plug-in <a name="installing"></a>

> **NOTE:** This section assumes that you already have **Kubernetes** or **OpenShift** deployed in your environment, in order to run the following commands.

1. Install the iSCSI and Multipath packages(Not needed for SLES)

```
$ yum install -y iscsi-initiator-utils device-mapper-multipath
```

2. Configure /etc/multipath.conf

```
$ vi /etc/multipath.conf
```

>Copy the following into /etc/multipath.conf

```
defaults
{
    polling_interval 10
    max_fds 8192
}

devices
{
    device
	{
        vendor                  "3PARdata"
        product                 "VV"
        no_path_retry           18
        features                "0"
        hardware_handler        "0"
        path_grouping_policy    multibus
        #getuid_callout         "/lib/udev/scsi_id --whitelisted --device=/dev/%n"
        path_selector           "round-robin 0"
        rr_weight               uniform
        rr_min_io_rq            1
        path_checker            tur
        failback                immediate
    }
}
```

3. Enable the iscsid and multipathd services

```
$ systemctl enable iscsid multipathd
$ systemctl start iscsid multipathd
```

For SLES:
```
$ dracut --force --add multipath
$ sudo systemctl enable multipathd
$ sudo systemctl start multipathd
```

4. Configure `MountFlags` in the Docker service to allow shared access to Docker volumes(Not needed for SLES)

```
$ vi /usr/lib/systemd/system/docker.service
```

>Change **MountFlags=slave** to **MountFlags=shared** (default is slave)
>
>Save and exit

5. Restart the Docker daemon(Not needed for SLES)

```
$ systemctl daemon-reload
$ systemctl restart docker.service
```

6. Setup the Docker plugin configuration file

```
$ mkdir –p /etc/hpedockerplugin/
$ cd /etc/hpedockerplugin
$ vi hpe.conf
```

>Copy the contents from the sample hpe.conf file, based on your storage configuration for either iSCSI or Fiber Channel:

>##### HPE 3PAR iSCSI:
>
>[/docs/config_examples/hpe.conf.sample.3parISCSI](/docs/config_examples/hpe.conf.sample.3parISCSI)


>##### HPE 3PAR Fiber Channel:
>
>[/docs/config_examples/hpe.conf.sample.3parFC](/docs/config_examples/hpe.conf.sample.3parFC)

7. Use Docker Compose to deploy the HPE 3PAR Volume Plug-In for Docker (Containerized Plug-in) from the pre-built image available on Docker Hub:

```
$ cd ~
$ vi docker-compose.yml
```

> Copy the content below into the `docker-compose.yml` file

```
hpedockerplugin:
  image: hpestorage/legacyvolumeplugin:2.1
  container_name: plugin_container
  net: host
  privileged: true
  volumes:
     - /dev:/dev
     - /run/lock:/run/lock
     - /var/lib:/var/lib
     - /var/run/docker/plugins:/var/run/docker/plugins:rw
     - /etc:/etc
     - /root/.ssh:/root/.ssh
     - /sys:/sys
     - /root/plugin/certs:/root/plugin/certs
     - /sbin/iscsiadm:/sbin/ia
     - /lib/modules:/lib/modules
     - /lib64:/lib64
     - /var/run/docker.sock:/var/run/docker.sock
     - /opt/hpe/data:/opt/hpe/data:rshared
```

For SLES use:
```
hpedockerplugin:
  image: hpestorage/legacyvolumeplugin:3.1
  container_name: plugin_container
  net: host
  restart: always
  privileged: true
  volumes:
     - /dev:/dev
     - /run/lock:/run/lock
     - /var/lib:/var/lib
     - /var/run/docker/plugins:/var/run/docker/plugins:rw
     - /etc:/etc
     - /root/.ssh:/root/.ssh
     - /sys:/sys
     - /root/plugin/certs:/root/plugin/certs
     - /sbin/iscsiadm:/sbin/ia
     - /lib/modules:/lib/modules
     - /lib64:/lib64
     - /var/run/docker.sock:/var/run/docker.sock
     - /var/lib/rancher:/var/lib/rancher:rshared
     - /usr/lib64:/usr/lib64
```

>Save and exit

> **NOTE:** Before we start the HPE 3PAR Volume Plug-in container, make sure etcd is running.
>
>Use the Docker command: `docker ps -a | grep -i etcd_hpe`

8. Start the HPE 3PAR Volume Plug-in for Docker (Containerized Plug-in)

>Make sure you are in the location of the `docker-compose.yml` filed

```
$ docker-compose up -d
```

>**NOTE:** In case you are missing `docker-compose`, https://docs.docker.com/compose/install/#install-compose
>
```
$ curl -x 16.85.88.10:8080 -L https://github.com/docker/compose/releases/download/1.21.0/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```
>
>Visit https://docs.docker.com/compose/install/#install-compose for latest curl details
>
>Test the installation:
```
$ docker-compose --version
docker-compose version 1.21.0, build 1719ceb
```
> Re-run step 8

9. Success, you should now be able to test docker volume operations like:
```
$ docker volume create -d hpe --name sample_vol -o size=
```

10. Install the HPE 3PAR FlexVolume driver:
```
$ wget https://github.com/hpe-storage/python-hpedockerplugin/raw/master/dory_installer
$ chmod u+x ./dory_installer
$ sudo ./dory_installer
```

11. Confirm HPE 3PAR FlexVolume driver installed correctly:
```
$ ls -l /usr/libexec/kubernetes/kubelet-plugins/volume/exec/hpe.com~hpe/
-rwxr-xr-x. 1 docker docker 47046107 Apr 20 06:11 doryd
-rwxr-xr-x. 1 docker docker  6561963 Apr 20 06:11 hpe
-rw-r--r--. 1 docker docker      237 Apr 20 06:11 hpe.json
```

12. Run the following command to start the HPE 3PAR FlexVolume dynamic provisioner:

```
$ sudo /usr/libexec/kubernetes/kubelet-plugins/volume/exec/hpe.com~hpe/doryd /etc/kubernetes/admin.conf hpe.

```
>**NOTE:** If you see the following error:

```
Error getting config from file /etc/kubernetes/admin.conf - stat /etc/kubernetes/admin.conf: no such file or directory
Error getting config cluster - unable to load in-cluster configuration, KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT must be defined
```
>Run the following commands:
```
$ mkdir –p /etc/kubernetes
$ cp /root/.kube/config /etc/kubernetes/admin.conf
```

>Re-run the command to start the HPE 3PAR FlexVolume dynamic provisioner

>For more information on the HPE FlexVolume driver, please visit this link:
>
>https://github.com/hpe-storage/dory/

13. Repeat steps 1-9 on all worker nodes. **Steps 10-12 only needs to be ran on the Master node.**

>**Upon successful completion of the above steps, you should have a working installation of Openshift 3.7 integrated with HPE 3PAR Volume Plug-in for Docker**

## Usage <a name="usage"></a>

For usage go to:

[Usage](/docs/usage.md)
