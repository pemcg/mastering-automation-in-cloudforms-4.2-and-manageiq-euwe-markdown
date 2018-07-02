# Approval

A newly provisioned virtual machine consumes resources in a virtual
infrastructure, and potentially costs money in a public cloud. To
control the consumption of resources and keep cloud costs in check, an
approval stage is built into the virtual machine and instance
provisioning workflow. By default requests for single small virtual
machines are auto-approved, but attempts to provision larger or multiple
VMs are redirected for administrative approval.

This chapter describes the approval process, and shows how we can
fine-tune the approval thresholds based on number of VMs, number of
CPUs, or amount of memory in the request.

## Approval Workflow

The provision request approval workflows are triggered by the
**request\_created** and **request\_pending** events (see
[figure\_title](#i5)).

![Event-triggered provision request approval
workflows](images/approval_workflow.png)

​  

### Request Created Event

The approval workflow for a virtual machine provision request is entered
as a result of the */System/Policy/MiqProvisionRequest\_created* policy
instance being run from a **request\_created** event. This policy
instance contains two relationships, **rel5** and **rel6**.

The **rel5** relationship performs a group profile lookup to read the
value of the **auto\_approval\_state\_machine** attribute, which by
default is *ProvisionRequestApproval* for an infrastructure virtual
machine or cloud instance provision request.

The **rel6** relationship runs the *Default* instance of this state
machine (see [figure\_title](#i1)).

![The ProvisionRequestApproval state machine instances and
methods](images/ss1.png)

​  

The *Default* instance of the ProvisionRequestApproval state machine has
the Field values shown in [figure\_title](#i2).

![The ProvisionRequestApproval/Default instance](images/ss2.png)

​  

This instance will auto-approve any VM provisioning request containing a
single VM, but requests for more than this number will require explicit
approval from an Administrator, or anyone in a group with the role
**EvmRole-approver** (or equivalent).

### Methods

The ProvisionRequestApproval state machine uses three methods to perform
the validation.

#### validate\_request

The *validate\_request* method is run from **On Entry** field of the
**ValidateRequest** state. It checks the provisioning request against
the schema **max\_** attributes, and if the request doesn’t exceed these
maxima, the method exits cleanly. If the request does exceed the maxima,
the method sets `$evm.root['ae_result'] = 'error'` and a reason message
before exiting.

#### pending\_request

The *pending\_request* method is run from the **On Error** field of the
**ValidateRequest** state. This will be run if *validate\_request* exits
with `$evm.root['ae_result'] = 'error'`. The method is simple, and
merely raises a **request\_pending** event to trigger the
*MiqProvisionRequest\_pending* policy instance:

``` ruby
# Raise automation event: request_pending
$evm.root["miq_request"].pending
```

#### approve\_request

The *approve\_request* method is run from the **On Entry** field of the
**ApproveRequest** state. This will be run if *validate\_request* exits
cleanly. This is another very simple method that merely auto-approves
the request:

``` ruby
# Auto-Approve request
$evm.log("info", "AUTO-APPROVING")
$evm.root["miq_request"].approve("admin", "Auto-Approved")
```

### Request Pending Event

If the *ProvisionRequestApproval* state machine doesn’t approve the
request, it calls `$evm.root["miq_request"].pending`, which triggers a
**request\_pending** event. This is the trigger point into the second
workflow through the *MiqProvisionRequest\_pending* policy instance.
This instance sends the emails to the requester and approver, notifying
that the provisioning request has not been auto-approved, and needs
manual approval.

## Overriding the Defaults

We can copy the *Default* instance (including path) to our own domain
and change or set any of the auto-approval schema Attributes - that is,
**max\_cpus**, **max\_vms**, **max\_memory** or
**max\_retirement\_days**. Our new values will then be used when the
next virtual machine is provisioned.

### Template Tagging

We can also override the auto-approval **max\_**\* values stored in the
*ProvisionRequestApproval* state machine on a per-template basis, by
applying tags from one or more of the following tag categories to the
template:

| Tag Category Name           | Tag Category Display Name          |
| --------------------------- | ---------------------------------- |
| prov\_max\_cpu              | Auto Approve - Max CPU             |
| prov\_max\_memory           | Auto Approve - Max Memory          |
| prov\_max\_retirement\_days | Auto Approve - Max Retirement Days |
| prov\_max\_vm               | Auto Approve - Max VM              |

If a template is tagged in such a way, then any VM provisioning request
*from* that template will result in the template’s tag value being used
for auto-approval considerations, rather than the attribute value from
the schema.

## VM Provisioning-Related Email

There are four email instances with corresponding methods that are used
to handle the sending of VM provisioning-related emails. The instances
each have the attributes **to\_email\_address**,
**from\_email\_address** and **signature** which we can (and should)
customise, after copying the instances to our own domain.

![Copying and editing the approval email schema fields](images/ss3.png)

​  

Three of the instances are approval-related. The **to\_email\_address**
value for the *MiqProvisionRequest\_Pending* instance should contain the
email address of a user (or mailing list) who is able to login to the
ManageIQ appliance as an Administrator or as a member of a group with
the **EvmRole-approver** role or equivalent (see [figure\_title](#i4)).

## Summary

This chapter shows how the virtual machine provisioning workflow allows
for the approval stage to filter requests for large virtual machines,
while auto-approving small requests. This simplifies our life as
virtualisation administrators considerably. It allows us to retain a
degree of control over large resource requests, even allowing us to
define our own concept of 'large' by setting schema attributes
accordingly. It also allows us to delegate responsibility for small
virtual machine requests to our standard users. Automation allows us to
intervene for the exceptional cases, yet auto-approve the ordinary
"business as usual" requests.

We have also seen how we can fine-tune these approval thresholds on a
per-template basis, so that if some of our users have valid reasons to
provision large virtual machines from specific templates, we can allow
them to without interruption.

The approval state machine and methods are a good example of the utility
of defining thesholds as schema attributes or by using tags. We can
customise the approval process to our own requirements without the need
to write or edit any Ruby code.
