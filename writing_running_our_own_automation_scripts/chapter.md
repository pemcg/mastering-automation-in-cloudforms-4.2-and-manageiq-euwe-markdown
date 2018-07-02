# Writing and Running Our Own Automation Scripts

Let’s jump right in and start writing our first automation script. In
time-honoured fashion we’ll write "Hello, World\!" to the Automate
logfile.

Before we do anything we need to ensure that the **Automation Engine**
server role is ticked on our ManageIQ appliance. We do this from the
**Configure → Configuration** menu, selecting the ManageIQ server in the
*Settings* accordion (see [figure\_title](#i1)).

> **Tip**
> 
> The **Automation Engine** server role is now enabled by default in
> CloudForms 4.0 (ManageIQ *Capablanca*) and later, but it’s still
> worthwhile to check that this role is set on our ManageIQ appliance.

![Setting the Automation Engine server role](images/ss1.png)

​  

Setting the **Automation Engine** role is necessary to be able to run
*queued* Automate tasks (this includes anything that starts off as an
automation *request*, which we’ll cover later in [Requests and
Tasks](../requests_and_tasks/chapter.asciidoc)). Automate actions
initiated directly from the WebUI - such as running instances from
simulation, or processing methods to populate dynamic dialogs - are run
on the WebUI appliance itself, regardless of whether it has the
**Automation Engine** role enabled.

Our first Automate examples in the book will be run from simulation, so
we don’t need the **Automation Engine** role to be set for these to
work. When we move on to more advanced ways of running our scripts we
will need the role enabled, so by checking that it’s set now, we’ll have
one less thing to troubleshoot as we progress through the book.

## Creating the Environment

Before we create our first automation script, we need to put some things
in place. We’ll begin by adding a new domain called *ACME*. We’ll add
all of our automation code into this new domain.

### Adding a New Domain

In the Automate Explorer, highlight the **Datastore** icon, and click
**Configuration → Add a New Domain** (see [figure\_title](#i2)).

![Adding a new domain](images/ss2.png)

​  

We’ll give the domain the **Name** *ACME*, the **Description** *ACME
Corp.*, and ensure the **Enabled** checkbox is selected.

### Adding a Namespace

Now we’ll add a namespace into this domain, called *General*. Highlight
the *ACME* domain icon in the sidebar, and click **Configuration → Add a
New Namespace** (see [figure\_title](#i3)).

![Adding a new namespace](images/ss3.png)

​  

Give the namespace the **Name** *General* and the **Description**
*General Content*

### Adding a Class

Now we’ll add a new class, called *Methods*.

> **Note**
> 
> It may seem that naming a class "Methods" is somewhat confusing,
> however many of the generic classes in the *ManageIQ* domain in the
> Automate Datastore are called "Methods" to signify their
> general-purpose nature).

Highlight the *General* domain icon in the sidebar, and click
**Configuration → Add a New Class** (see [figure\_title](#i4)).

![Adding a new class](images/ss4.png)

​  

Give the class the **Name** *Methods* and the **Description** *General
Instances and Methods*. We’ll leave the **Display Name** empty for this
example.

### Editing the Schema

We’ll create a simple schema. Click the **Schema** tab for the *Methods*
class, and click **Configuration → Edit selected Schema** (see
[figure\_title](#i5)).

![Editing the schema](images/ss5.png)

​  

Click **New Field**, and add a single field with name *execute*,
**Type** *Method* and **Data Type** *String* (see [figure\_title](#i6)).

![Adding a new schema field](images/ss6.png)

​  

Click the **tick** icon in the lefthand column to save the field entry,
and click the **Save** button to save the schema. We now have our
generic class definition called *Methods* setup, with a simple schema
that executes a single method.

## Hello, World\!

Our first Automate method is very simple, we’ll write an entry to the
*automation.log* file using a two-line script:

``` ruby
$evm.log(:info, "Hello, World!")
exit MIQ_OK
```

### Adding a New Instance

As mentioned in [Introduction to the Automate
Datastore](../introduction_to_the_automate_datastore/chapter.asciidoc)
the Automation Engine runs scripts within the context of *instances*, so
first we need to create an instance from our class. In the **Instances**
tab of the new **Methods** class, select **Configuration → Add a New
Instance** (see [figure\_title](#i8)).

![Adding a new instance to our class](images/ss8.png)

​  

We’ll call the instance *hello\_world*, and it’ll run (execute) a method
called *hello\_world* (see [figure\_title](#i9)).

![Entering the instance details](images/ss9.png)

​  

Click the **Add** button.

### Adding a New Method

In the **Methods** tab of the new *Methods* class, select
**Configuration → Add a New Method** (see [figure\_title](#i10)).

![Adding a new method to our class](images/ss10.png)

​  

Name the method *hello\_world*, and paste our two lines of code into the
**Data** window (see [figure\_title](#i11)).

![Entering the method details](images/ss11.png)

​  

Click **Validate**, and then the **Add** button.

> **Tip**
> 
> Get into the habit of using the **Validate** button, it can save a lot
> of time catching Ruby syntactical typos when you develop more complex
> scripts

## Running the Instance

We’ll run our new instance using the *Simulation* functionality of
Automate, but before we do that, login to CloudForms/ManageIQ again from
another browser or a private browsing tab, and navigate to **Automate →
Log** in the WebUI \[1\]

> **Note**
> 
> The CloudForms/ManageIQ WebUI uses browser session cookies, so if we
> want two or more concurrent login sessions (particularly as different
> users), it helps to use different web browsers or private/incognito
> windows.

In the simulation we actually run an instance called *call\_instance* in
the */System/Request/* namespace of the *ManageIQ* domain, and this in
turn calls our *hello\_world* instance using the *namespace*, *class*
and *instance* attribute/value pairs that we pass to it (see [Ways of
Entering Automate](../ways_of_entering_automate/chapter.asciidoc)).

From the **Automate → Simulation** menu, complete the details (see
[figure\_title](#i12)).

![Completing the Simulation details](images/ss12.png)

​  

Click **Submit**

If all went well, we should see our "Hello, World\!" message appear in
the *automation.log*
    file.

    Invoking [inline] method [/ACME/General/Methods/hello_world] with inputs [{}]
    <AEMethod [/ACME/General/Methods/hello_world]> Starting
    <AEMethod hello_world> Hello, World!
    <AEMethod [/ACME/General/Methods/hello_world]> Ending
    Method exited with rc=MIQ_OK

Success\!

## Exit Status Codes

In our example we used an exit status code of MIQ\_OK. Although with
simple methods such as this we don’t strictly need to specify an exit
code, it’s good practice to do so. When we build more advanced
multimethod classes and state machines, an exit code can signal an error
condition to the Automation Engine so that action can be taken.

There are four exit codes that we can use:

**MIQ\_OK** (0) - Continues normal processing. This is logged to
*automation.log* as:

    Method exited with rc=MIQ_OK

**MIQ\_WARN** (4) - Warning message, continues processing. This is
logged to *automation.log* as:

    Method exited with rc=MIQ_WARN

**MIQ\_ERROR / MIQ\_STOP** (8) - Stops processing current object. This
is logged to *automation.log* as:

    Stopping instantiation because [Method exited with rc=MIQ_STOP]

**MIQ\_ABORT** (16) - Aborts entire automation instantiation. This is
logged to *automation.log* as:

    Aborting instantiation because [Method exited with rc=MIQ_ABORT]

> **Note**
> 
> The difference between MIQ\_STOP and MIQ\_ABORT is subtle, but comes
> into play as we develop more advanced Automate workflows.
> 
> MIQ\_STOP stops the currently running instance, but if this instance
> was called via a reference from another ‘parent’ instance, the
> subsequent steps in the parent instance would still complete.
> 
> MIQ\_ABORT stops the currently running instance and any parent
> instance that called it, terminating the Automate task altogether.

## Summary

In this chapter we’ve seen how simple it is to create our own domain,
namespace, class, instance and method, and run our script from
simulation. These are the fundamental techniques that we use for all of
our automation scripts, and we’ll use this knowledge extensively as we
progress through the book.

We’ve also discovered the status codes that we should use to pass our
exit status back to the Automation Engine.

1.  Alternatively ssh into the appliance as *root*, and `tail -f
    /var/www/miq/vmdb/log/automation.log`
