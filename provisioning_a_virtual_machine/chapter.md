# Provisioning a Virtual Machine

One of the most common things that we do as cloud or virtualisation
administrators is to create new virtual machines or instances. We get
used to the procedure; picking a template, selecting a target cluster,
datastore and network, choosing a suitable name. These are generally
manual steps, but CloudForms and ManageIQ have an out-of-the-box virtual
machine provisioning workflow that automates the process.

There are many steps involved in automatically provisioning a virtual
machine. The provisioning workflow has been designed to be extremely
flexible, and allows a great deal of customisation based on tagging, the
requesting user’s group membership, and the destination Provider type
(e.g. RHEV, VMware, OpenStack, etc.).

## The Provisioning Process

The virtual machine provisioning process starts with a user (the
*requester*) selecting either **Provision VMs** from under the
**Infrastructure → Virtual Machines → Lifecycle** button group, or
**Provision Instances** or from under the **Cloud → Instances →
Lifecycle** button group(see [figure\_title](#i1)).

![Initiating a provisioning operation](images/ss1.png)

​  

This takes us into a selection dialog where we pick an image or template
to provision from, and click the **Continue** button (see
[figure\_title](#i2)).

![Selecting the provisioning source template](images/ss2.png)

​  

Once we click **Continue**, we enter into the virtual machine
provisioning workflow, starting with information retrieved from the
*profile* and moving into the *state machine*.

## Group-Specific Considerations, and Common Processing

Provisioning a virtual machine or instance involves many separate
decisions, and steps that come together to form the VM provisioning
workflow.

Some of these steps need to be performed or evaluated within the context
of the requesting user’s access control group membership, such as the
choice of provisioning dialog to present to the user in the WebUI. We
may for example, wish to customise the WebUI dialog to present a
restricted set of options to certain groups of users (see also [The
Provisioning Dialog](../the_provisioning_dialog/chapter.asciidoc)). We
can decide to apply quotas to access control groups, or create specific
customisations such as group-specific virtual machine naming schemes.
Group-specific processing is typically performed in *Request* context,
before the *Tasks* are created (see [Requests and
Tasks](../requests_and_tasks/chapter.asciidoc) for a description of
requests and tasks).

Other steps in the virtual machine provisioning workflow are common to
all virtual machine or instance provisioning operations. These typically
include the allocation of an IP address, registration with a CMDB, or
emailing the requester that the provision has completed for example.

The group-specific *Provisioning Profile* contains the per-group
attributes, instance and state machine names that are used in processing
the provisioning *Request*, and in preparing the provisioning *Task(s)*.

The more generic sequence of common steps involved in provisioning a
virtual machine or instance come from the *VM Provisioning State
Machine*. This is processed in the context of the provisioning *Task*.

## Summary

This short chapter has introduced the group-specific provisioning
profile and the more generic VM provisioning state machine that combine
to form the virtual machine provisioning workflow.

In the following chapters, we will examine these in more detail,
starting with the provisioning profile.
