# Docker

Contain yourself...

---

## Getting Started
By default, REX-Ray's embedded Docker Volume Plug-in endpoint handles
requests from the local Docker service via a UNIX socket. Doing so
restricts the endpoint to the localhost, increasing network security by removing
a possible attack vector. If an externally accessible Docker Volume Plug-in
endpoint is required, it's still possible to create one by overriding the
address for the `default-docker` module in REX-Ray's configuration file:

```yaml
rexray:
  modules:
    default-docker:
      host: tcp://:7981
```

The above example illustrates how to override the `default-docker` module's
endpoint address. The value `tcp://:7981` instructs the Docker Volume Plug-in
to listen on port 7981 for all configured interfaces.

Using a TCP endpoint has a side-effect however -- the local Docker instance
will not know about the Volume Plug-in endpoint as there is no longer a UNIX
socket file in the directory the Docker service continually scans.

On the local system, and in fact on all systems where the Docker service needs
to know about this externally accessible Volume Plug-in endpoint, a spec file
must be created at `/etc/docker/plug-ins/rexray.spec`. Inside this file simply
include a single line with the network address of the endpoint. For example:

```bash
tcp://192.168.56.20:7981
```

With a spec file located at `/etc/docker/plug-ins/rexray.spec` that contains
the above contents, Docker instances will query the Volume Plug-in endpoint at
`tcp://192.168.56.20:7981` when volume requests are received.

### Volume Management
The `volume` sub-command for Docker 1.12+ should look similar to the following:

```sh
$ docker volume

Usage:	docker volume [OPTIONS] [COMMAND]

Manage Docker volumes

Commands:
  create                   Create a volume
  inspect                  Return low-level information on a volume
  ls                       List volumes
  rm                       Remove a volume
```

#### List Volumes
The list command reviews a list of available volumes that have been discovered
via Docker Volume Plug-in endpoints such as REX-Ray. Each volume name is
expected to be unique. Thus volume names must also be unique across all
endpoints, and in turn, across all storage platforms exposed by REX-Ray.

With the exception of the `local` driver, the list of returned volumes is
generated by the backend storage platform to which the configured driver
communicates:

```sh
$ docker volume ls
DRIVER              VOLUME NAME
local               local1
scaleio             Volume-001
virtualbox          vbox1
```

#### Inspect Volume
The inspect command can be used to retrieve details about a volume related to
both Docker and the underlying storage platform. The fields listed under
`Status` are all generated by REX-Ray, including `Size in GB`, `Volume Type`,
and `Availability Zone`.

The `Scope` parameter ensures that when the specified volume driver is
inspected by multiple Docker hosts, the volumes tagged as `global` are all
interpreted as the same volume. This reduces unnecessary round-trips in
situations where an application such as Docker Swarm is connected to hosts
configured with REX-Ray.

```sh
$ docker volume inspect vbox1
[
    {
        "Name": "vbox1",
        "Driver": "virtualbox",
        "Mountpoint": "",
        "Status": {
            "availabilityZone": "",
            "fields": null,
            "iops": 0,
            "name": "vbox1",
            "server": "virtualbox",
            "service": "virtualbox",
            "size": 8,
            "type": ""
        },
        "Labels": {},
        "Scope": "global"
    }
]
```

#### Create Volume
Docker's `volume create` command enables the creation of new volumes on the
underlying storage platform. Newly created volumes are available immediately
to be attached and mounted. The `volume create` command also supports the CLI
flag `-o|--opt` in order to support providing custom data to the volume creation
workflow:

```sh
$ docker volume create --driver=virtualbox --name=vbox2 --opt=size=2
vbox2
```

Additional, valid options for the `-o|--opt` parameter include:

option|description
------|-----------
size|Size in GB
IOPS|IOPS
volumeType|Type of Volume or Storage Pool
volumeName|Create from an existing volume name
volumeID|Create from an existing volume ID
snapshotName|Create from an existing snapshot name
snapshotID|Create from an existing snapshot ID

#### Remove Volume
A volume may be removed once it is no longer in use by a container, running or
otherwise. The process of removing a container actually causes the volume to
be removed if that is the last container to leverage said volume:

```sh
$ docker volume rm vbox2
```

### Containers with Volumes
Please review the [Applications](../examples/apps.md) section for information on
configuring popular applications with persistent storage via Docker and REX-Ray.

## Configuration
libStorage's Docker Integration Driver is compatible with 1.10+.

However, Docker 1.10.2+ is suggested if volumes are shared between containers
or interactive volume inspection requests are desired via the `/volumes`,
`/volumes/{service}`, and  `/volumes/{service}/{volumeID}` resources.

