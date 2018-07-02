# Investigative Debugging

As we saw in [Peeping Under the
Hood](../peeping_under_the_hood/chapter.asciidoc), there is a lot of
useful information in the form of service model object attributes and
virtual columns, links to other objects via associations, and service
model methods that we can call. The challenge is sometimes knowing which
objects are available to us at any particular point in our workflow, and
how to access their properties and traverse associations to find the
information that we need.

Fortunately there are several ways of exploring the object structure in
the Automation Engine, both to investigate what might be available to
use, and to debug and troubleshoot our own automation code. This chapter
will discuss the various tools that we can use to reveal the service
model object structure available during any Automate operation.

## InspectMe

*InspectMe* is an instance/method combination supplied out-of-the-box
that we can call to dump some attributes of `$evm.root` and its
associated objects. As an example we can call InspectMe from a button on
a *VM and Instance* object as we did when running our
*AddCustomAttribute* instance in [Working with Virtual
Machines](../working_with_virtual_machines/chapter.asciidoc). As both
the instance and method are in the `ManageIQ/System/Request` namespace,
we can call InspectMe directly rather than calling `Call_Instance` as an
intermediary.

We can view the InspectMe output in *automation.log*:

    # vmdb
    # grep inspectme log/automation.log | awk 'FS="INFO -- :" {print $2}'
    Root:<$evm.root> Attributes - Begin
      Attribute - ae_provider_category: infrastructure
      Attribute - miq_server: #<MiqAeMethodService::MiqAeServiceMiqServer:0x000000...
      Attribute - miq_server_id: 1000000000001
      Attribute - object_name: Request
      Attribute - request: InspectMe
      Attribute - user: #<MiqAeMethodService::MiqAeServiceUser:0x0000000b86b540>
      Attribute - user_id: 1000000000001
      Attribute - vm: rhel7srv001
      Attribute - vm_id: 1000000000025
      Attribute - vmdb_object_type: vm
    Root:<$evm.root> Attributes - End
    ...

This log snippet shows the section of a typical InspectMe output that
dumps the attributes of `$evm.root`.

> **Tip**
> 
> Kevin Morey from Red Hat has written an improved version of InspectMe,
> available from <https://github.com/ramrexx/CloudForms_Essentials>. His
> InspectMe provides a very clear output of the attributes, virtual
> columns, associations and tags that are available for the object on
> which it was launched. For example if called from a button on a
> virtual machine, InspectMe will list all of that VM’s properties
> including hardware, operating system, and also the provisioning
> request and task details.

## object\_walker

*object\_walker* \[1\]is a slightly more exploratory tool that walks and
dumps the objects, attributes and virtual columns of `$evm.root` and its
immediate objects. It also recursively traverses associations to walk
and dump any objects that it finds, much like a web crawler would
explore a web site. It prints the output in a Ruby-like syntax that can
be copied and pasted directly into an Automation script to access or
walk the same path.

### Black or Whitelisting Associations

One of the features of object\_walker is the ability to selectively
choose which associations to "walk" to limit the output. We do so by
setting a `walk_association_policy` to `"whitelist"` or `"blacklist"`,
and then defining a `walk_association_whitelist` or
`walk_association_blacklist` JSON hash in the instance schema to list
the associations to be walked (whitelist), or not walked (blacklist).

In practice a `walk_association_policy` of `"blacklist"` produces so
much output that it’s rarely used, and so a `"whitelist"` is more often
defined, like
so:

``` ruby
{ 'MiqAeServiceManageIQ_Providers_Redhat_InfraManager_Vm': ['hardware', 'host', 'storage'],
'MiqAeServiceManageIQ_Providers_Vmware_InfraManager_Vm': ['hardware', 'host', 'storage'],
'MiqAeServiceHardware': ['nics', 'guest_devices', 'ports', 'storage_adapters' ],
'MiqAeServiceGuestDevice': ['hardware', 'lan', 'network'] }
```

### object\_walker\_reader

