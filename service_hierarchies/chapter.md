# Service Hierarchies

We have seen how service catalogs made up of catalog items and bundles
can simplify the process of ordering infrastructure or cloud instance
and virtual machines. Simplicity of ordering is not the only benefit of
services however.

When we order one or more virtual machines from a service catalog, a new
*service* is created for us that appears in **All Services** in the
WebUI. This service gives us a useful summary of its resources in the
**Totals for Service VMs** section. We can use this feature to extend
the utility of services into tracking and organising resources. We
could, for example, use a service to represent a project comprising many
dozens of virtual machines. We would be able to see the total virtual
machine resource consumption for the entire project in a single place in
the WebUI.

In line with this organisational use, we can arrange services in
hierarchies for further convenience (see [figure\_title](#i1)).

![A service hierarchy](images/ss1.png)

​  

In this example we have three child services, representing the three
tiers of our simple intranet platform. [figure\_title](#i2) shows the
single server making up the database tier of our architecture.

![The database tier](images/ss2.png)

​  

[figure\_title](#i3) shows the two servers making up the middleware tier
of our architecture.

![The middleware tier](images/ss3.png)

​  

[figure\_title](#i4) shows the four servers making up the web tier of
our architecture.

![The web tier](images/ss4.png)

​  

When we view the parent service, we see that it contains details of all
child services, including the cumulative CPU, memory and disk counts
(see [figure\_title](#i5)).

![Parent service view](images/ss5.png)

​  

## Organising Our Services

To make maximum use of service hierarchies, it is useful to be able to
create empty services, and to be able to move both services and VMs into
existing services.

### Creating an Empty Service

We could create a new service directly from automation, using the lines:

``` ruby
new_service = $evm.vmdb('service').create(:name => "My New Service")
new_service.display = true
```

For this example though, we’ll create our new empty service from a
service catalog.

#### State machine

First we’ll copy
*ManageIQ/Service/Provisioning/StateMachines/ServiceProvision\_Template/default*
into our own Domain, and rename it *EmptyService*. We’ll add a **pre5**
relationship to a new instance that we’ll create, called
*/Service/Provisioning/StateMachines/Methods/rename\_service* (see
[figure\_title](#i6)).

![Fields of the EmptyService state machine](images/ss6.png)

​  

#### Method

The **pre5** stage of this state machine is a relationship to a
*rename\_service* instance. This instance calls a *rename\_service*
method containing the following code:

``` ruby
begin
  service_template_provision_task = $evm.root['service_template_provision_task']
  service = service_template_provision_task.destination
  dialog_options = service_template_provision_task.dialog_options
  if dialog_options.has_key? 'dialog_service_name'
    service.name = "#{dialog_options['dialog_service_name']}"
  end
  if dialog_options.has_key? 'dialog_service_description'
    service.description = "#{dialog_options['dialog_service_description']}"
  end

  $evm.root['ae_result'] = 'ok'
  exit MIQ_OK
rescue => err
  $evm.log(:error, "[#{err}]\n#{err.backtrace.join("\n")}")
  $evm.root['ae_result'] = 'error'
  $evm.root['ae_reason'] = "Error: #{err.message}"
  exit MIQ_ERROR
end
```

#### Service dialog

We create a simple service dialog called "New Service" with element
names **service\_name** and **service\_description** (see
[figure\_title](#i7)).

![Service dialog](images/ss7.png)

​  

#### Putting it all together

Finally we assemble all of these parts by creating a new service catalog
called **General Services**, a new catalog item of type **Generic**
called *Empty Service* (see [figure\_title](#i8)).

![The completed "Empty Service" service catalog item](images/ss8.png)

​  

We can order from this service catalog item to create our new empty
services.

## Adding VMs and Services to Existing Services

We’ll provide the ability to move both services and virtual machines
into existing services, from a button. The button will present a
drop-down list of existing services that we can add as a new parent
service (see [figure\_title](#i9)).

![Listing available services in a dynamic drop-down](images/ss9.png)

​  

### Adding the Button

As before, the process of adding a button involves the creation of the
button dialog, and a button script. For this example however our dialog
will contain a dynamic drop-down list, so we must create a dynamic
element method as well to populate this list.

#### Button Dialog

We create a simple button dialog with a dynamic drop-down element named
**service** (see [figure\_title](#i10)).

![Button Dialog](images/ss10.png)

​  

#### Dialog element method

The dynamic drop-down element in the service dialog calls a method
called *list\_services*. We only wish to display a service in the
drop-down list if the user has permissions to see it via their tenant
membership and role-based access control (RBAC) filter. We define three
methods; `get_visible_tenant_ids` to get our tenant and any child tenant
IDs; `get_current_group_rbac_array` to retrieve a user’s RBAC filter
array, and `service_visible?` to check that a service has a tag that
matches the filter. The code is as follows:

``` ruby
def get_visible_tenant_ids
  tenant_ancestry = []
  tenant_ancestry << $evm.root['tenant'].id
  $evm.vmdb(:tenant).all.each do |tenant|
    unless tenant.ancestry.nil?
      ancestors = tenant.ancestry.split('/')
      if ancestors.include?($evm.root['tenant'].id.to_s)
        tenant_ancestry << tenant.id
      end
    end
  end
  tenant_ancestry
end

def get_current_group_rbac_array(rbac_array=[])
  user = $evm.root['user']
  unless user.current_group.filters.blank?
    user.current_group.filters['managed'].flatten.each do |filter|
      next unless /(?<category>\w*)\/(?<tag>\w*)$/i =~ filter
      rbac_array << {category => tag}
    end
  end
  rbac_array
end

def service_visible?(visible_tenants, rbac_array, service)
  visible = false
  $evm.log(:info, "Evaluating Service #{service.name}")
  if visible_tenants.include?(service.tenant.id)
    if rbac_array.length.zero?
      visible = true
    else
      rbac_array.each do |rbac_hash|
        rbac_hash.each do |category, tag|
          if service.tagged_with?(category, tag)
            visible = true
          end
        end
      end
    end
  end
  visible
end
```

When we enumerate the services, we check on visibility to the user
before adding to the drop-down list:

``` ruby
rbac_array       = get_current_group_rbac_array
visible_tenants  = get_visible_tenant_ids
values_hash      = {}
visible_services = []

$evm.vmdb(:service).all.each do |service|
  if service['display']
    if service_visible?(visible_tenants, rbac_array, service)
      visible_services << service
    end
  end
end
if visible_services.length > 0
  if visible_services.length > 1
    values_hash['!'] = '-- select from list --'
  end
  visible_services.each do |service|
    values_hash[service.id] = service.name
  end
else
  values_hash['!'] = 'No services are available'
end
```

Here we use a simple technique of keeping the string "-- select from
list --" at the top of the list, by using a key string of "\!" which is
the first ASCII printable nonwhitespace character.

#### Button method

The main instance and method called from the button are each called
*add\_to\_service*. This method adds the current virtual machine or
service, into the service selected from the drop-down list. As we wish
to be able to call this from a button on either a *Service* object type
or a *VM and instance* object type, we identify our context using
`$evm.root['vmdb_object_type']`.

If we are adding a virtual machine to an existing service, we should
allow for the fact that the virtual machine might itself have been
provisioned from a service. We detect any existing service membership,
and if the old service is empty after we move the virtual machine, we
delete the service from the VMDB:

``` ruby
begin
  new_service_id = $evm.root['dialog_service']
  new_service = $evm.vmdb('service', new_service_id) rescue nil
  if new_service.nil?
    $evm.log(:error, "Can't find service with ID: #{new_service_id}")
    exit MIQ_ERROR
  else
    case $evm.root['vmdb_object_type']
    when 'service'
      $evm.log(:info, "Adding Service #{$evm.root['service'].name}
         to #{new_service.name}")
      $evm.root['service'].parent_service = new_service
    when 'vm'
      vm = $evm.root['vm']
      #
      # See if the VM is already part of a service
      #
      unless vm.service.nil?
        old_service = vm.service
        vm.remove_from_service
        if old_service.v_total_vms.zero?
          $evm.log(:info, "Old service #{old_service.name} is now empty,
             removing it from VMDB")
          old_service.remove_from_vmdb
        end
      end
      $evm.log(:info, "Adding VM #{vm.name} to #{new_service.name}")
      vm.add_to_service(new_service)
      #
      # Set the VM's ownership to be the same as the new group
      #
      vm.owner = $evm.vmdb(:user).find_by_id(new_service.evm_owner_id)
         unless new_service.evm_owner_id.nil?
      vm.group = $evm.vmdb(:miq_group).find_by_id(new_service.miq_group_id)
         unless new_service.miq_group_id.nil?
    end
  end
  exit MIQ_OK
rescue => err
  $evm.log(:error, "[#{err}]\n#{err.backtrace.join("\n")}")
  exit MIQ_ERROR
end
```

The scripts in this chapter are available
[here](https://github.com/pemcg/mastering-automation-in-cloudforms-4.2-and-manageiq-euwe/tree/master/service_hierarchies/scripts)

#### Putting it all together

Finally we create two **Add to Service** buttons, one on a *Service*
object type, and one on a *VM and Instance* object type. We can go ahead
and organise our service hierarchies.

> **Note**
> 
> *Exercise*
> 
> Filter the list of services presented in the drop-down to remove the
> *current* service - we would never wish to add a service as its own
> parent.

## Summary

Organising our services in this way changes the way that we think about
our cloud or virtual infrastructure. We start to think in terms of
service workloads, rather than individual virtual machines or instances.
We can start to work in a more "cloudy" way, where we treat our virtual
machines as anonymous entities, and scale out or scale back according to
point-in-time application demand.

We can also use service bundles and hierachies of bundles to keep track
of the resources in projects and subprojects. This can help from an
organisational point of view, for example we can tag services, and our
method to add a virtual machine to a service can propagate any service
tags to the virtual machine. In this way we can assign project-related
chargeback costs to the tagged VMs, or apply WebUI display filters that
display project resources.
