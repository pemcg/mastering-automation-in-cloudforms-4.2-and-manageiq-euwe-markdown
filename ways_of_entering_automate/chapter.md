# Ways of Entering Automate

We are starting to see how powerful Automate can be, so let’s look at
the various ways that we can initiate an automation operation.

There are six methods that we generally use to launch into Automate to
run our instances or initiate our custom workflows. The method that we
choose determines the Automate Datastore entry point, and the objects
that are available to our method when it runs.

## Buttons and Simulation

So far we have launched automation scripts in two ways: from
**Simulation** and from a custom button. With either of these methods we
were presented with a drop-down list of entry points under
*/System/Process* into the Automate Datastore (see
[figure\_title](#i1)).

![Entry points into the Automate Datastore from
Simulation](images/ss4.png)

​  

In practice we only use */System/Process/Request* to launch our own
automation requests from a button or simulation. Entries at
*/System/Process/Request* get redirected to the **Request** instance
name that we specified with the call, which should be an instance in the
*/System/Request* namespace. In our examples so far we’ve used
**Request** instances of *Call\_Instance*, and *AddCustomAttribute*
(which we added to our *ACME* domain).

> **Note**
> 
> The usage of */System/Process/Event* has changed with ManageIQ
> *Capablanca*. We would need to pass an *EventStream* object with our
> request to use this entry point.
> 
> The */System/Process/Automation* entry point is used internally when
> *tasks* are created for such operations as virtual machine
> provisioning or retirement:
> 
>     Instantiating [/System/Process/AUTOMATION? \
>     MiqProvision%3A%3Amiq_provision=1000000000091& \
>     MiqServer%3A%3Amiq_server=1000000000001& \
>     User%3A%3Auser=1000000000001& \
>     object_name=AUTOMATION& \
>     request=vm_provision& \
>     vmdb_object_type=miq_provision]

## RESTful API

We can initiate an Automate operation using the RESTful API (See
[Calling Automation Using the RESTful
API](../calling_automation_using_the_restful_api/chapter.asciidoc) for
more details). In this case we can directly invoke any instance anywhere
in the Automate Datastore, we do not need to call
*/System/Process/Request*.

## Control Policy Actions

A *control policy action* can be created that launches a custom
automation instance (see [figure\_title](#i2)).

![Launching a custom automation as a control action](images/ss1.png)

​  

This can launch any instance in */System/Request*, but as before we can
use *Call\_Instance* to redirect the call via the built-in **rel2**
relationship to an instance in our own domain and namespace.

## Alerts

We can create an *alert* that sends a *management event*. The **Event
Name** field corresponds to the name of an instance that we create to
handle the alert. (see [figure\_title](#i3)).

![Creating an alert that sends a management event called
'ScaleOut'](images/ss2.png)

​  

The event name corresponds to an instance in the event switchboard under
*/System/Event/CustomEvent/Alert*. We can clone the
*/System/Event/CustomEvent/Alert* namespace into our own domain, and add
the corresponding instance (see [figure\_title](#i4)).

![Adding an instance to processs an alert management
event](images/ss3.png)

​  

This instance will now be run when the alert is triggered.

## Service Dialog Dynamic Elements

We can launch an Automate instance anywhere in the Automate datastore
from a dynamic service dialog element. In practice this type of script
is designed specifically to populate the element, and we wouldn’t launch
a general workflow in this manner. We cover dynamic service dialog
elements more in [Service Dialogs](../service_dialogs/chapter.asciidoc).

## Finding Out How Our Method Has Been Called

Our entry point into Automate governs the content of `$evm.root` - this
is the object whose instantiation took us into Automate. If we write a
generically useful method such as one that adds a disk to a virtual
machine, it might be useful to be able to call it in several ways,
without necessarily knowing what `$evm.root` might contain.

For example we might wish to add a disk during the provisioning workflow
for the VM; from a button on an existing VM object in the WebUI, or even
from an external RESTful call into the Automate Engine, passing the VM
ID as an argument. The content of `$evm.root` is different in each of
these cases.

For each of these cases we need to access the target VM object in a
different way, but we can use the `$evm.root['vmdb_object_type']` key to
help us establish context:

``` ruby
case $evm.root['vmdb_object_type']
when 'miq_provision'                  # called from a VM provision workflow
  vm = $evm.root['miq_provision'].destination
  ...
when 'vm'
  vm = $evm.root['vm']                # called from a button
  ...
when 'automation_task'                # called from a RESTful automation request
  attrs = $evm.root['automation_task'].options[:attrs]
  vm_id = attrs[:vm_id]
  vm = $evm.vmdb('vm').find_by_id(vm_id)
  ...
end
```

## Summary

In this chapter we’ve learned the various ways that we can enter
Automate and start running our scripts. We’ve also learned how to create
generically useful methods that can be called in several ways, and how
to establish their running context using
`$evm.root['vmdb_object_type']`.

Many of the Automate methods that we write are usable in several
different contexts; as part of a virtual machine provisioning workflow,
or from a button for example. They may be run from the first instance
called when we enter Automate, or via a relationship in another instance
already running in the Automation Engine. This instance might even be a
state machine (we discuss state machines in [State
Machines](../state_machines/chapter.asciidoc)), in which case we might
need to signal an exit condition using `$evm.root['ae_result']`:

``` ruby
  # Normal exit
  $evm.root['ae_result'] = 'ok'
  exit MIQ_OK
rescue => err
  $evm.root['ae_result'] = 'error'
  $evm.root['ae_reason'] = "Unspecified error, see automation.log for backtrace"
  exit MIQ_STOP
```

If we take all of these possible factors into account when we write our
scripts, we add flexibility in how they can be used and called. We
increase code reuse, and reduce the sprawl of multiple similar scripts
in our custom domains.
