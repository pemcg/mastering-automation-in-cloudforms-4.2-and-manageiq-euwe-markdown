# $evm and the Workspace

When we write automation scripts, we access the Automation Engine and
all of its objects through a single `$evm` variable \[1\]. This is
sometimes referred to as the *Workspace*.

As discussed in [Peeping Under the
Hood](../peeping_under_the_hood/chapter.asciidoc), the `$evm` variable
is a *DRb::DRbObject* object representing a dRuby client connection back
to the Automation Engine. The object at the dRuby server side of our
`$evm` variable is an instance of an `MiqAeService` object, which
contains over 40 methods. In practice we generally use only a few of
these methods, most commonly:

    $evm.root
    $evm.object
    $evm.current (this is equivalent to calling $evm.object(nil))
    $evm.parent
    $evm.log
    $evm.vmdb
    $evm.execute
    $evm.instantiate

We will look at these methods in more detail in the next sections.

## $evm.log

`$evm.log` is a simple method that we’ve used already. It writes a
message to *automation.log*, and accepts two arguments: a log level and
the text string to write. The log level can be written as a Ruby symbol
(e.g. `:info`, `:warn`), or as a text string (e.g. "info", "warn").

## $evm.root

`$evm.root` is the method that returns to us the root object in the
workspace (environment, variables, linked objects etc.). This is the
instance whose invocation took us into the Automation Engine. From
`$evm.root` we can access other service model objects such as
`$evm.root['vm']`, `$evm.root['user']`, or `$evm.root['miq_request']`
(the actual objects available depend on the context of the Automate
tasks that we are performing).

![The object model](images/object_model.png)

​  

`$evm.root` contains a lot of useful information that we use
programmatically to establish our running context - for example to see
if we’ve been called by an API call or from a button (see also
[Investigative Debugging](../investigative_debugging/chapter.asciidoc))

    $evm.root['vmdb_object_type'] = vm   (type: String)
    ...
    $evm.root['ae_provider_category'] = infrastructure   (type: String)
    ...
    $evm.root.namespace = ManageIQ/SYSTEM   (type: String)
    $evm.root['object_name'] = Request   (type: String)
    $evm.root['request'] = Call_Instance   (type: String)
    $evm.root['instance'] = object_walker   (type: String)

`$evm.root` also contains any variables that were defined on our entry
into the Automation Engine, such as the `$evm.root['dialog*']` variables
that were defined from our service dialog.

## $evm.object, $evm.current and $evm.parent

As we saw, `$evm.root` returns to us the object representing the
instance that was launched when we entered Automate. Many instances have
schemas that contain relationships to other instances, and as each
relationship is followed, a new child object is created under the
calling object to represent the called instance. These objects all live
in the same workspace, they share the same `$evm` variable. Fortunately
we can access any of the objects in this parent-child hierarchy using
`$evm.object`.

Calling `$evm.object` with no arguments returns the currently
instantiated/running instance. As automation scripters we can think of
this as "our currently running code", and we can also access it using
the alias `$evm.current`. When we wanted to access our 'username' schema
variable in [Using Schema
Variables](../using_schema_variables/chapter.asciidoc), we accessed it
using `$evm.object['username']`.

We can access our parent object (the one that called us) using
`$evm.object("..")`, or the alias `$evm.parent`.

> **Note**
> 
> `$evm.root` is the same as `$evm.object("/")`

When we ran our first example script, *hello\_world* (from Simulation),
we specified an entry point of */System/Process/Request*, and our
Request was to an instance called *Call\_Instance*. We passsed to this
the namespace, class and instance that we wanted it to run (via a
relationship).

This would have resulted in an object hierarchy (when viewed from the
*hello\_world* method) as follows:

``` 
     --- object hierarchy ---
     $evm.root = /ManageIQ/SYSTEM/PROCESS/Request
       $evm.parent = /ManageIQ/System/Request/Call_Instance
         $evm.object = /ACME/General/Methods/hello_world
```

## $evm.vmdb

`$evm.vmdb` is a useful method that can be used to retrieve any *service
model* object (see [Peeping Under the
Hood](../peeping_under_the_hood/chapter.asciidoc)). The method can be
called with one or two arguments.

### Single Argument Form

When called with a single argument, the method returns the generic
service model object type, and we can use any of the Rails helper
methods (see [Peeping Under the
Hood](../peeping_under_the_hood/chapter.asciidoc)) to search by database
column name:

``` ruby
vm = $evm.vmdb('vm').find_by_name('websrv031')
vm = $evm.vmdb('vm').where(:name => 'websrv031')
$evm.vmdb(:EmsCluster).all.each do |cluster|
```

We can also use more advanced query syntax to return results based on
multiple conditions, for example:

``` ruby
$evm.vmdb(:CloudTenant).where(["ems_id = ? AND name = ?", 2, 'admin'])
```

The service model object name can be specified in CamelCase (e.g.
'AvailabilityZone') or snake\_case (e.g. 'availability\_zone'), and can
be a string or symbol.

### Two-Argument Form

If we wish to find an object by its ID, we can use the two argument form
of the call. When called with two arguments, the second argument should
be the service model ID to search for, like so:

``` ruby
owner = $evm.vmdb('user', evm_owner_id)
```

