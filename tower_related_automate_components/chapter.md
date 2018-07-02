# Tower-Related Automate Components

In this chapter we’ll learn how to integrate Ansible playbooks into our
Automate workflows. We’ll look at the Tower-related components in the
Automate datastore, and how we can use them to interact with Ansible
Tower and launch jobs. We’ll also take a look at the service model
objects that represent Tower jobs, job templates and inventories, and
the useful characteristics of each that we can access from automation
scripts.

## Automate Datastore Components

The Tower-related Automate code is in the *ManageIQ* domain, under the
*ConfigurationManagement/AnsibleTower* namespace. The *Operations*
namespace contains the *JobTemplate* and *StateMachines/Job* classes
(see [figure\_title](#i1)).

![ConfigurationManagement/AnsibleTower namespace](images/ss1.png)

​  

> **Note**
> 
> There is a further namespace called *Service* under
> *ConfigurationManagement/AnsibleTower* that contains the
> service-related Automate components, but we’ll cover these in a later
> chapter.

### Job State Machine

The *Job* state machine class provides the workflow to integrate with
Ansible Tower. The out-of-the-box class has a single instance called
*default* that contains three states (see [figure\_title](#i2)).

![Fields of the default state machine](images/ss3.png)

​  

#### WaitForIP

The **WaitForIP** state calls the *wait\_for\_ip* method, which waits
for the `vm.ipaddresses` list attribute to be non-empty. The `vm` object
is populated from either `$evm.root['miq_provision'].destination` (i.e.
the VM resulting from a provisioning operation) or `$evm.root['vm']`
(i.e. the current VM object loaded from a button operation).

> **Note**
> 
> The native virtual infrastructure guest agent such as VMware tools or
> the RHEV-M management agent must be installed and running in the
> target VM for the `ipaddresses` list attribute to be populated.
> 
> The *wait\_for\_ip* method does not attempt to power-on the virtual
> machine. If the VM is not powered on by some other workflow step or
> manual operation, this state will exit with an error after 100
> retries.

#### Launch

The **Launch** state calls the *launch\_ansible\_job* method. This
method uses arguments passed to `$evm.object` to determine the job
template to be run, and any extra variables that should be passed to the
job.

For flexibility the method is able to identify the job template to be
run in three ways, as follows (in order):

1.  From the value of `$evm.object['job_template']` if it exists, which
    should be a
    `ManageIQ_Providers_AnsibleTower_ConfigurationManager_ConfigurationScript`
    object.

2.  From the value of `$evm.object['job_template_name']` if it exists,
    which should be a text string containing the job template name (e.g.
    "Tomcat Standalone Server")

3.  From the value of `$evm.object['job_template_id']` if it exists,
    which should be a text string containing the job template ID (e.g.
    "342")

The *launch\_ansible\_job* method also uses different ways to search for
arguments to pass to the job template as extra variables.

1.  If the *Job/default* state machine has been called as part of a
    provisioning operation (and therefore `$evm.root['miq_provision']`
    exists), the *launch\_ansible\_job* method searches the provisioning
    task options hash for keys with a name in the style
    `dialog_param_<extra_var>`. If any are found then the `<extra_var>`
    part is extracted from the key name, and it and the value are passed
    to the job template as extra variables.

2.  The *launch\_ansible\_job* method also searches `$evm.object` and
    all of its parent instances up to `$evm.root` for attribute keys
    with either the style `dialog_param_<extra_var> = value` or
    `dialog<n> = <extra_var>=value`. For example if passing an extra
    variable called "package\_name" with the value "vim-enhanced", we
    could use either of the following styles:

<!-- end list -->

``` ruby
$evm.object['dialog_param_package_name'] = 'vim-enhanced'
```

or

``` ruby
$evm.object['param1'] = 'package_name=vim-enhanced'
```

The first style makes it easy for us to prompt for extra variables from
a service dialog. In this example we need only give our service dialog
element the name "param\_package\_name", and the value is ready to pass
into the state machine.

Once *launch\_ansible\_job* has successfully launched the job template
in the Ansible Tower provider, it saves the job ID as the state variable
`:ansible_job_id` for reference elsewhere in the state machine.

#### WaitForCompletion

The **WaitForCompletion** state calls the *wait\_for\_completion*
method. This reads the saved job ID from the `:ansible_job_id` state
variable, and polls the Ansible Tower provider for the job completion
status.

The method exits with `$evm.root['ae_result']` set to 'error', 'retry'
or 'ok' as appropriate, and prints a message to *automation.log* in the
case of an error.

### Calling the State Machine from an Automate Method

We can easily run an Ansible Tower job template on any virtual machine
from an Automate method.

In this example we’ll run a job template called 'Install Single Package'
on a VM called 'testserver'. We’ll pass to the job template the extra
variable 'package\_name' with the value 'screen'. To prevent a
long-running Ansible playbook from timing out our currently running
method, we’ll launch the AnsibleTower state machine as a new automation
request, rather than using `$evm.instantiate`. As the new method will
not running as part of a provisioning operation, nor called from a
button on the VM in the WebUI, we must ensure that the VM object is
loaded into `$evm.root['vm']` at runtime. We do this by passing a
"Vm:vm" argument containing the VM object ID that we wish to load, as
shown in the following
code:

``` ruby
ANSIBLE_NAMESPACE = 'ConfigurationManagement/AnsibleTower/Operations/StateMachines'.freeze
ANSIBLE_STATE_MACHINE_CLASS = 'Job'.freeze
ANSIBLE_STATE_MACHINE_INSTANCE = 'default'.freeze
VM_CLASS = 'VmOrTemplate'.freeze
attrs = {}
attrs['job_template_name'] = 'Install Single Package'
attrs['dialog_param_package_name'] = 'screen'
#
# Passing an attribute of Vm::vm=id ensures that the executed method will
# have $evm.root['vm'] pre-loaded with the VM with this ID
#
vm = $evm.vmdb(VM_CLASS).find_by_name('testserver')
attrs['Vm::vm'] = vm.id
#
# make sure the VM is started
#
vm.start if vm.power_state != 'on'
#
# Call the job template as a new automation request in case it runs for
# longer than 10 minutes
#
options = {}
options[:namespace]     = ANSIBLE_NAMESPACE
options[:class_name]    = ANSIBLE_STATE_MACHINE_CLASS
options[:instance_name] = ANSIBLE_STATE_MACHINE_INSTANCE
options[:user_id]       = $evm.root['user'].id
options[:attrs]         = attrs
auto_approve            = true
$evm.execute('create_automation_request', options, $evm.root['user'].userid, auto_approve)
```

Rather than passing the attribute 'job\_template\_name', we could if we
wish pass 'job\_template\_id'. For example we may have been passed the
template ID from a service dialog, or we may have multiple Ansible Tower
providers with duplicate job template names, and wish to identify the
correct template. The following code shows how we would lookup and
specify the job template ID for a job template on a particular provider
(in this case the provider has the ID of
4):

``` ruby
SCRIPT_CLASS = 'ManageIQ_Providers_AnsibleTower_ConfigurationManager_ConfigurationScript'.freeze
job_template =  $evm.vmdb(SCRIPT_CLASS).where(
                ["manager_id = ? AND name = ?", 4, 'Install Single Package']
                ).first
attrs['job_template_id'] = job_template.id
```

### Calling the State Machine from an Instance

We can call the *Job/default* state machine directly from a relationship
in an instance, and even pass extra variable arguments from schema
attributes. This gives us the flexibility to be able to combine Ruby
methods and Ansible playbooks in a single instance if we wish (see
[figure\_title](#i3)).

![Combining Ruby methods and Ansible playbooks in a single
instance](images/ss6.png)

​  

### JobTemplate Class

The JobTemplate class has been created to simplify the process of
calling Ansible job templates with no requirement for any Ruby
scripting. We can call this class using a relationship URI that ends
with the name of an Ansible job template. An example showing a
relationship field calling the "JBoss\_Standalone\_Server" job template
is shown in [figure\_title](#i4).

![Calling the JBoss\_Standalone\_Server Ansible job
template](images/ss4.png)

​  

If there is no instance in the JobTemplate class with the same name as
the job template (in this example "JBoss\_Standalone\_Server"), the
*.missing* instance will be called (see [figure\_title](#i5)).

![Fields of the .missing instance](images/ss2.png)

​  

The *.missing* instance uses the translated `${#_missing_instance}`
substitution variable (which in this example will contain the string
"JBoss\_Standalone\_Server") as the value for the
**job\_template\_name** attribute. The instance then runs the
*Job/default* state machine from the **Launch** field.

> **Note**
> 
> To take advantage of the *.missing* instance behaviour in this way,
> our job template name should contain no spaces. The default extra
> variables will be used when the job is run (we can’t pass parameters
> to the job template).

#### User-defined JobTemplate Instances

If we have job templates that we call regularly with overriden extra
variables, or that contain spaces in the template name, we can define
our own instances under
*/ConfigurationManagement/AnsibleTower/Operations/JobTemplate* in a
custom domain (see [figure\_title](#i5)).

![User-defined JobTemplate instances](images/ss5.png)

​  

These custom instances can then be called in the usual manner.

### /System/Request/ansible\_tower\_job

There is an entry point under */System/Request* called
*ansible\_tower\_job* that we can call from any WebUI component that
expects an entry point under */System/Request* (such as a button). This
entry point contains a single relationship to
*/ConfigurationManagement/AnsibleTower/Operations/StateMachines/Job/default*,
so we must pass additional arguments such as "job\_template\_name" as
attribute/value pairs.

## Service Models

There are several service models that are of interest to us when we use
use the capabilities of Ansible Tower from our automation scripts.

### ManageIQ\_Providers\_AnsibleTower\_ConfigurationManager\_Job

The ManageIQ\_Providers\_AnsibleTower\_ConfigurationManager\_Job object
represents an Ansible Tower job. An object\_walker printout of a typical
object is as follows:

    --- attributes follow ---
    job.ancestry = nil
    job.cloud_tenant_id = nil
    job.created_at = 2016-11-23 17:38:00 UTC
    job.description = nil
    job.ems_id = 4
    job.ems_ref = 145
    job.id = 49
    job.name = JBoss_Standalone_Server
    job.orchestration_template_id = 5
    job.resource_group = nil
    job.retired = nil
    job.retirement_last_warn = nil
    job.retirement_requester = nil
    job.retirement_state = nil
    job.retirement_warn = nil
    job.retires_on = nil
    job.status = pending
    job.status_reason = nil
    job.type = ManageIQ::Providers::AnsibleTower::ConfigurationManager::Job
    job.updated_at = 2016-11-23 17:38:00 UTC
    --- end of attributes ---
    --- virtual columns follow ---
    job.region_description = Region 0
    job.region_number = 0
    job.total_cloud_networks = 0
    job.total_security_groups = 0
    job.total_vms = 0
    --- end of virtual columns ---
    --- associations follow ---
    job.ext_management_system
    job.job_template
    job.outputs
    job.parameters
    job.resources
    --- end of associations ---
    --- methods follow ---
    job.add_to_service
    job.error_retiring?
    job.finish_retirement
    job.inspect
    job.inspect_all
    job.model_suffix
    job.normalized_live_status
    job.raw_delete_stack
    job.raw_exists?
    job.raw_stdout
    job.raw_update_stack
    job.refresh_ems
    job.reload
    job.remove_from_vmdb
    job.retire_now
    job.retired?
    job.retirement_state=
    job.retirement_warn=
    job.retires_on=
    job.retiring?
    job.start_retirement
    job.tag_assign
    job.tag_unassign
    job.tagged_with?
    job.tags
    --- end of methods ---
    --- object does not support custom attributes ---

From this listing we notice several useful properties. There are some
interesting attributes, including:

  - `job.ems_ref` corresponds to the Job ID in Ansible Tower.

  - `job.orchestration_template_id` is the CloudForms/ManageIQ ID of the
    Ansible Tower job template

  - `job.status`, is the job status, but this is not necessarily current
    (see `normalized_live_status` below)

The `job.parameters` association is a list of
OrchestrationStackParameter service model objects representing the
parameters (i.e. extra variables) that were used when the job was run.
Typical attributes of a parameter object are as follows:

    parameter.ems_ref = 145_http_port
    parameter.id = 260
    parameter.name = http_port
    parameter.stack_id = 49
    parameter.value = 80

There are several useful
ManageIQ\_Providers\_AnsibleTower\_ConfigurationManager\_Job methods
that we can call, including:

  - `job.normalized_live_status` will return the current job status as a
    \[status, reason\] array from Ansible Tower. This is the method that
    */ConfigurationManagement/AnsibleTower/Operations/StateMachines/Job/wait\_for\_completion*
    calls to determine job status, and typical values might be
    \["create\_complete", "OK"\], or \["failed", "Job launching
    failed"\].

  - `job.raw_stdout` will return the raw output from the job, such
    as:

<!-- end list -->

    "Identity added: /tmp/ansible_tower_Qa8enO/credential (/tmp/ansible_tower_Qa8enO/credential)\r\nVault password: \r\n\r\nPLAY [Install Package] *********************************************************\r\n\r\nTASK [setup] *******************************************************************\r\nok: [..."

From the tag-related methods we see that a
ManageIQ\_Providers\_AnsibleTower\_ConfigurationManager\_Job object is
taggable.

### MIQ\_Providers\_AnsibleTower\_ConfigurationManager\_ConfigurationScript

The
ManageIQ\_Providers\_AnsibleTower\_ConfigurationManager\_ConfigurationScript
object represents an Ansible Tower job template. An object\_walker
printout of a typical object is as follows:

    --- attributes follow ---
    configuration_script.created_at = 2016-11-18 16:20:17 UTC
    configuration_script.description = Install a JBoss Standalone Server
    configuration_script.id = 5
    configuration_script.inventory_root_group_id = 8
    configuration_script.manager_id = 4
    configuration_script.manager_ref = 48
    configuration_script.name = JBoss_Standalone_Server
    configuration_script.survey_spec = {}
    configuration_script.type = ManageIQ::Providers::AnsibleTower::ConfigurationManager...Script
    configuration_script.updated_at = 2016-11-23 17:37:25 UTC
    configuration_script.variables = {"http_port"=>80, "https_port"=>443}
    --- end of attributes ---
    --- virtual columns follow ---
    configuration_script.region_description = Region 0
    configuration_script.region_number = 0
    --- end of virtual columns ---
    --- associations follow ---
    configuration_script.inventory_root_group
    configuration_script.manager
    --- end of associations ---
    --- methods follow ---
    configuration_script.inspect
    configuration_script.inspect_all
    configuration_script.model_suffix
    configuration_script.reload
    configuration_script.run
    configuration_script.tag_assign
    configuration_script.tag_unassign
    configuration_script.tagged_with?
    configuration_script.tags
    --- end of methods ---
    --- object does not support custom attributes ---

From this object we can see a number of useful properties, including the
following attributes:

  - `configuration_script.properties`, which is a hash containing the
    default extra variables that have been defined for the job template
    in Ansible Tower (we may wish to display these as element defaults
    in a service dialog for example).

  - `configuration_script.manager_ref` is the Ansible Tower ID for the
    job template.

The association `configuration_script.inventory_root_group` contains the
ManageIQ\_Providers\_ConfigurationManager\_InventoryRootGroup object
that represents the Tower inventory that the job template is defined to
run against (see below).

We see that a
ManageIQ\_Providers\_AnsibleTower\_ConfigurationManager\_ConfigurationScript
object is also taggable.

### ManageIQ\_Providers\_ConfigurationManager\_InventoryRootGroup

The ManageIQ\_Providers\_ConfigurationManager\_InventoryRootGroup object
represents an Ansible Tower inventory. An object\_walker printout of a
typical object is as follows:

    --- attributes follow ---
    inventory_root_group.created_on = 2016-10-19 15:38:35 UTC
    inventory_root_group.ems_id = 4
    inventory_root_group.ems_ref = 4
    inventory_root_group.ems_ref_obj = nil
    inventory_root_group.hidden = nil
    inventory_root_group.id = 8
    inventory_root_group.name = CloudForms VMs
    inventory_root_group.type = ManageIQ::Providers::ConfigurationManager::InventoryRootGroup
    inventory_root_group.uid_ems = nil
    inventory_root_group.updated_on = 2016-10-19 15:38:35 UTC
    --- end of attributes ---
    --- virtual columns follow ---
    inventory_root_group.aggregate_cpu_speed = 0
    inventory_root_group.aggregate_cpu_total_cores = 0
    inventory_root_group.aggregate_disk_capacity = 0
    inventory_root_group.aggregate_logical_cpus = 0
    inventory_root_group.aggregate_memory = 0
    inventory_root_group.aggregate_physical_cpus = 0
    inventory_root_group.aggregate_vm_cpus = 0
    inventory_root_group.aggregate_vm_memory = 0
    inventory_root_group.region_description = Region 0
    inventory_root_group.region_number = 0
    inventory_root_group.total_configured_systems = 15
    --- end of virtual columns ---
    --- associations follow ---
    inventory_root_group.configuration_scripts
    inventory_root_group.hosts
    inventory_root_group.manager
    inventory_root_group.vms
    --- end of associations ---
    --- methods follow ---
    inventory_root_group.folder_path
    inventory_root_group.inspect
    inventory_root_group.inspect_all
    inventory_root_group.model_suffix
    inventory_root_group.register_host
    inventory_root_group.reload
    inventory_root_group.tag_assign
    inventory_root_group.tag_unassign
    inventory_root_group.tagged_with?
    inventory_root_group.tags
    --- end of methods ---
    --- object does not support custom attributes ---

The most useful properties from this object are
`inventory_root_group.name`, and `inventory_root_group.ems_ref`, which
is the Ansible Tower ID for the inventory.

As with the other two objects, a
ManageIQ\_Providers\_ConfigurationManager\_InventoryRootGroup object is
also taggable.

## Summary

This chapter has explored the features of Automate introduced in
CloudForms 4.1/ManageIQ *Darga* that allow us to integrate with Ansible
Tower. They allow us to easily call Tower jobs, either from a running
Automate Ruby method, or from a simple instance relationship. We don’t
necessarily need to write any Ruby code to launch an Ansible Tower job,
we can just create a class and instance in the Automate datastore and
call this from a button, or - as we’ll see in a later chapter - as a
service.

### Further Reading

[Launching Ansible Tower Job Templates from
ManageIQ](http://talk.manageiq.org/t/launching-ansible-tower-job-templates-from-manageiq/1394)
