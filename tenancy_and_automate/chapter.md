# Tenancy and Automate

Since CloudForms 4.0/ManageIQ *Capablanca*, all user operations have
been performed within the context of a tenant, even if we don’t add any
new tenants of our own. The default (i.e. top-level) tenant is known as
*tenant zero* (although it actually has a tenant ID of 1), and all of
the out-of-the-box groups are members of this tenant. We can of course
define new tenants and groups within our organization, which will then
form a tenant hierarchy under tenant zero.

![A tenant hierarchy within an
organization](images/tenant_hierarchy.svg)

​  

Tenancy is very useful if we wish to sub-divide our organization into
multiple semi-autonomous units that we can delegate a degree of
authority to, and define quotas for. We create new RBAC groups, add our
users to these groups, and we can then assign the groups to tenants or
projects.

> **Note**
> 
> A *project* is a tenant that can have no child tenants of its own. A
> tenant object within Automate (see [The Tenant
> Object](#tenant-object)) has an attribute
> `$evm.root['tenant'].divisible` that is `true` for a normal tenant,
> but `false` for a project.

## Tenant Visibility Rules

In keeping with the organizational delegation concept, tenant users have
visibility of some infrastructure or cloud components that have been
defined or created elsewhere in the tenant hierarchy. For example if we
refer to [figure\_title](#i1), a user in the *Europe* tenant can see VM
templates or service catalogs defined in any of the *Europe*, *Sales* or
*Bit63* tenants, but not in the *Engineering*, *North America* or
*Scandinavia* tenants, nor the *Wonder\_Widget* project. The same user
can see VMs created in either of the *Europe* or *Scandinavia* tenants.

The tenant visibility rules are defined as follows:

### Our Tenant

The following objects, when created by a tenant should only be visible
within that tenant:

  - Automate requests

  - Automate tasks

### Parent Tenant

If defined in a parent tenant, the following objects should be visible
to any child tenant in its hierarchy branch:

  - Providers

  - Automate domains

  - Service catalogs

  - Service catalog items

  - VM templates or cloud images

### Child Tenant

If created in a child tenant, the following objects should be visible to
any parent tenant in the hierarchy branch:

  - Services

  - VMs & instances

If created in a child tenant, the following objects should not be
visible to any parent tenant in the hierarchy:

  - Automate domains

> **Note**
> 
> The tenant visibility rules defined above for VMs and templates only
> apply if the user’s group role has **VM & Template Access
> Restriction** set to *None*. If we set the role’s access restriction
> to *Only User or Group Owned* the user will only have visibility of
> their own group’s VMs or templates, as expected.
> 
> Tag-related RBAC works within the confines of the tenant parent/child
> hierarchy. Adding a visibility tag to a tenant’s VM does not make the
> VM visible outside of these tenant hierarchy rules, even if a group in
> a different tenant hierarchy branch is defined as having the same
> visibility tag.

## Tenancy Within Automate

All Automate operations include a tenant object that is accessible as
`$evm.root['tenant']`. This object contains the details of the tenant
context for the operation, and includes such useful information as the
current quota allocation for CPUs, storage and memory for that tenant
(see also [Provisioning Quota
Management](../provisioning_quota_management/chapter.asciidoc)).

### The Tenant Object

The following *object\_walker* extract shows the attributes, virtual
columns and associations of the tenant object. with typical values (the
object has no methods of its own):

    --- attributes follow ---
    $evm.root['tenant'].ancestry = 1   (type: String)
    $evm.root['tenant'].default_miq_group_id = 17   (type: Fixnum)
    $evm.root['tenant'].description = Engineering   (type: String)
    $evm.root['tenant'].divisible = true   (type: TrueClass)
    $evm.root['tenant'].domain = nil
    $evm.root['tenant'].id = 2   (type: Fixnum)
    $evm.root['tenant'].login_logo_content_type = nil
    $evm.root['tenant'].login_logo_file_name = nil
    $evm.root['tenant'].login_logo_file_size = nil
    $evm.root['tenant'].login_logo_updated_at = nil
    $evm.root['tenant'].login_text = nil
    $evm.root['tenant'].logo_content_type = nil
    $evm.root['tenant'].logo_file_name = nil
    $evm.root['tenant'].logo_file_size = nil
    $evm.root['tenant'].logo_updated_at = nil
    $evm.root['tenant'].name = Engineering   (type: String)
    $evm.root['tenant'].subdomain = nil
    $evm.root['tenant'].use_config_for_attributes = false   (type: FalseClass)
    --- end of attributes ---
    --- virtual columns follow ---
    $evm.root['tenant'].allocated_memory = 6442450944   (type: Fixnum)
    $evm.root['tenant'].allocated_storage = 161061273600   (type: Fixnum)
    $evm.root['tenant'].allocated_vcpu = 3   (type: Fixnum)
    $evm.root['tenant'].display_type = Tenant   (type: String)
    $evm.root['tenant'].parent_name = Bit63   (type: String)
    $evm.root['tenant'].provisioned_storage = 167503724544   (type: Fixnum)
    $evm.root['tenant'].region_description = Region 0   (type: String)
    $evm.root['tenant'].region_number = 0   (type: Fixnum)
    --- end of virtual columns ---
    --- associations follow ---
    $evm.root['tenant'].ae_domains (type: Association)
    $evm.root['tenant'].ext_management_systems (type: Association)
    $evm.root['tenant'].miq_groups (type: Association)
    $evm.root['tenant'].miq_request_tasks (type: Association)
    $evm.root['tenant'].miq_requests (type: Association)
    $evm.root['tenant'].miq_templates (type: Association)
    $evm.root['tenant'].providers (type: Association (empty))
    $evm.root['tenant'].service_templates (type: Association)
    $evm.root['tenant'].services (type: Association)
    $evm.root['tenant'].tenant_quotas (type: Association)
    $evm.root['tenant'].users (type: Association)
    $evm.root['tenant'].vm_or_templates (type: Association)
    $evm.root['tenant'].vms (type: Association)
    --- end of associations ---

All of the useful service models that we interact with when automation
scripting (such as `miq_group`, `vm`, or `service` for example) have a
`tenant_id` attribute and a `tenant` association that we can use to
determine tenant ownership, or retrieve the corresponding tenant object.

### Tenant Domains

A tenant user with an RBAC role of EvmRole-administrator or equivalent
can create a tenant-specific Automate domain. Such domains are useful
for creating tenant-specific workflows, or to override wider
organizational Automate schemes such as a VM naming policy.

A tenant domain will be visible and editable to all tenant users who
have access to the Automate Explorer. The domain will appear visible but
locked to any users in a child tenant who have access to the Automate
Explorer.

![Automate explorer view from an Engineering domain
Administrator](images/ss2.png)

​  

The domain will not be visible to any users in a parent tenant who have
access to the Automate Explorer (even if they have an RBAC role of
EvmRole-super\_administrator or equivalent).

![Automate explorer view from a Bit63 domain Super
Administrator](images/ss1.png)

​  

Tenant domains follow the same priority order as any other Automate
domains, although only unlocked domains can be re-ordered in priority.

### Writing Automate Code to be Tenant-Aware

Although the Automation Engine gives us a tenant object to refer to, the
Engine does not execute our code within the confines of our tenant’s
RBAC constraints or visibility rules\[1\]. For example
`$evm.vmdb(:vm).all` will return to us the same unfiltered list of VMs,
regardless of which tenant we call it from.

If we wish to present a list of existing VMs or available templates in a
service dialog to a tenant user, we must apply our own tenant-related
filtering to our dynamic method so that the correct lists are presented
to users in different tenants. When we define a service catalog in our
*Bit63* tenant, a user viewing the dialog in the *Europe* tenant should
see a different VM list to a user in the *Engineering* tenant, even
though they are running the same code.

Fortunately the tenant object has an `ancestry` attribute that we can
use. The ancestry values for the tenant hierarchy shown in
[figure\_title](#i1) are as follows:

| Tenant              | Tenant ID | Tenant ancestry |
| ------------------- | --------- | --------------- |
| Bit63 (tenant zero) | 1         | nil             |
| Engineering         | 2         | "1"             |
| Sales               | 3         | "1"             |
| Wonder\_Widget      | 4         | "1/2"           |
| Europe              | 5         | "1/3"           |
| North America       | 6         | "1/3"           |
| Scandinavia         | 7         | "1/3/5"         |

#### Determining Visible Ancestral Tenants

We can use the `ancestry` attribute to calculate which ancestral tenants
should be visible to our tenant, for example to determine the list of
'visible' infrastructure templates to present in a drop-down dialog
element:

``` ruby
def tenant_infra_templates(tenant_id)
  $evm.vmdb(:template_infra).where(:tenant_id => tenant_id)
end

def tenant_ancestor_ids(tenant)
  return [] if tenant.ancestry.blank?
  tenant.ancestry.split('/')
end

def ancestor_infra_templates(tenant)
  tenant_ancestor_ids(tenant).map { |t| tenant_infra_templates(t) }.flatten
end

def tenant_and_ancestor_infra_templates(tenant)
  tenant_infra_templates(tenant.id) + ancestor_infra_templates(tenant)
end
```

We can call the method as follows:

``` ruby
templates = tenant_and_ancestor_infra_templates($evm.root['tenant'])
```

#### Determining Visible Child Tenants

We can also calculate which child tenants should be visible to our
current tenant. We might wish to do this to determine a list of VMs or
services that should be visible to our tenant, for example.

``` ruby
def tenant_child_ids(tenant)
  child_ids = []
  child_ids << tenant.id.to_s  # include this tenant's ID
  $evm.vmdb(:tenant).all.each do |t|
    unless t.ancestry.blank?
      if t.ancestry.split('/').include?(tenant.id.to_s)
        child_ids << t.id.to_s
      end
    end
  end
  child_ids
end
```

We can then use the VM object’s `tenant_id` attribute to match the
virtual machines that are visible to this tenant user, as follows:

``` ruby
child_ids = tenant_child_ids($evm.root['tenant'])
vms = $evm.vmdb(:vm).all.select { |vm| child_ids.include?(vm.tenant_id.to_s) }
```

## Summary

This chapter has shown how we can use Automate to our advantage when we
sub-divide our organization into multiple tenants. We can allow child
tenants to create their own Automate domains, enabling them to implement
custom workflows, or to override enterprise-wide settings such as a VM
naming scheme or placement policy.

We have also seen how we sometimes need to take tenancy filtering into
account when we write our automation scripts - particularly for dynamic
dialog methods - in order to comply with visibility rules.

As Super Administrators we need to exercise a degree of trust when we
implement a tenant hierarchy, particularly when adding users with
EvmRole-administrator or equivalent rights to the tenant. A tenant user
with WebUI access to the Automate Explorer is able to access some
'global' Automate objects, and the output from `$evm.log` called from
any tenant is always written into the common *automation.log* file.

1.  Further tenant RBAC-enablement for Automate is in development, and
    we should get three new `$evm` methods in a future release of
    ManageIQ: `enable_rbac`, `disable_rbac` and `rbac_enabled?`
