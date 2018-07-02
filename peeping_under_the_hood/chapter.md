# Peeping Under the Hood

We’ve now worked with an Automation Engine object that represents a
virtual machine, and we’ve called one of its methods to add a custom
attribute to the VM.

In this chapter we’ll take a deeper look at these Automation Engine
objects, and at some of the technology that exists behind the scenes in
Rails when we run automation scripts. It is useful background
information, but can be skipped on first read if required.

## A Little Rails Knowledge (Goes a Long Way)

Firstly, by way of reassurance…​

**We do not need to know Ruby on Rails to write automation scripts**.

It can, however, be useful to have an appreciation of Rails *models*,
and how the Automation Engine encapsulates these as Ruby objects that we
can program with. The objects represent the things that we are typically
interested in when we write automation code, such as VMs, Clusters,
Guest Applications, or even Provisioning Requests.

### Plain Old Ruby

The Ruby scripts that we write are just plain Ruby 2.x, although the
Active Support core extensions to Ruby are available if we wish to use
them.

> **Note**
> 
> The Active Support extensions can make our lives easier. For example
> rather than adding math to our automation script to work out the
> number of seconds in a two-month time span (perhaps to specify a VM
> retirement period), we can just specify `2.months`.

Our automation scripts access Ruby objects, made available to us by the
Automation Engine via the `$evm` variable (`$evm` is described in more
detail in [$evm and the
Workspace](../evm_and_the_workspace/chapter.asciidoc)). Behind the
scenes these are Rails objects, which is why having some understanding
of Rails can help our investigation into how we can use these objects to
our maximum benefit.

### Model-View-Controller

Rails is a Model-View-Controller (MVC) application \[1\]

![An MVC application](images/mvc.png)

​  

The *model* is a representation of the underlying, logical structure of
data from the database (which in the case of CloudForms and ManageIQ is
PostgreSQL). When writing automation scripts, we work with models
extensively (although we may not necessarily realise it).

Rails models are called *active records*. They always have a singular
*CamelCase* name (e.g. GuestApplication), and their corresponding
database tables have a plural *snake\_case* name (e.g.
guest\_applications).

### Active Record Associations

Active record *associations* link the models together in one-to-many or
one-to-one relationships that allow us to traverse objects.

We can illustrate this by looking at some of the Rails code that defines
the *Host* (i.e. Hypervisor) active record:

``` ruby
class Host < ActiveRecord::Base
  ...
  belongs_to                :ext_management_system, :foreign_key => "ems_id"
  belongs_to                :ems_cluster
  has_one                   :operating_system, :dependent => :destroy
  has_one                   :hardware, :dependent => :destroy
  has_many                  :vms_and_templates, :dependent => :nullify
  has_many                  :vms
  ...
```

We see that there are several associations from a host object, including
to the cluster that it’s a member of, and to the virtual machines that
run on that host.

