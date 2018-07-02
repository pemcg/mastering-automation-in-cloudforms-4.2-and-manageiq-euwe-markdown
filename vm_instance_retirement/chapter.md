# Virtual Machine and Instance Retirement

CloudForms and ManageIQ are virtual machine and instance lifecycle
management tools, and so far we have concentrated on the provisioning
phase of that lifecycle. The products also have virtual machine
retirement workflows that lets us manage the retirement and deletion of
our virtual machines or instances, both from the provider and from the
CloudForms/ManageIQ VMDB if required.

The retirement process allows us to treat differently virtual machines
that were provisioned by CloudForms or ManageIQ, and those that might
have already existed on the provider infrastructure, or have been
provisioned by the native infrastructure manager.

> **Note**
> 
> We may wish to keep the VMDB record of a virtual machine long after
> its deletion from the provider, for recording and auditing purposes.

In this chapter we’ll examine the retirement process for virtual
machines and instances.

## Initiating Retirement

Virual machine or instance retirement is initiated from the
**Lifecycle** menu on the VM details page (see [figure\_title](#i1)).

![Virtual machine or instance retirement menu](images/ss1.png)

​  

Clicking on **Retire this VM** raises a **request\_vm\_retire** event
that begins a chain of relationships through the datastore:

  - **request\_vm\_retire** →
    
      - */System/Event/MiqEvent/POLICY/request\_vm\_retire* →
    
      - */{Cloud,Infrastructure}/VM/Lifecycle/Retirement*
        →
    
      - */{Cloud,Infrastructure}/VM/Retirement/StateMachines/VMRetirement/{Default,Unregister}*

The relationship from *Lifecycle/Retirement* forwards the event
processing into the preferred virtual machine retirement state machine:
*Default* for cloud instances, *Default* or *Unregister* for
infrastructure virtual machines. We can select our preferred
infrastructure state machine (*Default* is the default) by copying the
*/Infrastructure/VM/Lifecycle/Retirement* instance to our own domain and
editing accordingly.

## Retirement-Related Attributes and Methods

A virtual machine object has a number of retirement-related methods:

    $evm.root['vm'].retire_now
    $evm.root['vm'].start_retirement
    $evm.root['vm'].finish_retirement
    $evm.root['vm'].retiring?
    $evm.root['vm'].retired?
    $evm.root['vm'].error_retiring?
    $evm.root['vm'].retirement_state=
    $evm.root['vm'].retirement_warn=
    $evm.root['vm'].retires_on=

> **Note**
> 
> CloudForms 4.2/ManageIQ *Euwe* also added a new method
> `extend_retires_on` so that the retirement date can be extended.

It also has several attributes:

    $evm.root['vm'].retired = nil
    $evm.root['vm'].retirement_last_warn = nil
    $evm.root['vm'].retirement_requester = nil
    $evm.root['vm'].retirement_state = nil
    $evm.root['vm'].retirement_warn = nil
    $evm.root['vm'].retires_on = nil

During the retirement process some of these are set to indicate
progress:

    $evm.root['vm'].retirement_requester = admin   (type: String)
    $evm.root['vm'].retirement_state = retiring   (type: String)

and completion:

    $evm.root['vm'].retired = true   (type: TrueClass)
    $evm.root['vm'].retirement_requester = nil
    $evm.root['vm'].retirement_state = retired   (type: String)
    $evm.root['vm'].retires_on = 2015-12-10   (type: Date)

## VM Retirement State Machine

The VM retirement state machines(s) undo many of the operations
performed by the VM provision state machine. They allow us to optionally
deactivate a CI record from a CMDB; unregister from DHCP, Active
Directory, and DNS; and release both MAC and IP addresses (see
[figure\_title](#i2)).

![Fields of the VM retirement state machine](images/ss2.png)

​  

### StartRetirement

The *StartRetirement* instance calls the *start\_retirement* state
machine method, which checks whether the VM is already in state
'retired' or 'retiring', and if so it aborts. If in neither of these
states it calls the VM’s `start_retirement` method, which sets the
`retirement_state` attribute to 'retiring'.

### PreRetirement/CheckPreRetirement

The state machine allows us to have provider-specific instances and
methods for these stages. The out-of-the-box infrastructure
*PreRetirement* instance runs a vendor-independant *pre\_retirement*
method that just powers off the VM. The out-of-the-box cloud
*PreRetirement* instance runs the appropriate vendor-specific
*pre\_retirement* method, i.e. *amazon\_pre\_retirement*,
*azure\_pre\_retirement* or *openstack\_pre\_retirement*.

*CheckPreRetirement* checks that the power off has completed. The cloud
versions have corresponding vendor-specific *check\_pre\_retirement*
methods.

### RemoveFromProvider/CheckRemovedFromProvider

The **RemoveFromProvider** state allows us some flexibility in handling
the actual removal of the VM, and this is where the *Default* and
*Unregister* state machines differ.

#### Default

The **RemoveFromProvider** state of the *Default* state machine links to
the *RemoveFromProvider* instance, which calls the
*remove\_from\_provider* state machine method, passing the
`removal_type` argument of `'remove_from_disk'`. This checks whether the
VM was provisioned from ManageIQ (`vm.miq_provision` is not **nil** ),
**or** if the VM is tagged with **lifecycle/retire\_full**. If either of
these is true it fully deletes the VM from the underlying provider,
including the disk image. Having done so it sets a boolean state
variable `vm_removed_from_provider` to `true`.

If neither of these checks returns **true**, no action is performed.

#### Unregister

The **RemoveFromProvider** state of the *Unregister* state machine links
to the *UnregisterFromProvider* instance, which calls the
*remove\_from\_provider* state machine method, passing the
`removal_type` argument of `'unregister'`. This checks whether the VM
was provisioned from ManageIQ (`vm.miq_provision` is not **nil** ),
**or** if the VM is tagged with **lifecycle/retire\_full**. If either of
these is true it deletes the VM from the underlying provider, but
retains the VM’s disk image, allowing the VM to be re-created if
required in the future. Having done so it sets a boolean state variable
`vm_removed_from_provider` to `true`.

If neither of these checks is true, no action is performed.

### FinishRetirement

The *FinishRetirement* instance calls the *finish\_retirement* state
machine method that sets the following VM object attributes:

    :retires_on       => Date.today
    :retired          => true
    :retirement_state => "retired"

It also raises a **vm\_retired** event that can be caught by an Automate
action or control policy.

### DeleteFromVMDB

The *DeleteFromVMDB* instance calls the *delete\_from\_vmdb* state
machine method that checks for the state variable
`vm_removed_from_provider`, and if found (and true) it removes the
virtual machine record from the VMDB. With CloudForms 4.0/ManageIQ
*Capablanca*, this state was enabled by default, meaning that all VM
entries were deleted. With 4.1/*Darga* this entry is commented out, and
so we see retired VMs in the WebUI as having an 'A' in their tile
quadrant, indicating their archived status.

## Summary

This chapter shows that retirement is a more complex process than simply
deleting the virtual machine. We must potentially free up resources that
were allocated when the VM was created, such as an IP address. We might
need to delete a CI record from a CMDB, unregister from Active
Directory, or even keep the VMDB object inside CloudForms or ManageIQ
for auditing purposes.

> **Note**
> 
> If the VM remains in the VMDB in an archived state, it will still be
> returned as a valid VM if we run a `$evm.vmdb(:vm).all` call. This can
> get even more confusing if we have subsequently re-provisioned a VM
> with the same name, as `$evm.vmdb(:vm).find_by_name(vm_name)` may
> return the 'wrong' object to us. Fortunately there is a `vm.archived`
> boolean attribute that we can check to determine whether a VM object
> is active or archived.

As we have seen, the retirement workflow allows us to fine-tune all of
these options, and handle retirement in a manner that suits us.

## Further Reading

[Provisioning Virtual Machines and Hosts Chapter 6 -
Retirement](https://access.redhat.com/documentation/en/red-hat-cloudforms/4.0/provisioning-virtual-machines-and-hosts/chapter-6-retirement)

[Deleting VMs from Foreman during
Retirement](http://www.jung-christian.de/2015/06/delete-vm-from-foreman-during-retirement/)
