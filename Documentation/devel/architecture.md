# rkt architecture

## Overview

rkt's primary interface is a command-line tool, `rkt`, which does not require a long running daemon.
This architecture allows rkt to be updated in-place without affecting application containers which are currently running.
It also means that levels of privilege can be separated out between different operations.

All state in rkt is communicated via the filesystem.
Facilities like file-locking are used to ensure co-operation and mutual exclusion between concurrent invocations of the `rkt` command.

## Stages

Execution with rkt is divided into several distinct stages.

_**NB** The goal is for the ABI between stages to be relatively fixed, but while rkt is still under heavy development this is still evolving.
Until https://github.com/coreos/rkt/issues/572 is resolved, this should be considered in flux and the description below may not be authoritative._

### Stage 0

The first stage is the actual `rkt` binary itself.
When running a pod, this binary is responsible for performing a number of initial preparatory tasks:

- Fetching the specified ACIs, including the stage1 ACI of --stage1-{url,path,name,hash,from-dir} if specified.
- Generating a Pod UUID
- Generating a Pod Manifest
- Creating a filesystem for the pod
- Setting up stage1 and stage2 directories in the filesystem
- Unpacking the stage1 ACI into the pod filesystem
- Unpacking the ACIs and copying each app into the stage2 directories

Given a run command such as:

```
# rkt run app1.aci app2.aci
```

a pod manifest compliant with the ACE spec will be generated, and the filesystem created by stage0 should be:

```
/pod
/stage1
/stage1/manifest
/stage1/rootfs/init
/stage1/rootfs/opt
/stage1/rootfs/opt/stage2/${app1-name}
/stage1/rootfs/opt/stage2/${app2-name}
```

where:

- `pod` is the pod manifest file
- `stage1` is a copy of the stage1 ACI that is safe for read/write
- `stage1/manifest` is the manifest of the stage1 ACI
- `stage1/rootfs` is the rootfs of the stage1 ACI
- `stage1/rootfs/init` is the actual stage1 binary to be executed (this path may vary according to the `coreos.com/rkt/stage1/run` Annotation of the stage1 ACI)
- `stage1/rootfs/opt/stage2` are copies of the unpacked ACIs

At this point the stage0 execs `/stage1/rootfs/init` with the current working directory set to the root of the new filesystem.

### Stage 1

The next stage is a binary that the user trusts to set up cgroups, execute processes, and perform other operations as root on the host.
This stage has the responsibility of taking the pod filesystem that was created by stage0 and creating the necessary cgroups, namespaces and mounts to launch the pod.
Specifically, it must:

- Read the Image and Pod Manifests. The Image Manifest defines the default `exec` specifications of each application; the Pod Manifest defines the ordering of the units, as well as any overrides.
- Generate systemd unit files from those Manifests
- Create and enter network namespace if rkt is not started with `--net=host`
- Start systemd-nspawn (which takes care of the following steps)
    - Set up any external volumes
    - Launch systemd as PID 1 in the pod within the appropriate cgroups and namespaces
    - Have systemd inside the pod launch the app(s).

This process is slightly different for the qemu-kvm stage1 but a similar workflow starting at `exec()`'ing kvm instead of an nspawn.

### Stage 1 systemd Architecture

rkt's Stage1 includes a very minimal systemd that takes care of launching the apps in each pod, apply per-app resource isolators and make sure the apps finish in an orderly manner.

We will now detail how the starting, shutdown, and exist status collection of the apps in a pod are implemented internally.

![rkt-systemd](rkt-systemd.png)

There's a systemd rkt apps target (`default.target`) which has a [*Wants*](http://www.freedesktop.org/software/systemd/man/systemd.unit.html#Wants=) and [*After*](http://www.freedesktop.org/software/systemd/man/systemd.unit.html#Before=) dependency on each app's service file, making sure they all start.

Each app's service has a *Wants* dependency on an associated reaper service that deals with writing the app's status exit.
Each reaper service has a *Wants* and *After* dependency with a shutdown service that simply shuts down the pod.

