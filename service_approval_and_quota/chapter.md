# Approval and Quota

We discovered in [Provisioning
Approval](../provisioning_approval/chapter.asciidoc) and [Provisioning
Quota Management](../provisioning_quota_management/chapter.asciidoc)
that the virtual machine provisioning process includes an approval stage
- to allow administrators to optionally approve large VM requests - and
a quota checking stage that enforces quotas applied to access control
groups or tenants. We also learnt that these workflows were triggered
from *MiqProvisionRequest\_created* and *MiqProvisionRequest\_starting*
policy Instances. When we create a virtual machine from a service
catalog item however, our request object is different, so we cannot rely
on the existing approval and quota workflow triggers.

This chapter examines the approval and quota workflows when we provision
a virtual machine from a service catalog.

## Triggering Events

When we order a new virtual machine from a service catalog, our request
still needs to be approved and matched against our current tenant or
group quota. As with virtual machine provisioning, each of the
corresponding service provision workflows is triggered by the
**request\_created** and **request\_approved** events, but the request
object type is different. It is now a
service\_template\_provision\_request.

## Approval

The approval process for a service provision request starts with the
*/System/Policy/ServiceTemplateProvisionRequest\_created* Instance being
run as a result of a **request\_created** event. This Instance contains
two relationships, **rel5** and **rel6**.

The **rel5** relationship performs a service provisioning profile lookup
to read the value of the **auto\_approval\_state\_machine** attribute,
which by default is *ServiceProvisionRequestApproval* for a service
provision request.

The **rel6** relationship runs the *Default* Instance of this state
machine.

![ServiceProvisionRequestApproval state machine instance and
methods](images/ss2.png)

​  

The schema for the *ServiceProvisionRequestApproval/Default* state
machine is shown in [figure\_title](#i2).

![Fields of the ServiceProvisionRequestApproval/Default state
machine](images/ss1.png)

​  

The methods *pending\_request* and *approve\_request* are the same as
their counterparts for virtual machine provisioning approval. The
default value of *validate\_request* does nothing, so this state machine
instance will auto-approve all service provisioning requests.

### Customising Approval

If universal auto-approval is not the required behaviour, we can copy
the *ServiceProvisionRequestApproval/Default* state machine methods to
our own domain and edit them as necessary. \[1\]

## Quota

Quota checking for a service provision request uses the same
consolidated quota mechanism as described in [Provisioning Quota
Management](../provisioning_quota_management/chapter.asciidoc). The
quota checking process for a service provision request starts with the
*/System/Policy/ServiceTemplateProvisionRequest\_starting* Instance
being run as a result of a **request\_starting** event. This policy
Instance runs the */System/CommonMethods/QuotaStateMachine/quota* state
machine from its **rel2** relationship.

### Email

In ManageIQ there are no email instances that handle the sending of
service "approval denied/pending" and "quota exceeded" emails. We would
have to implement this functionality ourselves if we wish to send such
emails.

CloudForms does provide suitable email instances in the RedHat domain,
but by default they are not wired in to any policy instances. If we wish
to use them we can add a policy instance called
*/System/Policy/ServiceTemplateProvisionRequest\_denied* to our own
domain. This should contain a relationship to the
*/Service/Provisioning/Email/ServiceTemplateProvisionRequest\_Denied*
email instance in the RedHat domain.

## Summary

We have seen how the approval and quota checking mechanism for services
mirrors that for virtual machines, but uses different policy instances
to trigger the workflows. The out-of-the-box approval workflow
auto-approves all service requests, but we can copy the instance to our
own domain to customise the behaviour if we wish.

In practice we rarely need to customise these workflows. As
virtualisation administrators, when we provide a self-service catalog to
our users, we generally accept the delegation of control and degree of
responsibility that we pass to our users. This is after all one of the
many benefits of implementing an Infrastructure as a Service cloud
model. We almost certainly allocate quotas, but we rarely need to
implement per-order approval as well. The default behavour of
auto-approval of all service requests is valid in most situations.

1.  Nick Catling from Red Hat has written a nice example of how we can
    customise service provisioning approval on a per-group basis if we
    wish. The code is available on his github repository:
    <https://github.com/supernoodz/CloudForms/tree/master/Approval>
