# The Service Provisioning State Machine

As might be expected, both CloudForms and ManageIQ use a state machine
to intelligently handle the workflow for provisioning a service.
Although we rarely modify the *service provisioning state machine*, it
is useful to have an understanding of its steps and the functions that
it performs. This more theoretical chapter examines the state machine
and discusses its role in passing into the provisioning workflow the
service dialog values that the user has input.

## Class and Instances

The service provisioning state machine (the *ServiceProvision\_Template*
class) controls the sequence of steps involved in provisioning the
service. The *ManageIQ* domain contains four instances of this state
machine (see [figure\_title](#i1)).

![ServiceProvision\_Template class, instances and
method](images/ss1.png)

​  

The *ServiceProvision\_Template* class schema contains a number of
states. [figure\_title](#i2) shows the *default* instance of this state
machine:

![Fields of the ServiceProvision\_Template class](images/ss2.png)

​  

As we can see, most of the fields are **pre** and **post** placeholders
around the main **provision** and **checkprovisioned** states, to allow
for optional processing if required. The **configurechilddialog** state
can be used to populate the `options[:dialog]` hash in the child task if
required.

## Passing Service Dialog Options to the Child and Grandchild Tasks

One of the more complex tasks that must be achieved by some state in the
service provisioning state machine is to pass the values received from
the service dialog (if there is one) to the actual tasks performing the
provisioning of the virtual machine(s). The complexity arises from the
three 'generations' of task object involved in creating the service, the
service resources, and the actual VMs (see [figure\_title](#i3)).

![Task object hierarchy](images/task_hierarchy.png)

​  

This object hierarchy is represented at the highest level by the service
template provision task. We access this from:

``` ruby
$evm.root['service_template_provision_task']
```

The service\_template\_provision\_task has an assocation,
`miq_request_tasks`, containing the *miq\_request\_task* objects
representing the creation of the *service resource(s)*. These are the
items or resources making up the service request (even a single service
catalog item is treated as a bundle containing one service resource).

Each *child* (service resource) miq\_request\_task also has a
`miq_request_tasks` assocation containing the VM provisioning tasks
associated with creating the actual VMs for the service resource. This
*miq\_request\_task* is provider-specific.

It is to the second level of miq\_request\_task (also known as the
*grandchild task*) that we must pass the service dialog values that
affect the provisioning of the VM (such as `:vm_memory` or
`:vm_target_name`).

([Service Objects](../service_objects/chapter.asciidoc) discusses the
service object structure in more detail)

## Accessing the Service Dialog Options

If a service dialog has been used in the creation of an automation
request (either from a button or from a service), then the key/value
pairs from the service dialog are added to the request and subsequent
task objects. These are available in two places; as individual keys
accessible from `$evm.root`, and from the task object’s options hash as
the `:dialog` key.

``` ruby
$evm.root['service_template_provision_task'].options[:dialog] = \
           {
           "dialog_option_0_service_name"        => "New Server",
           "dialog_option_0_service_description" => "My New Server",
           "dialog_option_0_vm_name"             => "rhel7srv023",
           "dialog_tag_0_department"             => "engineering",
           "request"                             => "clone_to_service"
           }
```

or

``` ruby
$evm.root['dialog_option_0_service_description'] = My New Server
$evm.root['dialog_option_0_service_name'] = New Server
$evm.root['dialog_option_0_vm_name'] = rhel7srv023
$evm.root['dialog_tag_0_department'] = engineering
```

Accessing the dialog options from `options[:dialog]` is easier when we
don’t necessarily know the option name.

### ConfigureChildDialog

When we have several generations of child task object (as we do when
provisioning VMs from a service), we also need to pass the dialog
options from the parent object (the service template provision task), to
the various child objects, otherwise they won’t be visible to the
children.

This is generally done at the **configurechilddialog** state of the
state machine. In the *default* instance of the
*ServiceProvision\_Template* state machine this state is not used, but
we can add our own instance/method if we wish to use this functionality.

If we do decide to add our own method at this stage, we can insert the
key/value pairs from the service dialog into the `options[:dialog]` hash
of a child task object using the `set_dialog_option` method.

For example:

``` ruby
stp_task = $evm.root["service_template_provision_task"]
vm_size = $evm.root['dialog_vm_size']
stp_task.miq_request_tasks.each do |child_task|
  case vm_size
  when "Small"
    memory_size = 4096
  when "Large"
    memory_size = 8192
  end
  child_task.set_dialog_option('dialog_memory', memory_size)
end
```

This enables the child and grandchild virtual machine provision
workflows (which run through the standard VM provision state machine
that we have already studied) to access their own task object
`options[:dialog]` hash, and set the custom provisioning options
accordingly.

## VM Naming for Services

Although not immediately obvious, the service provision state machine is
run in *task* context, so any access control group profile processing,
including naming and approval, has already taken place by the time any
of our state machine methods run (we have
`$evm.root['service_template_provision_task']` rather than
`$evm.root['service_template_provision_request']`).

As we’re working in the task context of the provisioning process, the
input variables to the naming process - `:vm_name`, `:vm_prefix`, and so
on - are of no use to us (see [VM Naming During
Provisioning](../vm_naming_during_provisioning/chapter.asciidoc)). The
naming process has already been run; they will not be referenced again.

We can, however, directly update the `:vm_target_name` and
`:vm_target_hostname` values in the task object’s options hash at any
point before the **Provision** state of the *VMProvision\_VM* state
machine, like so:

``` ruby
task.set_option(:vm_target_name, "server001")
task.set_option(:vm_target_hostname, "server001")
```

Unfortunately at this stage we don’t have the ability to add the "$n{2}"
style syntax to our VM name either, hoping that the Automate Engine will
assign us the next unique number. If we wanted to guarantee uniqueness
we’d have to use something like the following code:

``` ruby
for i in (1..999)
  new_vm_name = "#{vm_prefix}#{function}#{i.to_s.rjust(2, "0")}#{suffix}"
  break if $evm.vmdb('vm_or_template').find_by_name(new_vm_name).blank?
end
```

This loop iterates through all numbers from 1 to 999, appending each
number as a zero-padded three digit suffix to the virtual machine name
prefix part. The script performs a service model lookup of a
`vm_or_template` object containing that name/suffix combination, and if
a virtual machine of that name doesn’t exist, the loop exits with the
variable `new_vm_name` set accordingly.

## Summary

This has been a brief overview of the service provisioning state
machine, showing its relative simplicity.

One of the main tasks of the state machine is to pass values from the
service dialog into the provisioning workflow, and we’ve seen how to
navigate down the three generations of task object involved in a service
provision operation in order to achieve this. Two out-of-the-box state
machine instances have been created to simplify this task for us, and we
will study those in the next chapter.
