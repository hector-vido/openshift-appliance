# OpenShift-based Appliance Builder

`openshift-appliance` is a command line utility for building a disk image that orchestrates OpenShift installation using the Agent-based installer. 
The primary use-case of the appliance is to support a fully disconnected installation
of an OpenShift cluster. Thus, all required images are included in the appliance disk image.

## Quick Start

### Build (binary or container image)

#### Build binary

##### Install dependencies

* libguestfs-tools
* coreos-installer
* oc
* skopeo
* podman

Note: for oc-mirror usage, the builder ensures that the pull secret exists at `~/.docker/config.json`

##### Build

``` bash
make build-appliance
```

#### Build container image

``` bash
export IMAGE=<image_url>

make build
```

### Run

#### Commands
* build
* clean
* generate-config

#### Flags
* --dir
* --log-level

#### Create config file (appliance-config.yaml)

A configuration file named `appliance-config.yaml` is required for running the tool. The file should include the following properties:

* ocpRelease.version: OCP release version in major.minor or major.minor.patch format (for major.minor, latest patch version will be used)
* ocpRelease.channel: OCP release update channel (stable|fast|eus|candidate)
* ocpRelease.cpuArchitecture: CPU architecture of the release payload (x86_64|aarch64|ppc64le)
* diskSizeGB: Virtual size of the appliance disk image
* pullSecret: PullSecret required for mirroring the OCP release payload
* sshKey: Public SSH key for accessing the appliance

##### Generate config file template

Using binary:
``` bash
./bin/openshift-appliance generate-config --dir <assets-dir>
```

Using container image:
``` bash
export CMD=generate-config
export ASSETS=<assets-dir>

make run --dir assets
```

##### Example

```
apiVersion: v1beta1
kind: ApplianceConfig
ocpRelease:
  version: "4.12.10"
  channel: "stable"
  cpuArchitecture: "x86_64"
diskSizeGB: 200
pullSecret: ...
sshKey: ...

```

#### Start appliance disk image build flow

Using binary:
``` bash
export LIBGUESTFS_BACKEND=direct

./bin/openshift-appliance build --dir <assets-dir> --log-level info
```

Using container image:
``` bash
export CMD=build
export ASSETS=<assets-dir>
export LOG_LEVEL=info/debug/error

make run build --dir assets
```

##### Cleanup

After a successful build, use the `clean` command before re-building the appliance (removes temp folder and state file).

## Development

### Running tests
```bash
skipper make test
```

### Running lint
```bash
make lint
```

### Debug

#### Bootstrap step

Add `--debug-bootstrap` flag to the build command to avoid machine reboot on bootstrap step completion. Useful for taking a snapshot of the appliance disk image before testing changes in the install ignition. 

#### Install step

Add `--debug-install` flag to the build command for enabling ssh login on the installation step.
The public ssh key provided in appliance-config.yaml is used (`sshKey` property).
During installation, to run `oc` commands, define `KUBECONFIG` using:
```bash
export KUBECONFIG=/etc/kubernetes/bootstrap-secrets/kubeconfig
```

#### Test changes in the install ignition

To debug/test changes made in the `InstallIgnition` asset, follow the steps described on [test_install_ignition.md](/hack/diskimage/test_install_ignition.md)

## Main Components

### Recovery ISO Assets (pkg/asset/recovery/)
* BaseISO - a CoreOS LiveCD used as a base for the recovery ISO
* RecoveryISO - the base ISO with an embedded recovery ignition

### Appliance Assets (pkg/asset/appliance/)
* BaseDiskImage - a CoreOS qcow2 disk image used as a base for the appliance disk image
* ApplianceDiskImage - the output disk image of the builder

### Ignition Assets (pkg/asset/ignition/)
* BootstrapIgnition - the ignition config used for cluster bootstrap step
* InstallIgnition - the ignition config used for cluster installation step
* RecoveryIgnition - the ignition config embedded into the recovery ISO

## High-Level Flow

The appliance build process consists of the following steps:
1. Download CoreOS ISO
   * Extracted from the `machine-os-images` image (included in the release payload)
   * Used as the base image of the recovery ISO
2. Generate recovery CoreOS ISO
   * Generated by embedding a custom bootstrap ignition to the base CoreOS ISO
   * Used as the `recovery` partition (labeled 'agentboot') of the appliance disk image
3. Pull registry image
   * Used for serving the OCP release images on bootstrap and installation steps
4. Pull release images required for bootstrap
   * Only a subset of the entire release payload (i.e. images that are required for bootstrap)
5. Pull release images required for installation
   * Includes the entire release payload
6. Generate data ISO
   * Includes the registry and release images that are pulled in previous steps
   * Used as the 'data partition' of the appliance disk image
7. Download appliance base disk image
   * A qcow2 image of CoreOS
   * Used as the base disk image of the appliance
8. Generate appliance disk image
   * Contains the following:
     * An ignition for orchestrating the OpenShift cluster installation 
     * A recovery partition for reinstalling if necessary
     * The full OCP release payload for supporting disconnected environments

[![asciicast](https://asciinema.org/a/583868.svg)](https://asciinema.org/a/583868)

## Appliance disk image structure

``` bash
$ virt-filesystems -a assets/appliance.raw -l -h
Name       Type        VFS      Label       Size  Parent
/dev/sda2  filesystem  vfat     EFI-SYSTEM  127M  -
/dev/sda3  filesystem  ext4     boot        350M  -
/dev/sda4  filesystem  xfs      root        180G  -
/dev/sda5  filesystem  ext4     agentboot   1.2G  -
/dev/sda6  filesystem  iso9660  agentdata   18G   -
```