We should exercise caution when using the two-argument form. If there is
no service model matching the specified ID, the method will raise a
`MiqAeException::ServiceNotFound` exception rather than return `nil`. We
can guard against this by catching the exception ourselves, as follows:

``` ruby
owner = $evm.vmdb('user', evm_owner_id) rescue nil
```

**Question:** When should we use 'vm' (`:Vm`) or 'vm\_or\_template'
(`:VmOrTemplate`) in our `$evm.vmdb` searches?

**Answer:** Searching for a 'vm\_or\_template'
(`MiqAeServiceVmOrTemplate`) object will return both virtual machines
*and* templates that satisfy the search criteria, whereas searching for
a 'vm' object (`MiqAeServiceVm`) will only return virtual machines.
Think about whether you need both returned.

There are some subtle differences between the objects. `MiqAeServiceVm`
is a subclass of `MiqAeServiceVmOrTemplate` that adds 2 additional
methods that are not relevant for templates: `add_to_service` and
`remove_from_service`.

Both `MiqAeServiceVmOrTemplate` and `MiqAeServiceVm` have a boolean
attribute `template`, which is *true* for an image or template, and
*false* for a VM.

## $evm.execute

We can use `$evm.execute` to call one of 13 miscellaneous but useful
methods. The methods are defined in service model called *Methods*
(`MiqAeServiceMethods`), and are as follows:

  - `send_email(to, from, subject, body, content_type = nil)`

  - `snmp_trap_v1(inputs)`

  - `snmp_trap_v2(inputs)`

  - `category_exists?(category)`

  - `category_create(options = {})`

  - `tag_exists?(category, entry)`

  - `tag_create(category, options = {})`

  - `service_now_eccq_insert(server, username, password, agent, queue,
    topic, name, source, *params)`

  - `service_now_task_get_records(server, username, password, *params)`

  - `service_now_task_update(server, username, password, *params)`

  - `service_now_task_service(service, server, username, password,
    *params)`

  - `create_provision_request(*args)`

  - `create_automation_request(options, userid = "admin", auto_approve =
    false)`

### Examples

We can see some examples of calling these methods.

#### Creating a tag if one doesn’t already exist

``` ruby
unless $evm.execute('tag_exists?', 'cost_centre', '3376')
  $evm.execute('tag_create', "cost_centre", :name => '3376',
                                            :description => '3376')
end
```

In this example we call the `tag_exists?` method to see if the tag
'cost\_centre/3376' exists. If it doesn’t (i.e. `tag_exists?` returns
`false`), then we call the `tag_create` method to create the tag,
passing the tag category arguments, `:name` and `:description`.

#### Sending an Email

``` ruby
to = 'pemcg@redhat.com'
from = 'miq01@uk.bit63.com'
subject = 'Test Message'
body = 'What an awesome cloud management product!'
$evm.execute('send_email', to, from, subject, body)
```

Here we define the 'to', 'from', 'subject' and 'body' arguments, and
call the `send_email` method.

#### Creating a new automation request

The `create_automation_request` method is new with ManageIQ
*Capablanca*, and it enables us to chain automation requests together.
This is also very useful when we wish to explicitly launch an automation
task in a different zone than the one in which our currently running
script resides.

``` ruby
options = {}
options[:namespace]     = 'Stuff'
options[:class_name]    = 'Methods'
options[:instance_name] = 'MyInstance'
options[:user_id]       = $evm.vmdb(:user).find_by_userid('pemcg').id
# options[:attrs]       = attrs
# options[:miq_zone]    = zone
auto_approve            = true

$evm.execute('create_automation_request', options, 'admin', auto_approve)
```

In this example we define the namespace, class and instance names to be
used for the automation request, and we lookup the service model object
of the user who we want to run the automation task as. The 'admin' user
in the argument list is the *requester* to be used for approval
purposes.

## $evm.instantiate

We can use `$evm.instantiate` to launch another Automate instance
programmatically from a running method, by specifying its URI within the
Automate namespace e.g.

``` ruby
$evm.instantiate('/Discovery/ObjectWalker/object_walker')
```

Instances called in this way execute synchronously, so the calling
method waits for completion before continuing. The called instance also
appears as a child object of the caller (it sees the caller as its
`$evm.parent`).

## Summary

This has been a more theoretical chapter, examining the eight most
commonly used `$evm` methods.\[2\] In our simple scripts so far we have
already used three of them; `$evm.log`, `$evm.object` and `$evm.root`.
Our next example in [Enforcing Anti-Affinity
Rules](../enforcing_anti_affinity_rules/chapter.asciidoc) uses two
others, and we will use the remaining three as we progress through the
book. These methods form a core part of our scripting toolbag, their use
will become second nature as we advance our automation scripting skills.

### Further Reading

[class
MiqAeService](https://github.com/ManageIQ/manageiq/blob/capablanca/lib/miq_automation_engine/engine/miq_ae_service.rb)

1.  The original ManageIQ product was called *Enterprise Virtualization
    Manager*, often abbreviated to "EVM".

2.  There are a further three state-machine specific $evm methods that
    we frequently use, but we’ll cover those in [State
    Machines](../state_machines/chapter.asciidoc)