There is a companion script, *object\_walker\_reader*, that can be
copied to the CloudForms or ManageIQ appliance to extract the
object\_walker outputs from *automation.log*. The reader can also list
all outputs by timestamp, dump a particular output by timestamp, and
even *diff* two outputs - useful when running object\_walker before and
after a built-in method (for example in a State Machine) to see what the
method has changed.

    Object Walker 1.8 Starting
         --- walk_association_policy details ---
         walk_association_policy = whitelist
         walk_association_whitelist = { 'MiqAeServiceMiqProvisionRequest': ['miq_request',...
         --- $evm.current_* details ---
         $evm.current_namespace = Bit63/Stuff   (type: String)
         $evm.current_class = ObjectWalker   (type: String)
         $evm.current_instance = object_walker   (type: String)
         $evm.current_method = object_walker   (type: String)
         $evm.current_message = create   (type: String)
         $evm.current_object = /Bit63/Stuff/ObjectWalker/object_walker   (type: DRb::DRbObject ...
         $evm.current_object.current_field_name = execute   (type: String)
         $evm.current_object.current_field_type = method   (type: String)
         --- automation instance hierarchy ---
         /ManageIQ/System/Process/AUTOMATION  ($evm.root)
         |    /ManageIQ/infrastructure/VM/Lifecycle/Provisioning
         |    |    /ManageIQ/Infrastructure/VM/Provisioning/Profile/EvmGroup-super_administrator
         |    |    /Bit63/Infrastructure/VM/Provisioning/StateMachines/VMProvision_vm/template  ($evm.parent)
         |    |    |    /ManageIQ/Infrastructure/VM/Provisioning/StateMachines/Methods/CustomizeRequest
         |    |    |    /ManageIQ/Infrastructure/VM/Provisioning/Placement/default
         |    |    |    /Bit63/Stuff/ObjectWalker/object_walker  ($evm.object)
         --- walking $evm.root ---
         $evm.root = /ManageIQ/System/Process/AUTOMATION
         |    --- attributes follow ---
         |    $evm.root['ae_next_state'] =    (type: String)
         |    $evm.root['ae_provider_category'] = infrastructure   (type: String)
         |    $evm.root['ae_result'] = ok   (type: String)
         |    $evm.root['ae_state'] = WalkObjects   (type: String)
         |    $evm.root['ae_state_retries'] = 0   (type: Fixnum)
         |    $evm.root['ae_state_started'] = 2016-08-01 09:55:59 UTC   (type: String)
         |    $evm.root['ae_state_step'] = main   (type: String)
         |    $evm.root['ae_status_state'] = on_exit   (type: String)
         |    $evm.root['miq_group'] => #<MiqAeMethodService::MiqAeServiceMiqGroup:0x00...
         |    |    --- attributes follow ---
         |    |    $evm.root['miq_group'].created_on = 2016-05-25 08:09:35 UTC
         |    |    $evm.root['miq_group'].description = EvmGroup-super_administrator   (type: String)
         |    |    $evm.root['miq_group'].filters = nil
         |    |    $evm.root['miq_group'].group_type = system   (type: String)
         |    |    $evm.root['miq_group'].id = 2   (type: Fixnum)
         |    |    $evm.root['miq_group'].sequence = 1   (type: Fixnum)
         |    |    $evm.root['miq_group'].settings = nil
         |    |    $evm.root['miq_group'].tenant_id = 1   (type: Fixnum)
         |    |    $evm.root['miq_group'].updated_on = 2016-05-25 08:09:35 UTC
         |    |    --- end of attributes ---
         |    |    --- virtual columns follow ---
         |    |    $evm.root['miq_group'].allocated_memory = 1073741824   (type: Fixnum)
         |    |    $evm.root['miq_group'].allocated_storage = 42949672960   (type: Fixnum)
         |    |    $evm.root['miq_group'].allocated_vcpu = 1   (type: Fixnum)

Here we see a partial output from *object\_walker\_reader*, showing the
traversal of the associations between objects, and list of attributes,
virtual columns, associations and methods for each object encountered.

## Rails console

We can connect to the Rails console to have a look around.

> **Caution**
> 
> When we’re working with the Rails command line, we have full
> read/write access to the objects and tables that we find there. We
> should purely use this technique for read-only investigation, and at
> our own risk. Making any additions or changes may render our appliance
> unstable.

On the CloudForms or ManageIQ appliance
    itself:

    # vmdb   # alias vmdb='cd /var/www/miq/vmdb/' is defined on the appliance
    # source /etc/default/evm
    # bin/rails c
    Loading production environment (Rails 3.2.17)
    irb(main):001:0>

Once in the Rails console there are many things that we can do, such as
use Rails object syntax to look at all *Host* active records:

    irb(main):002:0> Host.all
       (3.6ms)  SELECT version()
      Host Load (0.7ms)  SELECT "hosts".* FROM "hosts"
      Host Inst (85.2ms - 2rows)
    => [#<HostRedhat id: 1000000000002, name: "rhelh02.bit63.net", \
                            hostname: "192.168.12.22", ipaddress: "192.168.12.22",...
    
    irb(main):003:0>

We can even generate our own `$evm` variable that matches the Automation
Engine
default:

``` ruby
$evm=MiqAeMethodService::MiqAeService.new(MiqAeEngine::MiqAeWorkspaceRuntime.new)
```

With our `$evm` variable we can emulate actions that we perform from an
automation script:

    irb(main):002:0> $evm.log(:info, "test from the Rails console")
    => true

As with a "real" Automation Method, this writes our message to
*automation.log*:

    ...8:45:11.223058 #2109:eb9998]  INFO -- : <AEMethod > test from the Rails console

## Rails db

It is occasionally useful to be able to examine some of the database
tables (such as to look for column headers that we can find\_by\_\* on)
\[2\]. We can connect to Rails db, which puts us directly into a psql
session:

    [root@miq03 ~]# vmdb
    [root@miq03 vmdb]# source /etc/default/evm
    [root@miq03 vmdb]# bin/rails db
    psql (9.4.5)
    Type "help" for help.
    
    vmdb_production=#

Once in the Rails db session we can freely examine the VMDB database.
For example we could look at the columns in the `guest_devices` table:

    vmdb_production=# \d guest_devices
                                          Table "public.guest_devices"
          Column       |          Type          |               Modifiers
    -------------------+------------------------+------------------------------------
     id                | bigint                 | not null default nextval('guest_...
     device_name       | character varying(255) |
     device_type       | character varying(255) |
     location          | character varying(255) |
     filename          | character varying(255) |
     hardware_id       | bigint                 |
     mode              | character varying(255) |
     controller_type   | character varying(255) |
     size              | bigint                 |
     free_space        | bigint                 |
     size_on_disk      | bigint                 |
     address           | character varying(255) |
     switch_id         | bigint                 |
     lan_id            | bigint                 |
    ...

We could list all templates on our appliance (templates are in the `vms`
column, but have a boolean `template` attribute that is true):

    vmdb_production=# select id,name from vms where template = 't';
          id       |                  name
    ---------------+----------------------------------------
     1000000000014 | RedHat_CFME-5.5.0.13
     1000000000015 | rhel7-generic
     1000000000016 | rhel-guest-image-7.0-20140930.0.x86_64
     1000000000017 | RHEL 7
     1000000000029 | ManageIQ_Capablanca
     1000000000053 | Fedora 23
    (6 rows)

## Summary

In this chapter we’ve learned four very useful ways of investigating the
object model. We can use `InspectMe`, or `object_walker` to print the
structure to *automation.log*, or we can interactively use the Rails
command line.

We use these tools and techniques extensively when developing our
scripts, both to find out the available objects that we might use, and
also to debug our scripts when things are not working as expected.

### Further Reading

[inspectXML – Dump Objects as
XML](http://cloudformsblog.redhat.com/tag/xml-format/)

1.  object\_walker is available from
    <https://github.com/pemcg/object_walker>, along with instructions
    for use

2.  A diagram of the database layout is available from
    <http://people.redhat.com/~mmorsi/cfme_db.png>
