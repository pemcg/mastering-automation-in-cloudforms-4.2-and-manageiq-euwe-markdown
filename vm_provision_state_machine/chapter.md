# The Provisioning State Machine

So far in Part II we have studied the access control group-specific
processing related to provisioning a virtual machine. We have seen how
approval and quota are handled, and how the entries from the WebUI
provisioning dialog are added to the provisioning request and task
objects in the options hash.

The common workflow for provisioning virtual machines is handled by the
*VM provision state machine*.

The virtual machine and instance provisoning workflows are each
controlled by a VM provision state machine in their respective
*Infrastructure* and *Cloud* namespaces. These state machines define the
steps in the virtual machine provisioning workflow, and contain flexible
preprovision and postprovision processing options. Instances run as part
of this state machine have access to the provisioning task object via
`$evm.root['miq_provision']`.

## State Machine Schema

The VM provision state machine
(*{Cloud/Infrastructure}/VM/Provisioning/StateMachines/VMProvision\_VM*)
Class schema contains a number of states (see [figure\_title](#i1)).

![Fields of the VMProvision\_VM state machine](images/ss1.png)

​  

Several of these states (such as **RegisterCMDB** or **RegisterAD**)
contain no out-of-the-box values, but are there as placeholders should
we wish to add the functionality to our own customised Instance.

Some states (such as **PreProvision**) have values that include an
appended message, for
    example:

    ...StateMachines/Methods/PreProvision#${/#miq_provision.source.vendor}

The message is selected at runtime from a variable substitution for
`#${/#miq_provision.source.vendor}`, and allows for the dynamic
selection of provider-specific processing options (in this case allowing
for alternative preprovisioning options for VMware, RedHat, Microsoft,
Amazon or OpenStack).

## Filling the Blanks

We can copy the VM provision state machine into our own domain, and add
instance URIs to any of the blank states as required, or extend the
state machine by inserting new states. A common addition is to add a
method at the **AcquireIPAddress** step to retrieve an IP address from a
corporate IPAM solution such as an Infoblox appliance. Once retrieved,
the IP address is inserted into the *task’s* options hash using the
`set_option` method, like so:

``` ruby
$evm.root['miq_provision'].set_option(:ip_addr, allocated_ip_address)
```

## Summary

The VM provisioning state machine is one of the most complex that we
find in CloudForms and ManageIQ. There are versions of this state
machine in both the /Infrastructure and /Cloud namespaces, and they
orchestrate the provisioning steps into their respective providers.

The state machines are designed to be extensible, however, and we’ll
develop this concept in the next chapter, where we’ll copy the state
machine to our own domain and extend it to add a second disk as part of
the provisioning process.
