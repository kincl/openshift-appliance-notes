# OpenShift Appliance Notes

Upstream docs:
- https://access.redhat.com/articles/7065136
- https://github.com/openshift/appliance/blob/master/docs/user-guide.md

Images:
- Upstream: `quay.io/edge-infrastructure/openshift-appliance`
- Downstream: `registry.redhat.io/assisted/agent-preinstall-image-builder-rhel9`

**Prefer the upstream builder for now which is where they are fixing bugs**

## Generate configuration

> It is not *strictly necessary* to use `sudo podman` in these examples below but you will **have** to use it later on to build the actual disk image so rather than copying the image between the root image storage and the user image storage I opted for just doing sudo for all commands :(

```
export APPLIANCE_IMAGE="quay.io/edge-infrastructure/openshift-appliance"
export APPLIANCE_ASSETS="/home/jkincl/storage/openshift-appliance"

sudo podman pull $APPLIANCE_IMAGE
mkdir $APPLIANCE_ASSETS
sudo podman run --rm -it -v $APPLIANCE_ASSETS:/assets:Z $APPLIANCE_IMAGE generate-config
```

## Configuration

- Change ocpRelease info
- Disk size in GB, only the number
- Set pull secret and SSH key
- allow defaults for imageRegistry section (comment it out)

### Example ApplianceConfig

```
apiVersion: v1beta1
kind: ApplianceConfig
ocpRelease:
  version: 4.16
  channel: stable
  cpuArchitecture: x86_64
diskSizeGB: 150
pullSecret: '<redacted>'
sshKey: '<redacted>'
userCorePass: <redacted>
enableDefaultSources: false
stopLocalRegistry: false
# Additional images to be included in the appliance disk image.
# [Optional]
# additionalImages:
#   - name: image-url
# Operators to be included in the appliance disk image.
# See examples in https://github.com/openshift/oc-mirror/blob/main/docs/imageset-config-ref.yaml.
# [Optional]
# operators:
#   - catalog: catalog-uri
#     packages:
#       - name: package-name
#         channels:
#           - name: channel-name
```

## Build the image

To move the image from your user to root's image storage **(not needed if using `sudo podman pull` above)**:

```
$ podman save $APPLIANCE_IMAGE | sudo podman load
```

> `sudo` is required here because we are running containers in containers

```
$ sudo podman run --privileged --net=host --rm -it -v $APPLIANCE_ASSETS:/assets:Z $APPLIANCE_IMAGE build
WARNING OCP release version 4.16.10 is not supported. Latest supported version: 4.15.
INFO Successfully downloaded appliance base disk image
INFO Successfully extracted appliance base disk image
INFO Successfully pulled container registry image
INFO Successfully pulled OpenShift 4.16.10 release images required for bootstrap
INFO Successfully pulled OpenShift 4.16.10 release images required for installation
INFO Successfully generated data ISO
INFO Successfully fetched openshift-install binary
INFO Successfully downloaded CoreOS ISO
INFO Successfully generated recovery CoreOS ISO
INFO Successfully generated appliance disk image
INFO Time elapsed: 9m32s
INFO
INFO Appliance disk image was successfully created in assets directory: assets/appliance.raw
INFO
INFO Create configuration ISO using: openshift-install agent create config-image
INFO Download openshift-install from: https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.16.10/openshift-install-linux.tar.gz
```

## Creating the config ISO

```
$ curl https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.16.10/openshift-install-linux.tar.gz
```

### Generate agent-config.yaml and install-config.yaml files

- https://github.com/openshift/appliance/blob/master/README.md#examples

This agent-config.yaml assumes that the SNO will get an IP of `172.31.255.6`

**agent-config.yaml**
```
apiVersion: v1alpha1
kind: AgentConfig
rendezvousIP: 172.31.255.6
```

**install-config.yaml**

The pullSecret is a dummy one and is required to get past the openshift-install config checks

```
apiVersion: v1
metadata:
  name: kincl-sno
baseDomain: airgap.dota-lab.iad.redhat.com
controlPlane:
  name: master
  replicas: 1
compute:
- name: worker
  replicas: 0
networking:
  networkType: OVNKubernetes
  machineNetwork:
  - cidr: 172.31.255.0/24
platform:
  none: {}
pullSecret: '{"auths":{"":{"auth":"dXNlcjpwYXNz"}}}'
sshKey: '<redacted>'
```

I place these files in a directory and then copy to a new `deployed/` subdirectory since openshift-install will consume the config files as part of the install

### Generate the ISO

```
$ ./openshift-install agent create config-image --dir deployed
INFO Configuration has 1 master replicas and 0 worker replicas
INFO The rendezvous host IP (node0 IP) is 172.31.255.6
INFO Consuming Install Config from target directory
INFO Consuming Agent Config from target directory
INFO Config-Image created in: deployed/auth
```

## Testing on KubeVirt

First convert the image to qemu to sparsify it:

```
$ qemu-img convert -O qcow2 appliance.raw appliance.qcow2
$ ls -lh
-rw-r--r--. 1 jkincl jkincl  25G Sep 16 16:25 appliance.qcow2
-rw-r--r--. 1 root   root   150G Sep 16 12:17 appliance.raw
```

```
$ oc create -f sno-vm.yaml
```

```
$ virtctl image-upload pvc kincl-sno-root -n jkincl --size 150Gi --image-path appliance.qcow2
```

```
$ virtctl image-upload pvc kincl-sno-config-iso -n jkincl --size 1Gi --image-path deployed/agentconfig.noarch.iso
```

Attach the root and ISO to boot the VM from the CD