Although these associations are defined in Rails, they are available to
us when we work with the corresponding *service model* objects from the
Automation Engine (see [Service Models](#service-models)).

### Rails Helper Methods (.find\_by\_\*)

Rails does a lot of things to make our lives easier, including
dynamically creating *helper methods*. The most useful ones to us as
automation scripters are the find\_by\_columnname methods.

``` ruby
vm = $evm.vmdb('vm').find_by_name(vm_name)
```

We can `.find_by_` any column name in a database table. For example in
PostgreSQL we can look at the *services* table that represents services
created via a service catalog.

    vmdb_production=# \d services
                                              Table "public.services"
            Column        |            Type             |                 Modifiers
    ----------------------+-----------------------------+---------------------------
     id                   | bigint                      | not null default nextva...
     name                 | character varying(255)      |
     description          | character varying(255)      |
     guid                 | character varying(255)      |
     type                 | character varying(255)      |
     service_template_id  | bigint                      |
     options              | text                        |
     display              | boolean
     ...

We see that there is a `description` column, so if we wanted we could
call:

``` ruby
$evm.vmdb('service').find_by_description('My New Service')
```

Earlier versions of Rails also had a `find_all_by_*` helper method that
returned all results matching a search. Rails 5 has removed this in
favour of `where`, which is now also supported by service models. The
syntax of using `where` is as follows:

``` ruby
$evm.vmdb('service').where(:description =>'My New Service')
```

> **Note**
> 
> `where` returns a list, even if it only finds one item

## Service Models

We saw earlier that Rails data models are called *active records*. We
can’t access these directly from an automation script, but fortunately
most of the useful ones are made available to us as Automation Engine
*service model* objects.

The objects that we work with in the Automation Engine are all service
models; instances of an *MiqAeService* class that abstract and make
available to us their corresponding Rails active record.

For example if we’re working with a *User* object (representing a
person, such as the owner of a virtual machine), we might access that
object in our script via `$evm.root['user']`. This is actually an
instance of an *MiqAeServiceUser* class, which represents the
corresponding Rails *User* Active Record. There are service model
objects representing all of the things that we need to work with when we
write automation scripts. These include the traditional components in
our infrastructure such as virtual machines, hypervisor clusters,
operating systems or ethernet adapters, but also the intangible objects
such as provisioning requests or automation tasks.

All of the MiqAeService\* objects extend a common
*MiqAeServiceModelBase* class that contains some common methods
available to all objects, such as:

    .tagged_with?(category, name)
    .tags(category = nil)
    .tag_assign(tag)

Many of the service model objects have several levels of superclass, for
example:

    MiqAeServiceManageIQ_Providers_Redhat_InfraManager_ProvisionViaPxe <
        MiqAeServiceManageIQ_Providers_Redhat_InfraManager_Provision <
            MiqAeServiceMiqProvision <
                MiqAeServiceMiqRequestTask <
                    MiqAeServiceModelBase

### Service Model Names and Provider Namespacing

The service model names for any provider-specific classes follow the
provider namespacing scheme introduced in CloudForms 4.0 (ManageIQ
*Capablanca*). This separates the providers in several categories and in
the current versions of the tools these categories are as follows:

  - CloudManager

  - ContainerManager

  - ConfigurationManager

  - InfraManager

  - NetworkManager

The provider-specific service model objects are named in the following
way:

    MiqAeServiceManageIQ_Providers_<ProviderName>_<ProviderCategory>_<ProviderObject>

For example the service model object name for an OpenStack cloud subnet
is:

    MiqAeServiceManageIQ_Providers_Openstack_NetworkManager_CloudSubnet

The object name for a VMware ESX host is:

    MiqAeServiceManageIQ_Providers_Vmware_InfraManager_HostEsx

> **Note**
> 
> The pre-CloudForms 4.0 provider-specific service model names have been
> retained for backwards compatibility, so for now we can still use a
> command such as:
> 
>     $evm.vmdb(:CloudSubnet).all

## Service Model Object Properties

The service model objects that the Automation Engine makes available to
us have four properties that we frequently work with, *attributes*,
*virtual columns*, *associations* and *methods*.

### Attributes

Just like any other Ruby object, the service model objects that we work
with have *attributes* that we often use. A service model object
represents a record in a database table, and the object’s attributes
correspond to the columns in the table for that record.

For example, some attributes for a RHEV Host (i.e. Hypervisor) object
(the `MiqAeServiceManageIQ_Providers_Redhat_InfraManager_Host` service
model), with typical values, are:

    host.connection_state = connected
    host.created_on = 2014-11-13 17:53:34 UTC
    host.ems_cluster_id = 1000000000001
    host.ems_id = 1000000000001
    host.ems_ref = /api/hosts/b959325b-67-4e3a-a52e-fd936c225a1a
    host.ems_ref_obj = /api/hosts/b959325b-67-4e3a-a52e-fd936c225a1a
    host.guid = fcea82c8-6b5d-11e4-98ac-001a4aa01599
    host.hostname = 192.168.1.224
    host.hyperthreading = nil
    host.id = 1000000000001
    host.ipaddress = 192.168.1.224
    host.last_perf_capture_on = 2015-06-05 10:25:46 UTC
    host.name = rhelh03.bit63.net
    host.power_state = on
    host.settings = {:autoscan=>false, :inherit_mgt_tags=>false, :scan_frequency=>0}
    host.smart = 1
    host.type = HostRedhat
    host.uid_ems = b959325b-67-4e3a-a52e-fd936c225a1a
    host.updated_on = 2015-06-05 10:43:00 UTC
    host.vmm_product = rhel
    host.vmm_vendor = RedHat

We can enumerate an object’s attributes using:

``` ruby
this_object.attributes.each do |key, value|
```

### Virtual Columns

In addition to the standard object attributes (which correspond to
'real' database columns), Rails dynamically adds a number of *virtual
columns* to many of the service models.

> **Note**
> 
> A virtual column is a computed database column that is not physically
> stored in the table. Virtual columns often contain more dynamic values
> than attributes, such as the number of VMs currently running on a
> hypervisor.

Some virtual columns for our same RHEV Host object, with typical values,
are:

    host.authentication_status = Valid
    host.derived_memory_used_avg_over_time_period = 790.1026640002773
    host.derived_memory_used_high_over_time_period = 2586.493300608264
    host.derived_memory_used_low_over_time_period = 0
    host.os_image_name = linux_generic
    host.platform = linux
    host.ram_size = 15821
    host.region_description = Region 1
    host.region_number = 1
    host.total_cores = 4
    host.total_vcpus = 4
    host.v_owning_cluster = Default
    host.v_total_miq_templates = 0
    host.v_total_storages = 3
    host.v_total_vms = 7

We access theses virtual columns just as we would access attributes,
using "object.virtual\_column\_name" syntax. If we want to enumerate
through all of an object’s virtual columns getting the corresponding
values, we must use `.send`, specifying the virtual column name, like
so:

``` ruby
this_object.virtual_column_names.each do |virtual_column_name|
  virtual_column_value = this_object.send(virtual_column_name)
```

### Associations

We saw earlier that there are associations between many of the Active
Records (and hence service models), and we use these extensively when
scripting.

For example we can discover more about the hardware of our virtual
machine (VM) by following associations between the VM object
(`MiqAeServiceManageIQ_Providers_Redhat_InfraManager_Vm`), and its
Hardware and GuestDevice objects (`MiqAeServiceHardware` and
`MiqAeServiceGuestDevice`), as follows:

``` ruby
hardware = $evm.root['vm'].hardware
hardware.guest_devices.each do |guest_device|
  if guest_device.device_type == "ethernet"
    nic_name = guest_device.device_name
  end
end
```

Fortunately we don’t need to know anything about the Active Records or
service models behind the scenes, we just magically follow the
association. See [Investigative
Debugging](../investigative_debugging/chapter.asciidoc) to find out what
associations there are to follow.

Continuing our exploration of our RHEV Host object, the associations
available to this object are:

    host.datacenter
    host.directories
    host.ems_cluster
    host.ems_events
    host.ems_folder
    host.ext_management_system
    host.files
    host.guest_applications
    host.hardware
    host.lans
    host.operating_system
    host.storages
    host.switches
    host.vms

We can enumerate an object’s associations using:

``` ruby
this_object.associations.each do |association|
```

### Methods

Most of the objects that we work with have useful methods defined that
we can use, either in their own class or one of their parent
superclasses. For example the methods available to call for our RHEV
Host object are:

    host.authentication_password
    host.authentication_userid
    host.credentials
    host.current_cpu_usage
    host.current_memory_headroom
    host.current_memory_usage
    host.custom_get
    host.custom_keys
    host.custom_set
    host.domain
    host.ems_custom_get
    host.ems_custom_keys
    host.ems_custom_set
    host.event_log_threshold?
    host.get_realtime_metric
    host.scan
    host.ssh_exec
    host.tagged_with?
    host.tags
    host.tag_assign

Enumerating a service model object’s methods is more challenging,
because the actual object that we want to enumerate is running in the
Automation Engine on the remote side of a dRuby call (see below), and
all we have is the local DRb::DRbObject accessible from `$evm`. We can
use `method_missing`, but we get returned the entire method list, which
includes attribute names, virtual column names, association names,
superclass methods, and so on.

``` ruby
this_object.method_missing(:class).instance_methods
```

## Distributed Ruby

The Automation Engine runs in a CloudForms/ManageIQ *worker* thread, and
it launches one of our automation scripts by spawning it as a child Ruby
process. We can see this from the command line using **`ps`** to check
the PID of the worker processes and its children:

    \_ /var/www/miq/vmdb/lib/workers/bin/worker.rb
    |   \_ /opt/rh/rh-ruby22/root/usr/bin/ruby  <-- automation script running

An automation script runs in its own process space, but it must somehow
access the service model objects that reside in the Automation Engine
process. It does this using Distributed Ruby.

We can use `rake evm:status` to see which workers are running on a
CloudForms or ManageIQ appliance:

    vmdb
    bin/rake evm:status
    
    ...
     Worker Type                                                       | Status  |
    -------------------------------------------------------------------+---------+
     ManageIQ::Providers::Redhat::InfraManager::EventCatcher           | started |
     ManageIQ::Providers::Redhat::InfraManager::MetricsCollectorWorker | started |
     ManageIQ::Providers::Redhat::InfraManager::MetricsCollectorWorker | started |
     ManageIQ::Providers::Redhat::InfraManager::RefreshWorker          | started |
     MiqEmsMetricsProcessorWorker                                      | started |
     MiqEmsMetricsProcessorWorker                                      | started |
     MiqEventHandler                                                   | started |
     MiqGenericWorker                                                  | started |
     MiqGenericWorker                                                  | started |
     MiqPriorityWorker                                                 | started |
     MiqPriorityWorker                                                 | started |
     MiqReportingWorker                                                | started |
     MiqReportingWorker                                                | started |
     MiqScheduleWorker                                                 | started |
     MiqSmartProxyWorker                                               | started |
     MiqSmartProxyWorker                                               | started |
     MiqUiWorker                                                       | started |
     MiqWebServiceWorker                                               | started |

Distributed Ruby (dRuby) is a distributed client-server object system
that allows a client Ruby process to call methods on a Ruby object
located in another (server) Ruby process. This can even be on another
machine.

The object in the remote dRuby server process is locally represented in
the dRuby client by an instance of a *DRb::DRbObject* object. In the
case of an automation script, this object is our `$evm` variable.

The Automation Engine cleverly handles everything for us. When it runs
our automation script, the Engine sets up the dRuby session
automatically, and we access all of the service model objects
seamlesssly via `$evm` in our script. Behind the scenes the dRuby
library handles the TCP/IP socket communication with the dRuby server in
the worker running the Automation Engine.

We gain insight into this if we examine some of these `$evm` objects
using `object_walker`, for
    example:

    $evm.root['user'] => #<MiqAeMethodService::MiqAeServiceUser:0x0000000c5431c8>   \
                                (type: DRb::DRbObject, URI: druby://127.0.0.1:38842)

Although the use of dRuby mostly transparent to us, it can occasionally
produce unexpected results. Perhaps we are hoping to find some useful
user-related method that we can call on our user object, which we know
we can access as `$evm.root['user']`. We might try to call a standard
Ruby method such as:

``` ruby
$evm.root['user'].instance_methods
```

If we were to do this we would actually get a list of the instance
methods for the local *DRb::DRbObject* object, rather than the remote
MiqAeServiceUser service model; probably not what we want.

When we get more adventurous in our scripting, we also occasionally get
a *DRb::DRbUnknown* object returned to us, indicating that the class of
the object is unknown in our dRuby client’s namespace.

## Summary

This chapter has given us some good insight into the Rails active
records that CloudForms/ManageIQ uses internally to represent our
virtual infrastructure, and how these are made available to us as
service model objects. We’ve also seen how these service model objects
have four specific properties that we frequently make use of:
attributes, virtual columns, associations and methods.

### Further Reading

[Methods Available For
Automation](http://CloudForms/ManageIQ.org/pdf/CloudForms/ManageIQ-0-Methods_Available_for_Automation-en-US.pdf)

[Change Automate Methods to Communicate via REST
API](https://github.com/CloudForms/ManageIQ/CloudForms/ManageIQ/issues/2215)

[Support 'where' Method for Service
Models](https://github.com/CloudForms/ManageIQ/CloudForms/ManageIQ/pull/6046)

Masatoshi Seki: The dRuby Book

1.  See also [Ruby on Rails/Getting
    Started/Model-View-Controller](http://en.wikibooks.org/wiki/Ruby_on_Rails/Getting_Started/Model-View-Controller)
