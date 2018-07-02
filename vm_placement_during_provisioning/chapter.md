# Virtual Machine Placement During Provisioning

The process of deciding where in our virtual infrastructure to position
a new virtual machine - the hypervisor or cluster, and datastore - is
another step that can be automated as part of the VM provisioning
workflow. We might wish to locate VMs on the cluster with the lightest
current load for example, or restrict visibility of selected datastores
to some of our users.

CloudForms and ManageIQ refer to this process as *placement*, and there
is a corresponding **Placement** stage in the *VMProvision\_VM* state
machine.

In this chapter we’ll look at the various options for automating
placement, and how we can create customised placement algorithms to suit
our own requirements.

## Placement Methods

There are several alternative placement methods that we can use
out-of-the-box to determine where to place our new virtual machines, for
example there are three in the *ManageIQ* domain (see
[figure\_title](#i1)).

![Placement methods in the ManageIQ domain](images/ss2.png)

​  

The default value for the **Placement** stage in the
*VMProvision\_VM/template* state machine is as
    follows:

    /Infra.../VM/Provisioning/Placement/default#${/#miq_provision.source.vendor}

We can see that this URI includes a *message* component at the end,
which corresponds to the runtime value of the
`${/#miq_provision.source.vendor}` attribute. This is the string value
for the provider type that we are provisioning into.

If we look at the schema fields in the
*/Infrastructure/VM/Provisioning/Placement/default* placement instance,
we see that the messages correspond to the same provider type string:
'redhat', 'vmware' or 'microsoft' (see [figure\_title](#i2)).

![The fields of the default placement instance](images/ss1.png)

​  

This is a neat way of handling provider-specific placement requirements
in an automated manner. The message is dynamically translated for each
provisioning operation, and this selects the correct placement method.

## Method Description

The *redhat\_best\_fit\_cluster* method just places the new VM into the
same cluster as the source template. The other two methods select the
host with the least running VMs, and most available datastore space.

## Customising Placement

As part of the added value that CloudForms brings over ManageIQ, the
*RedHat* domain includes improved placement methods that we can
optionally use (see [figure\_title](#i3)).

![Placement methods in the RedHat domain](images/ss4.png)

​  

The \_with\_scope methods allow us to apply a tag from the `prov_scope`
(provisioning scope) tag category to selected hosts and datastores. This
tag indicates whether or not they should be included for consideration
for automatic VM placement. The `prov_scope` tag should be `all`, or the
name of a CloudForms user group. By tagging with a group name, we can
direct selected workloads (such as developer VMs) to specific hosts and
datastores.

The vmware\_best\_fit\_with\_tags method considers any host or datastore
tagged with the same tag as the provisioning request; that is, selected
from the Purpose tab of the provisioning dialog.

All three *RedHat* domain methods also allow us to set thresholds for
datastore usage in terms of utilization percentage and number of
existing VMs when considering datastores for placement.

### Using Alternative Placement Methods

To use the *RedHat* domain placement methods (or any others that we
choose to write), we copy the
*ManageIQ/Infrastructure/VM/Provisioning/Placement/default* instance
into our own domain and edit the value for the `redhat`, `vmware`, or
`microsoft` schema fields as appropriate to specify the name of our
preferred method.

![Editing the Placement/default instance](images/ss3.png)

​  

For example, if we wished to use the RHEV placement method from the
*RedHat* domain we would set the `redhat` schema field value to be
`redhat_best_placement_with_scope`.

## Summary

We can see that we have a lot of per-provider control of the placement
options available to us when we provision a virtual machine. We can also
add our own placement methods to take into account our own specific
requirements if we wish.

When we start working with custom placement methods, we also need to
take into account the infrastructure components that a user can see from
their role-based access control filters. When we configure CloudForms or
ManageIQ access control groups, we can set optional *assigned filters*
to selected hosts and clusters. We can also restrict a group’s
visibility of infrastructure components to those tagged with specific
tags. If we use assigned filters in this way, we need to ensure that our
placement logic doesn’t select a host, cluster or datastore that the
user doesn’t have RBAC permission to see, otherwise the provisioning
operation will fail.

### Further Reading

[Placement Profile – Best Fit Cluster using
Tags](http://cloudformsblog.redhat.com/2013/09/06/placement-profile-best-fit-cluster-using-tags/)
