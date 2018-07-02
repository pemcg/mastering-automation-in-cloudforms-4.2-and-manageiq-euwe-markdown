# Service Retirement

We saw in [VM and Instance
Retirement](../vm_instance_retirement/chapter.asciidoc) how individual
virtual machines or instances can be retired from their **Lifecycle**
menu button, and we can also retire services in the same way. The
service retirement process follows a similar workflow to the VM
retirement process, but we have the flexibility to specify per-service
retirement state machines if we wish.

## Defining a Service Retirement Entry Point

When we create a service catalog item, we can optionally specify a
retirement entry point (see [figure\_title](#i1)).

![Setting a service retirement entry point state
machine](images/ss3.png)

​  

If we specify our own retirement entry point, then this state machine
will be used to retire any services created from this catalog item. If
we do not specify our own entry point here then then the *Default*
retirement state machine will be used..

## Initiating Retirement

Service retirement is initiated from the Lifecycle menu on the service
details frame (see [figure\_title](#i2)).

![Service Retirement Menu](images/ss1.png)

​  

Clicking on **Retire this Service** raises a
**request\_service\_retire** event that begins a chain of relationships
through the datastore:

  - **request\_service\_retire** →
    
      - */System/Event/MiqEvent/POLICY/request\_service\_retire*
        →
    
      - */Service/Retirement/StateMachines/Methods/GetRetirementEntrypoint*

*GetRetirementEntrypoint* runs a method *get\_retirement\_entry\_point*
that returns the retirement entry point state machine defined when the
service catalog item was created (see [figure\_title](#i3)). If this is
empty then */Service/Retirement/StateMachines/ServiceRetirement/Default*
is returned.

## Retirement-Related Attributes and Methods

A service object has a number of retirement-related methods:

    $evm.root['service'].automate_retirement_entrypoint
    $evm.root['service'].start_retirement
    $evm.root['service'].finish_retirement
    $evm.root['service'].retire_now
    $evm.root['service'].retire_service_resources
    $evm.root['service'].retiring?
    $evm.root['service'].retired?
    $evm.root['service'].error_retiring?
    $evm.root['service'].retirement_state=
    $evm.root['service'].retirement_warn=
    $evm.root['service'].retires_on=

and attributes:

    $evm.root['service'].retired = nil
    $evm.root['service'].retirement_last_warn = nil
    $evm.root['service'].retirement_requester = nil
    $evm.root['service'].retirement_state = nil
    $evm.root['service'].retirement_warn = nil
    $evm.root['service'].retires_on = nil

## Service Retirement State Machine

The *Default* service retirement state machine is simpler than its VM
counterpart (see [figure\_title](#i3))

![Fields of the default service retirement state
machine](images/ss5.png)

​  

### StartRetirement

The *StartRetirement* instance calls the *start\_retirement* state
machine method, which checks whether the service is already in state
*retired* or *retiring*, and if so it aborts. If in neither of these
states it calls the service’s `start_retirement` method, which sets the
`retirement_state` attribute to 'retiring'.

### RetireService/CheckServiceRetired

The *RetireService* instance calls the *retire\_service* state machine
method, which in turn calls the service object’s
`retire_service_resources` method. This method calls the `retire_now`
method of every VM comprising the service, to initiate their retirement.
**CheckServiceRetired** retries the stage until all VMs are retired or
deleted.

### FinishRetirement

The **FinishRetirement** state sets the following Service object
attributes:

    :retires_on       => Date.today
    :retired          => true
    :retirement_state => "retired"

It also raises a **service\_retired** event that can be caught by an
Automate action or control policy.

### DeleteServiceFromVMDB

The *DeleteServiceFromVMDB* instance calls the
*delete\_service\_from\_vmdb* state machine method, which removes the
service record from the VMDB.

## Summary

We have seen in this chapter how the process of retiring a service will
also trigger the retirement of its virtual machines. If we are using
service hierarchies however, or services to manage cloud-style workloads
as single entities, this might not be our desired behaviour.

Fortunately the service retirement mechanism is flexible enough that we
can create per-service retirement state machines that we can customise
to suit our individual use cases and workloads.