Please  note that this is *not* the same as
[Docker's Volume Plug-in](https://docs.docker.com/engine/extend/plugins_volume/).
libStorage does not provide a way to expose the Docker Integration Driver
via the Docker Volume Plug-in, but REX-Ray, which embeds libStorage,
does.

### Example Configuration
Below is an example `config.yml` that can be used.  The `volume.mount.preempt`
is an optional parameter here which enables any host to take control of a
volume irrespective of whether other hosts are using the volume.  If this is
set to `false` then plugins should ensure `safety` first by locking the
volume from to the current owner host. We also specify `docker.size` which will
create all new volumes at the specified size in GB.

```yaml
libstorage:
  host: unix:///var/run/libstorage/localhost.sock
  integration:
    volume:
      mount:
        preempt: true
      create:
        default:
          size: 1 # GB
  server:
    endpoints:
      localhost:
        address: unix:///var/run/libstorage/localhost.sock
    services:
      virtualbox:
        driver: virtualbox
        virtualbox:
          endpoint:       http://10.0.2.2:18083
          tls:            false
          volumePath:     $HOME/VirtualBox/Volumes
          controllerName: SATA
```

### Configuration Properties
The Docker integration driver adheres to the properties described in the
section on an
[Integration driver's volume-related properties](../servers/libstorage.md#volume-properties).

Please note that with `Docker` 1.9.1 or below, it is recommended that the
property `libstorage.integration.volume.remove.disable` be set to `true` in
order to prevent `Docker` from removing external volumes in-use by containers
that are forcefully removed.

## Managed Plug-ins
The REX-Ray managed plug-ins for Docker work with Docker 1.13+.

### Installation
Docker managed plug-ins may be installed with following command:

```bash
$ docker plugin install rexray/driver[:version]
```

The `[:version]` component in the above command is known as a Docker
_tag_ and its value follows the semantic versioning model. Omitting
the version is equivalent to specifying the `latest` tag -- the most
recent, stable version of a plug-in. The `edge` tag requests the most
recent, bleeding-edge version of the plug-in.

!!! note "note"
    Please note that most of REX-Ray's plug-ins must be configured and
    installed at the same time since Docker starts the plug-in when installed.
    Otherwise the plug-in will fail since it is not yet configured. Please
    see the sections below for platform-specific configuration options.

### Configuration
Docker volume plug-ins are configured via environment variables, and all
REX-Ray plug-ins share the following, common configuration options:

Environment Variable | Description | Default Value
---------------------|-------------|--------------
`REXRAY_FSTYPE` | The type of file system to use | `ext4`
`REXRAY_LOGLEVEL` | The log level | `warn`
`REXRAY_PREEMPT` | Enable preemption | `false`
`LIBSTORAGE_INTEGRATION_VOLUME_OPERATIONS_MOUNT_ROOTPATH` | The path within the volume to return to the integrator | `/data`
`LINUX_VOLUME_ROOTPATH` | A path to auto create within the volume | '/data'
`LINUX_VOLUME_FILEMODE` | File mode for mounted path | `0700`

### Building a Plug-in
Please see the build reference for
[Docker plug-ins](../../dev-guide/build-reference.md#building-docker-plug-ins).

### Creating a Plug-in
Please see the build reference for
[Docker plug-ins](../../dev-guide/build-reference.md#creating-docker-plug-ins).

### Storage Platforms
The following table lists the available REX-Ray managed Docker plug-ins:

| Provider              | Storage Platform  |
|-----------------------|----------------------|
| Amazon EC2 | [EBS](./docker/plug-ins/aws.md#aws-ebs) |
| | [EFS](./docker/plug-ins/aws.md#aws-efs) |
| | [S3FS](./docker/plug-ins/aws.md#aws-s3fs) |
| Ceph | [RBD](./docker/plug-ins/ceph.md#ceph-rbd) |
| Local | [CSI-NFS](./docker/plug-ins/csi-nfs.md) |
| Dell EMC | [Isilon](./docker/plug-ins/dellemc.md#dell-emc-isilon) |
| | [ScaleIO](./docker/plug-ins/dellemc.md#dell-emc-scaleio) |
| DigitalOcean | [Block Storage](./docker/plug-ins/digitalocean.md#do-block-storage) |
| Google | [GCE Persistent Disk](./docker/plug-ins/google.md#gce-persistent-disk) |
| Microsoft | [Azure Unmanaged Disk](./docker/plug-ins/microsoft.md#azure-ud) |
| OpenStack | [Cinder](./docker/plug-ins/openstack.md#cinder) |

## Examples
This section illustrates how to use REX-Ray and Docker together. The
typical name of the Docker volume driver when using REX-Ray is
`rexray`. However, please note that when using Docker Managed Plug-ins
the name of the volume driver becomes `rexray/STORAGE` with `STORAGE`
the name of the storage platform.

### Create a volume
The following example illustrates creating a volume:

```bash
$ docker volume create --driver rexray/ebs --name test-vol-1
```

Verify the volume was successfully created by listing the volumes:

```bash
$ docker volume ls
DRIVER          VOLUME NAME
rexray/ebs      test-vol-1
```

### Inspect a volume
The following example illustrates inspecting a volume:

```bash
$ docker volume inspect test-vol-1
```

```json
[
    {
        "Driver": "rexray/ebs",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/plug-ins/9f30ec546a4b1bb19574e491ef3e936c2583eda6be374682eb42d21bbeec0dd8/rootfs",
        "Name": "test-vol-1",
        "Options": {},
        "Scope": "global",
        "Status": {
            "availabilityZone": "default",
            "fields": null,
            "iops": 0,
            "name": "test-vol-1",
            "server": "ebs",
            "service": "ebs",
            "size": 16,
            "type": "default"
        }
    }
]
```

### Use a volume
The following example illustrates using a volume:

```bash
$ docker run -v test-vol-1:/data busybox mount | grep "/data"
/dev/xvdf on /data type ext4 (rw,seclabel,relatime,nouuid,attr2,inode64,noquota)
```

### Remove a volume
The following example illustrates removing a volume created:

```bash
$ docker volume rm test-vol-1
```

Validate the volume was deleted successfully by listing the volumes:

```bash
$ docker volume ls
DRIVER              VOLUME NAME
```

## Troubleshooting
If the REX-Ray service or a Docker Managed Plug-in is restarted while
volumes are shared between containers then problems may arise when one
of the containers is halted.

To avoid this issue, please consider avoiding halting containers that
consume shared volumes until all participating containers can be stopped
at the same time.