The reaper services and the shutdown service all start at the beginning but do nothing and remain after exit (with the [*RemainAfterExit*](http://www.freedesktop.org/software/systemd/man/systemd.service.html#RemainAfterExit=) flag).
By using the [*StopWhenUnneeded*](http://www.freedesktop.org/software/systemd/man/systemd.unit.html#StopWhenUnneeded=) flag, whenever they stop being referenced, they'll do the actual work via the *ExecStop* command.

This means that when an app service is stopped, its associated reaper will run and will write its exit status to `/rkt/status/${app}` and the other apps will continue running.
When all apps' services stop, their associated reaper services will also stop and will cease referencing the shutdown service causing the pod to exit.
Every app service has an [*OnFailure*](http://www.freedesktop.org/software/systemd/man/systemd.unit.html#OnFailure=) flag that starts the `halt.target`.
This means that if any app in the pod exits with a failed status, the systemd shutdown process will start, the other apps' services will automatically stop and the pod will exit.
In this case, the failed app's exit status will get propagated to rkt.

A [*Conflicts*](http://www.freedesktop.org/software/systemd/man/systemd.unit.html#Conflicts=) dependency was also added between each reaper service and the halt and poweroff targets (they are triggered when the pod is stopped from the outside when rkt receives `SIGINT`).
This will activate all the reaper services when one of the targets is activated, causing the exit statuses to be saved and the pod to finish like it was described in the previous paragraph.

### Stage 2

The final stage, stage2, is the actual environment in which the applications run, as launched by stage1.


### Image lifecycle

rkt commands like prepare and run, as a first step, need to retrieve all the images requested in the command line and prepare the stage2 directories with the application contents.

This is done with the following chain:

```
 -----------            -----------            ------------
|           |          |           |          |            |
|   Fetch   |--------->|   Store   |--------->|   Render   |
|           |          |           |          |            |
 -----------            -----------            ------------

```

* Fetch: in the fetch phase rkt retrieves the requested images. The fetching implementation depends on the provided image argument such as an image string/hash/https URL/file (e.g. `example.com/app:v1.0`).
* Store: in the store phase the fetched images are saved to the local store. The local store is a cache for fetched images and related data.
* Render: in the render phase, a renderer pulls the required images from the store and renders them so they can be easily used as stage2 content.


These three logical blocks are implemented inside rkt in this way:

```
 ------------            ---------------                -------------                    ------------------------
|            |          |               |              |             |     overlayfs    |                        |
|  Fetchers  |--------->|  Image Store  |<-------------|  TreeStore  |<-----------------|  Stage1-2 fs contents  |
|            |          |               |<----         |             |           -------|                        |
 ------------            ---------------      \         -------------           /        ------------------------
                                               \                               /
                                                \       -----------------     /
                                                 \     |                 |   /
                                                  -----| Direct stage1-2 |---
                                                       |   renderer      |
                                                       |                 |
                                                        -----------------
```

Currently rkt implements the [appc][appc-spec] internally, converting to it from other container image formats for compatibility. In the future, additional formats like the [OCI image spec][oci-img-spec] may be added to rkt, keeping the same basic scheme for fetching, storing, and rendering application container images.

* Fetchers: Fetchers retrieve images from either a provided URL, or a URL found by [image discovery][appc-discovery] on a given image string. Fetchers read data from the Image Store to check if an image is already present. Once fetched, images are verified with their signatures, then saved in the Image Store. An image's [dependencies][appc-dependencies] are also discovered and fetched. For details, see the [image fetching][rkt-image-fetching] documentation.
* Image Store: the Image Store is used to store images (currently ACIs) and their related information.
* The render phase can be done in different ways:
 * Directly render the stage1-2 contents inside a pod. This will require more disk space and more stage1-2 preparation time.
 * Render in the treestore. The treestore is a cache of rendered images (currently ACIs). When using the treestore, rkt mounts an overlayfs with the treestore rendered image as its lower directory.

When using stage1-2 with overlayfs a pod will contain references to the required treestore rendered images. So there's an hard connection between pods and the treestore.

*Aci Renderer*

Both stage1-2 render modes internally uses the [aci renderer](https://github.com/appc/spec/tree/master/pkg/acirenderer).
Since an ACI may depend on other ones the acirenderer may require other ACIs.
The acirenderer only relies on the ACIStore, so all the required ACIs must already be available in the store.
Additionally, since appc dependencies can be found only via discovery, a dependency may be updated and so there can be multiple rendered images for the same ACI.

Given this 1:N relation between an ACI and their rendered images, the ACIStore and TreeStore are decoupled.

[appc-spec]: https://github.com/appc/spec
[appc-dependencies]: https://github.com/appc/spec/blob/master/spec/aci.md#image-manifest-schema
[appc-discovery]: https://github.com/appc/spec/blob/master/spec/discovery.md
[oci-img-spec]: https://github.com/opencontainers/image-spec
[rkt-image-fetching]: ../image-fetching-behavior.md
