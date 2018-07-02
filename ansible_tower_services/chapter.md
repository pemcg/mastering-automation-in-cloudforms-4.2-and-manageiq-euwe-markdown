# Ansible Tower Services

One of the available catalog item types when we create a new service
item is **AnsibleTower** (see [figure\_title](#i1))

![Ansible Tower catalog item type](images/ss1.png)

​  

In this chapter we’ll investigate the Automate datastore components that
allow us to create Ansible Tower services and service bundles that
include Ansible Tower jobs.

## The Ansible Tower Service Provisioning State Machine

The */ConfigurationManagement/AnsibleTower/Service/Provisioning*
namespace in the datastore contains the service provisioning state
machines, methods and associated email classes that are required to
provisioning services into Ansible Tower (see [figure\_title](#i2)).

![ConfigurationManagement/AnsibleTower namespace](images/ss2.png)

​  

The two out-of-the-box instances of the *Provision* state machine are
*default* and *provision\_from\_bundle*. We use *default* when we’re
creating a single standalone service, and *provision\_from\_bundle* when
we’re running an Ansible Tower job as part of a service bundle
comprising both VM provisioning and Ansible configuration operations.

### The *default* State Machine Instance

The *default* state machine instance is called to process individual
service catalog items. The fields of this state machine are shown in
[figure\_title](#i3).

![Fields of the default state machine](images/ss3.png)

​  

#### pre1

The **pre1** state calls the *preprovision* method, that checks whether
the inputs are valid, and prints some of the input values to
*automation.log*. It contains a useful method called
*modify\_job\_options* that by default is not called (the call is
commented out), but would allow us to customise any of the job options
if we wished to clone and edit the method.

``` ruby
def modify_job_options(service)
  # Example how to programmatically modify job options:
  job_options = service.job_options
  job_options[:limit] = 'someHost'
  job_options[:extra_vars]['flavor'] = 'm1.small'

  # Important: set stack_options
  service.job_options = job_options
end
```

#### provision

The **provision** state calls the *provision* method, which performs
some preliminary checking before calling the service object’s
`launch_job` method.

#### checkprovisioned

The **checkprovisioned** state calls the *check\_provisioned* method,
which calls the service object’s `job` method to retrieve the
ManageIQ\_Providers\_AnsibleTower\_ConfigurationManager\_Job object, and
then calls the `normalized_live_status` job method to retrieve the
current job status.

#### post1

The **post1** state calls the *post\_provisioned* method which allows us
to perform any optional post-processing that we might deem necessary. It
contains a useful method called *dump\_job\_outputs* that by default is
not called (the call is commented out), but would allow us to write the
job output to *automation.log* if required.

``` ruby
def dump_job_outputs(job)
  log_type = job.status == 'failed' ? 'error' : 'info'
  @handle.log(log_type, "Ansible Tower Job #{job.name} standard output: #{job.raw_stdout}")
end
```

#### EmailOwner

The **EmailOwner** state calls the *ServiceProvision\_complete* email
instance to notify the service requester that the service has completed.

#### Finished

The **Finished** state calls the
*/System/CommonMethods/StateMachineMethods/service\_provision\_finished*
instance to terminate the service provision state machine processing.

### The *provision\_from\_bundle* State Machine Instance

The *provision\_from\_bundle* state machine instance is called when an
Ansible service catalog item is to be called from a service bundle after
a VM provisioning service catalog item. The fields of this state machine
are shown in [figure\_title](#i4).

![Fields of the provision\_from\_bundle state machine](images/ss4.png)

​  

As can be seen, the difference between this state machine and *default*
is that *preprovision* has moved to the **pre2** state, and there are
new relationships in the **sequencer** and **pre1** states to call
*GroupSequenceCheck* and *CatalogItemInitialization*.

#### Sequencer

The **Sequencer** state calls the same *GroupSequenceCheck* instance and
method that the VM provision state machines run. The
*GroupSequenceCheck* method checks the eligibility of the current
service template provisioning task to run, according to the provision
order defined when the resources were added to the service bundle.
*GroupSequenceCheck* allows the state machine to continue if all other
tasks with a lower provisioning priority have a `state` attribute of
"finished". If any of the lower priority tasks are incomplete,
*GroupSequenceCheck* exits with a state retry and a retry interval of
one minute.

The common call to *GroupSequenceCheck* made by both VM provisioning and
AnsibleTower job state machines allows us to interleave VM provisioning
service items with Ansible configuration service items. We can be sure
that the Ansible configuration will not proceed until the virtual
machine has been fully provisioned.

#### pre1

The **pre1** state calls calls the same *CatalogItemInitialization*
instance and method that the VM provision state machines run. This is to
ensure that any service dialog values passed into the service bundle are
available to the Ansible service template provisioning task.

## Service Models

The Ansible-related service model that is of interest to us is the
MiqAeServiceServiceAnsibleTower object.

### MiqAeServiceServiceAnsibleTower

The MiqAeServiceServiceAnsibleTower object represents an Ansible Tower
service. An object\_walker printout of a typical object is as follows:

``` 
 --- attributes follow ---
 service.ancestry = nil
 service.created_at = 2016-12-01 11:11:00 UTC
 service.description = Install a Simple LAMP Stack
 service.display = true
 service.evm_owner_id = 1
 service.guid = d709ae06-b7b6-11e6-b465-001a4aa0151a
 service.id = 5
 service.miq_group_id = 2
 service.name = Simple LAMP Stack
 service.options[:dialog] = {"dialog_limit"=>"lampsrv001", "dialog_param_ntpserver"=>"192.168.xx.xx", "dialog_param_mysql_port"=>"3306", "dialog_param_dbname"=>"foodb", "dialog_param_dbuser"=>"foouser", "dialog_param_dbpass"=>"secret", "dialog_param_httpd_port"=>"80", "dialog_param_repository"=>"https://github.com/pemcg/mywebapp.git"}
 service.retired = nil
 service.retirement_last_warn = nil
 service.retirement_requester = nil
 service.retirement_state = nil
 service.retirement_warn = nil
 service.retires_on = nil
 service.service_template_id = 2
 service.tenant_id = 1
 service.type = ServiceAnsibleTower
 service.updated_at = 2016-12-01 11:11:00 UTC
 --- end of attributes ---
 --- virtual columns follow ---
 service.aggregate_all_vm_cpus = 0
 service.aggregate_all_vm_disk_count = 0
 service.aggregate_all_vm_disk_space_allocated = 0
 service.aggregate_all_vm_disk_space_used = 0
 service.aggregate_all_vm_memory = 0
 service.aggregate_all_vm_memory_on_disk = 0
 service.aggregate_direct_vm_cpus = 0
 service.aggregate_direct_vm_disk_count = 0
 service.aggregate_direct_vm_disk_space_allocated = 0
 service.aggregate_direct_vm_disk_space_used = 0
 service.aggregate_direct_vm_memory = 0
 service.aggregate_direct_vm_memory_on_disk = 0
 service.custom_1 = nil
 service.custom_2 = nil
 service.custom_3 = nil
 service.custom_4 = nil
 service.custom_5 = nil
 service.custom_6 = nil
 service.custom_7 = nil
 service.custom_8 = nil
 service.custom_9 = nil
 service.evm_owner_email = nil
 service.evm_owner_name = Administrator
 service.evm_owner_userid = admin
 service.has_parent = false
 service.owned_by_current_ldap_group = nil
 service.owned_by_current_user = nil
 service.owning_ldap_group = EvmGroup-super_administrator
 service.power_state = nil
 service.power_status = nil
 service.region_description = Region 0
 service.region_number = 0
 service.service_id = nil
 service.v_total_vms = 0
 --- end of virtual columns ---
 --- associations follow ---
 service.all_service_children
 service.direct_service_children
 service.direct_vms
 service.indirect_service_children
 service.indirect_vms
 service.parent_service
 service.root_service
 service.service_resources
 service.service_template
 service.tenant
 service.vms
 --- end of associations ---
 --- methods follow ---
 service.automate_retirement_entrypoint
 service.configuration_manager
 service.custom_get
 service.custom_keys
 service.custom_set
 service.description=
 service.dialog_options
 service.display=
 service.error_retiring?
 service.extend_retires_on
 service.finish_retirement
 service.get_dialog_option
 service.group=
 service.inspect
 service.inspect_all
 service.job
 service.job_options
 service.job_options=
 service.job_template
 service.job_template=
 service.launch_job
 service.model_suffix
 service.name=
 service.owner=
 service.parent_service=
 service.reload
 service.remove_from_vmdb
 service.retire_now
 service.retire_service_resources
 service.retired?
 service.retirement_state=
 service.retirement_warn=
 service.retires_on=
 service.retiring?
 service.set_dialog_option
 service.shutdown_guest
 service.start
 service.start_retirement
 service.stop
 service.suspend
 service.tag_assign
 service.tag_unassign
 service.tagged_with?
 service.tags
 --- end of methods ---
```

The object is an extension of the standard MiqAeServiceService object
type, but adds several useful Ansible-specific methods, as follows:

``` 
 service.configuration_manager
 service.job
 service.job_options
 service.job_options=
 service.job_template
 service.job_template=
 service.launch_job
```

It is the `launch_job` method that is called during the state machine
**provision** state to initiate the running of the Ansible Tower job.

## Summary

The chapter has completed our examination of the Tower-related
components in the Automate datastore that we started in [Tower Related
Automate
Components](../tower_related_automate_components/chapter.asciidoc). The
state machines, instances and methods that we’ve studied here are used
when we create services to deploy Ansible configuration scripts.

In the next chapter we’ll run through two examples of creating Ansible
Tower services; one for a single catalog item, and another as part of a
catalog bundle.
