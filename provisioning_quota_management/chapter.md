# Quota Management

In the last chapter we saw how every virtual machine or instance
provisioning request involves an approval process, and that requests for
larger VMs would normally require administrative approval. Even with
auto-approval thresholds set at their low defaults however, our users
could still over time create a large number of small virtual machines,
and consume our virtual infrastructure resources and increase cloud
costs.

For this reason CloudForms and ManageIQ also allow us to establish
quotas on tenants or user groups. Quotas can be set for number of
virtual machines, number of CPUs, amount of memory, or quantity of
storage 'owned' by the tenant or group. If a virtual machine
provisioning request would result in the quota being exceeded, the
request is rejected and the requesting user is emailed.

Quotas are not enabled by default, but they are simple to turn on and
configure.

## Quota in CloudForms 4.0

Quota management has been completely rewritten for CloudForms
4.0/ManageIQ *Capablanca*. Prior to this release, quota management for
cloud instance, infrastructure virtual machine, and service provisioning
was handled in different places under the respective */Cloud*,
*/Infrastructure*, and */Service* namespaces. In *Capablanca* these have
been consolidated these under */System/CommonMethods* in the Automate
Datastore. (see [figure\_title](#i1)).

![Quota classes, instances and methods](images/ss1.png)

​  

The *ManageIQ/System/CommonMethods/QuotaStateMachine/quota* state
machine instance has the field values shown in [figure\_title](#i2).

![Schema of the quota state machine](images/ss2.png)

​  

We can see that quota processing follows a simple workflow of:

1.  Determine the quota source

2.  Determine the quota limits assigned to that source

3.  Determine the resources currently used by that source

4.  Determine the new resources requested by the source

5.  Validate whether the new requested amount would exceed quota

### Quota Source

A new concept with the re-implemented quota management mechanism is that
of *quota source*. This is the entity to which the quota is applied, and
by default is a tenant. Tenant quotas can be edited in the WebUI under
**Configure → Configuration → Access Control → Tenants → *tenant*** (see
[figure\_title](#i3)).

![Setting quotas for a tenant](images/ss3.png)

​  

The tenant object keeps track of allocated values in virtual columns:

    --- virtual columns follow ---
    $evm.root['tenant'].allocated_memory = 48318382080   (type: Fixnum)
    $evm.root['tenant'].allocated_storage = 498216206336   (type: Fixnum)
    $evm.root['tenant'].allocated_vcpu = 23   (type: Fixnum)
    $evm.root['tenant'].provisioned_storage = 546534588416   (type: Fixnum)

#### Alternative Quota Sources

If we wish to use an alternative quota source, we can copy the *quota*
state machine instance to our own domain, and edit
**quota\_source\_type** attribute. If we set this to be 'group', the
provisioning user’s group will be used as the quota source, and quota
handling will be handled in the pre-CloudForms 4.0 way. We can set quota
in two ways.

#### Defining Quota in the State Machine Schema (the model)

We can set generic warn and max values for **VM Count**, **Storage**,
**CPU** and **Memory**, by copying the
*ManageIQ/System/CommonMethods/QuotaStateMachine/quota* instance into
our Domain, and editing any of the eight schema attributes.

Quotas defined in the model in this way apply to all instances of the
quota source (e.g. all groups)

#### Defining Quota Using Tags

We can override the default model attributes by applying tags from one
or more of the following tag categories to individual quota source
entities (e.g. individual
groups):

| Tag category name    | Tag category display name | Pre-exists          |
| -------------------- | ------------------------- | ------------------- |
| quota\_warn\_vms     | Quota - Warn VMs          | No; must be created |
| quota\_max\_vms      | Quota - Max VMs           | No; must be created |
| quota\_warn\_storage | Quota - Warn Storage      | No; must be created |
| quota\_max\_storage  | Quota - Max Storage       | Yes                 |
| quota\_warn\_cpu     | Quota - Warn CPUs         | No; must be created |
| quota\_max\_cpu      | Quota - Max CPUs          | Yes                 |
| quota\_warn\_memory  | Quota - Warn Memory       | No; must be created |
| quota\_max\_memory   | Quota - Max Memory        | Yes                 |

If a group is tagged in such a way, then any VM or service provisioning
request from any group member is matched against the currently allocated
CPUs, memory or storage for the group.

If quotas are defined both in the model and with tags, the tagged value
takes priority.

## Quota Workflow

The quota checking process for a virtual machine or instance provision
request is triggered by a **request\_starting** event (see
[figure\_title](#i4))

![Event-triggered provision request quota
workflow](images/quota_workflow.png)

​  

This event policy is handled by the
*/System/Policy/MiqProvisionRequest\_starting* policy instance, which
has a single **rel5** relationship that calls the
*/System/CommonMethods/QuotaStateMachine/quota* state machine.

If the provisioning request would result in the quota being exceeded,
then the request is rejected, and the requesting user is emailed through
the
*/{Infrastructure,Cloud}/VM/Provisioning/Email/MiqProvisionRequest\_Denied*
email class.

If the request is within the quota then the workflow simply exits.

## Summary

Quotas allow us to maintain a degree of control over the depletion of
our expensive virtualisation resources, while still empowering our users
to create their own virtual machines or instances.

Quotas can be applied to access control groups or tenants. A quota
allocated to a tenant can be further subdivided between any child
tenants or tenant projects. For example we might have a tenant
representing our application development team, and they might have
tenant projects representing applications currently under development.
We can allocate the **EvmRole-tenant\_quota\_administrator** access
control role to a virtualisation administrator, who can then further
sub-divide the development team’s quota between projects as requested.

When we apply quotas to access control groups, we can additionally tag
the groups with *warn* and *max* threshold tags on a per-group basis to
fine-tune the quota allocation.

### Further Reading

[Consolidated Service/VM quota
validation](https://github.com/ManageIQ/manageiq/pull/4338)
