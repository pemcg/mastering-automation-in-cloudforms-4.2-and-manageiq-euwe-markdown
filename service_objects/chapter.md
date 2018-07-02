# Service Objects

We saw in [VM Provisioning
Objects](../vm_provisioning_objects/chapter.asciidoc) that provisioning
operations always include a *request* object, and a *task* object that
links to *source* and *destination* objects.

When we provision a virtual machine from a service there are many more
objects involved, because we are creating and referencing more items
(creating both a service and potentially several new component VMs).
When we provision from a service *bundle*, there will be several
individual *items* to provision as part of the bundle. Even when we
provision from a single service *item* however, the objects are
structured as if we were provisioning a bundle containing only one item.

In this chapter we will look at some of the objects involved in
provisioning a single VM from a service catalog item. The objects are
visible to us during the processing of the *CatalogItemInitialization*
state machine.

For this example:

  - We are using CloudForms 4.1

  - The provider is RHEV

  - The service catalog item name that we’ve ordered from is called
    "RHEL 7.2 Server (RHEV)"

  - The service catalog item was created to clone from a RHEV template
    called "rhel72-generic"

  - The new service name is "My New Service"

  - The resulting service contains a VM called test05.

We can use *object\_walker* with the following
`walk_association_whitelist` to dump the objects:

``` ruby
{ 'MiqAeServiceServiceTemplateProvisionTask': ['source',
                                               'destination',
                                               'miq_request',
                                               'miq_request_tasks',
                                               'service_resource'],
'MiqAeServiceServiceResource': ['resource','service_template'],
'MiqAeServiceMiqProvisionRequestTemplate': ['source','destination'],
'MiqAeServiceManageIQ_Providers_Redhat_InfraManager_Provision': ['source','destination'],
'MiqAeServiceManageIQ_Providers_Redhat_InfraManager_Vm': ['service'] }
```

We’ll call the object\_walker instance from the **post5** state/stage of
the CatalogItemInitialization state machine.

## Object Structure

We can illustrate the main object structure in [figure\_title](#i1)
(some objects and links/relationships have been omitted for clarity).

![Service Object Relationship](images/service_objects.png)

​  

### Service Template Provision Task

Our entry point into the object structure from `$evm` is to the main
*ServiceTemplateProvisionTask* object. We access this from:

``` ruby
$evm.root['service_template_provision_task']
```

From here we can access any of the other objects by following
associations.

#### Source

accessed from:

``` ruby
$evm.root['service_template_provision_task'].source
```

This is the *ServiceTemplate* object representing the service catalog
item that has been ordered from.

#### Destination

accessed from:

``` ruby
$evm.root['service_template_provision_task'].destination
```

This is the *Service* object representing the new service that will be
created under *My Services*.

### Service Template Provisioning Request

accessed from:

``` ruby
$evm.root['service_template_provision_task'].miq_request
```

This is the initial *ServiceTemplateProvisionRequest* object that was
created when we first ordered the new service. It is the request object
for the entire service provision operation, including all VMs created as
part of the service. This request object has associations to each of the
task objects involved in assembling the service, and they in turn have
backlinks to this request object.

### Child miq\_request\_task

accessed
from:

``` ruby
$evm.root['service_template_provision_task'].miq_request_tasks.each do |child_task|
```

This is also a *ServiceTemplateProvisionTask* object, and is the task
object that represents the creation of an item for the new service.
There will be a child miq\_request\_task for each item (e.g. virtual
machine) that makes up the final service, so for a service bundle
containing three VMs, there will be three child miq\_request\_tasks.

#### Service resource

accessed from:

``` ruby
child_task.service_resource
```

This *ServiceResource* object stores details about this particular
service item, and its place in the overall service structure. A
*ServiceResource* object has attributes such as:

``` ruby
service_resource.group_idx
service_resource.provision_index
...
service_resource.start_action
service_resource.start_delay
service_resource.stop_action
service_resource.stop_delay
```

These are generally zero or *nil* for a single-item service, but
represent the values selected in the WebUI for a multi-item service
bundle (see [figure\_title](#i1)).

![Start and stop actions and delays in a multi-item
bundle](images/ss1.png)

​  

The service resource has a relationship to the *ServiceTemplate* object
via `child_task.service_resource.service_template`.

#### Source

accessed from:

``` ruby
child_task.source
```

or

``` ruby
child_task.service_resource.resource
```

This is the *MiqProvisionRequestTemplate* object that describes how the
resulting VM will be created. The object looks very similar to a
traditional VM provisioning request object, and contains an options hash
populated from the dialog options that were selected when the service
item was created (e.g. placement options, memory size, CPUs, etc).

#### Destination

accessed from:

``` ruby
child_task.destination
```

This is the same *Service* object that is accessible from
`$evm.root['service_template_provision_task'].destination`.

### Grandchild miq\_request\_task

accessed from:

``` ruby
child_task.miq_request_tasks.each do |grandchild_task|
```

This is an *ManageIQ\_Providers\_Redhat\_InfraManager\_Provision*
miq\_request\_task object, and is the task object that represents the
creation of the VM. This is exactly the same as the task object
described in [???](#vm-provisioning-objects).

It is the grandchild miq\_request\_task that contains the options hash
for the VM to be provisioned; this being cloned from the options hash in
the *MiqProvisionRequestTemplate* object. If we have a service dialog
that prompts for properties affecting the provisioned VM (such as VM
name, number of CPUs, memory, etc.), we must pass these dialog values to
the grandchild task options hash.

#### Source

accessed from:

``` ruby
grandchild_task.source
```

This is the *ManageIQ\_Providers\_Redhat\_InfraManager\_Template* object
that represents the RHEV template that the new VM will be cloned from.

#### Destination

accessed from:

``` ruby
grandchild_task.destination
```

or

``` ruby
grandchild_task.vm
```

This is the *ManageIQ\_Providers\_Redhat\_InfraManager\_Vm* object that
represents the newly created VM. This VM object has an association
`service` that links to the newly created service object.

## Summary

In this chapter we’ve taken a detailed look at the various objects that
are involved in provisioning a virtual machine from a service. This is
the object view from any method running as part of the service provision
state machine.

The lowest layer of objects in [figure\_title](#i1) - the grandchild
miq\_request\_task with its source and destination objects - correspond
to the virtual machine provisioning objects that we discussed in [VM
Provisioning Objects](../vm_provisioning_objects/chapter.asciidoc). When
the service provision state machine hands over to the VM provision state
machine, these are indeed the objects that are referenced at this latter
stage, just like any other VM provision workflow. Any VM provision state
machine methods that we may have written that access the attributes of
these objects will see no difference. The only change is in the type of
request object; `$evm.root['miq_provision'].miq_provision_request` will
in this case be a service\_template\_provision\_request object.
