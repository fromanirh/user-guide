# Virtual machine templates

## What is a virtual machine template?

The KubeVirt projects provides a set of [templates](https://docs.okd.io/latest/dev_guide/templates.html) to create VMs to handle common usage scenarios.
These templates provide a combination of some key factors that could be further customized and processed to have a Virtual Machine object.
The key factors which define a template are

- Workload
  Most Virtual Machine should be *generic* to have maximum flexibility; the *highperformance* workload trades some of this flexibility to
  provide better performances.
- Guest Operating System (OS)
  This allow to ensure that the emulated hardware is compatible with the guest OS. Furthermore, it allows to maximize the stability
  of the VM, and allows performance optimizations.
- Size (flavor) 
  Defines the amount of resources (CPU, memory) to allocate to the VM.

More documentation is available in the [common templates subproject](https://github.com/kubevirt/common-templates)

## Accessing the virtual machine templates

If you installed KubeVirt using a supported method you should find the common templates preinstalled in the cluster.
Should you want to upgrade the templates, or install them from scratch, you can use one of the [supported releases](https://github.com/kubevirt/common-templates/releases)

To install the templates:
```bash
$ export VERSION="v0.3.1"
$ oc create -f https://github.com/kubevirt/common-templates/releases/download/$VERSION/common-templates-$VERSION.yaml
```

## Editable fields

You can edit the fields of the templates which define the amount of resources which the VMs will receive.

Each template can list a different set of fields that are to be considered editable.
The fields are used as hints for the user interface, and also for other components in the cluster.

The editable fields are taken from annotations in the template. Here is a snippet presenting a couple of most
commonly found editable fields:

```yaml
metadata:
  annotations:
    template.cnv.io/editable: |
      /objects[0].spec.template.spec.domain.cpu.cores
      /objects[0].spec.template.spec.domain.resources.requests.memory
```

Each entry in the editable field list must be a [jsonpath](https://kubernetes.io/docs/reference/kubectl/jsonpath/).
The jsonpath root is the objects: element of the template.
The actually editable field is the last entry (the "leaf") of the path. For example, the following minimal snippet highlights
the fields which you can edit:
```yaml
objects:
  spec:
    template:
      spec:
        domain:
          cpu:
            cores:
              VALUE # this is editable
          resources:
            requests:
              memory:
                VALUE # this is editable
```

## Relationship between templates and VMs

Once [processed](https://docs.openshift.com/enterprise/3.0/dev_guide/templates.html#creating-from-templates-using-the-cli), the templates produce VM objects to be
used in the cluster. The VMs produced from templates will have a `vm.cnv.io/template` label, whose value will be the name of the parent template,
for example `fedora-generic-medium`:
```yaml
  metadata:
    labels:
      vm.cnv.io/template: fedora-generic-medium
```
This make it possible to query for all the VMs built from any template.

Example:
```bash
oc process -o yaml rhel7-generic-tiny PVCNAME=mydisk NAME=rheltinyvm
```

And the output:
```yaml
apiVersion: v1
items:
- apiVersion: kubevirt.io/v1alpha2
  kind: VirtualMachine
  metadata:
    labels:
      vm.cnv.io/template: rhel7-generic-tiny
    name: rheltinyvm
    osinfoname: rhel7.0
  spec:
    running: false
    template:
      spec:
        domain:
          cpu:
            cores: 1
          devices:
            disks:
            - disk:
                bus: virtio
              name: rootdisk
              volumeName: rootvolume
            rng: {}
          resources:
            requests:
              memory: 1G
        terminationGracePeriodSeconds: 0
        volumes:
        - name: rootvolume
          persistentVolumeClaim:
            claimName: mydisk
        - cloudInitNoCloud:
            userData: |-
              #cloud-config
              password: redhat
              chpasswd: { expire: False }
          name: cloudinitvolume
kind: List
metadata: {}
```

You can add add the VM from the template to the cluster in one go
```bash
oc process rhel7-generic-tiny PVCNAME=mydisk NAME=rheltinyvm | oc apply -f -
```

Please note that, after the generation step, VM objects and template objects have no relationship with each other besides the aforementioned label (e.g. changes
in templates do not automatically affect VMs, or vice versa).

## common template customization

The templates provided by the kubevirt project provide a set of conventions and annotations that augment the basic feature of the
[openshift templates](https://docs.okd.io/latest/dev_guide/templates.html).
You can customize your kubevirt-provided templates editing these annotations, or you can add them to your existing templates to
make them consumable by the kubevirt services.

Here's a description of the kubevirt annotations. Unless otherwise specified, the following keys are meant to be top-level entries
of the template metadata, like

```yaml
apiVersion: v1
kind: Template
metadata:
  name: windows-10
  annotations:
    openshift.io/display-name: "Generic demo template"
```

All the following annotations are prefixed with `defaults.template.cnv.io`, which is omitted below for brevity. So the actual annotations you should use will look like

```yaml
apiVersion: v1
kind: Template
metadata:
  name: windows-10
  annotations:
    defaults.template.cnv.io/disk: default-disk
    defaults.template.cnv.io/volume: default-volume
    defaults.template.cnv.io/nic: default-nic
    defaults.template.cnv.io/network: default-network
```

Unless otherwise specified, all annotations are meant to be safe defaults, both for performance and compability, and hints for the CNV-aware UI and tooling.

#### disk
default disk type to use. It is recommended to aim for maximum compatibility.

Example:
```yaml
apiVersion: v1
kind: Template
metadata:
  name: Linux
  annotations:
    defaults.template.cnv.io/disk: virtio
```

#### nic
default nic model to use. Likewise the disk implementation, it is recomended to pick default which ensure the maximum compatibility.

Example:
```yaml
apiVersion: v1
kind: Template
metadata:
  name: Windows
  annotations:
    defaults.template.cnv.io/nic: e1000
```

#### volume
Default volume implementation to be used as backend for disks. The underlying volume implementation allow to configure the performance parameters (cache).

Example:
```yaml
apiVersion: v1
kind: Template
metadata:
  name: Linux
  annotations:
    defaults.template.cnv.io/volume: custom-volume
```

#### network
Default network to attach the NIC(s) to. Likewise the volume, the underlying network should be configured for the performance and the connectivity.

Example:
```yaml
apiVersion: v1
kind: Template
metadata:
  name: Linux
  annotations:
    defaults.template.cnv.io/network: fast-net
```
