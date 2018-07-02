# Catalog{Item,Bundle}Initialization

In [The Service Provisioning State
Machine](../the_service_provisioning_state_machine/chapter.asciidoc) we
saw that two of the service provisioning state machines instances are
called *CatalogItemInitialization* and *CatalogBundleInitialization*.
These two state machines greatly simplify the process of creating
service catalog items and bundles, fronted by rich service dialogs,
without the need for any Ruby scripting.

> **Note**
> 
> A service catalog *item* generally provisions a single type of virtual
> machine (although it may result in multiple VMs of the same type being
> provisioned). A service catalog *bundle* can provision multiple
> service items in one go, allowing us to deploy multitier server
> workloads from a single click.

In this chapter we’ll take a look at these two state machine instances
in detail. We’ll see how they allow us to name our service dialog
elements in such a way that values are automatically passed into the
provisioning workflow, with no need for further automation scripting.

## CatalogItemInitialization

We can specify the *CatalogItemInitialization* state machine as the
provisioning entry point when we create a service catalog item.

The state machine has been written to simplify the process of
customising the provisioned VMs, using values read from the service
dialog. It does this by setting options hash values or tags in the child
and grandchild `miq_request_task` objects, from specially constructed
service dialog element names. It also allows us to specify a new unique
name and description for the resulting service.

The schema for the *CatalogItemInitialization* instance is shown in
[figure\_title](#i1).

![The fields of the CatalogItemInitialization state machine
instance](images/ss1.png)

​  

We can see that the schema uses the **pre1** state to add a dialog
parser (common between the *CatalogItemInitialization* and
*CatalogBundleInitialization* state machines). It also uses the **pre2**
state to specify the *CatalogItemInitialization* method, which sets the
child object options hash values and/or tags accordingly.

The *CatalogItemInitialization* method itself relies on the dialog
inputs having been parsed correctly by *dialog\_parser*, and this
requires us to use a particular naming convention for our service dialog
elements.

### Service Dialog Element Naming Convention

To perform the service dialog → options\_hash or tag substitution
correctly, we must name our service dialog elements in a particular way.

#### Single options hash key

The simplest service dialog element to process is one that prompts for
the value of a single options hash key. We name the service dialog
element as:

**option\_0\_key\_name\_** (for backwards compatibility with CloudForms
3.1/ManageIQ *Anand*)

or just

***key\_name*** (valid for CloudForms 3.2/ManageIQ *Botvinnik* and
later)

For example we can create a service dialog element as shown in
[figure\_title](#i2).

![Service dialog element to prompt for an options hash
key](images/ss2.png)

​  

The resulting value input from this dialog at runtime will be propagated
to the child task’s options hash as:

``` ruby
miq_request_task.options[:vm_memory]
```

The '0' in the dialog name refers to the item sequence number when
provisioning. For a service dialog fronting a single catalog *item* this
will always be zero. For a service dialog fronting a catalog *bundle*
comprising several items, the sequence number indicates which of the
component items the dialog option should be passed to (an element with a
name sequence of '0' will be propagated to all items).

Several **key\_name** values are recognised and special-cased by the
*CatalogItemInitialization* method. We can name a text box element as
either **vm\_name** or **vm\_target\_name**, and the resulting text
string input value will be propagated to all of:

``` ruby
miq_request_task.options[:vm_target_name]
miq_request_task.options[:vm_target_hostname]
miq_request_task.options[:vm_name]
miq_request_task.options[:linux_host_name]
```

If we name a text box element as **service\_name**, then the resulting
service will be named from the text value of this element.

If we name a text box element as **service\_description**, then the
resulting service description will be updated from the text value of
this element.

#### Single tag

We can also create a text box service dialog element to apply a single
tag. The naming format is similar to that of naming an option, but using
a prefix of "tag\_", and a suffix of the tag category name.

For example we can prompt for a tag in the **department** category by
naming the service dialog element as **tag\_0\_department** (see
[figure\_title](#i3)).

![Service dialog element to prompt for a tag value](images/ss3.png)

​  

The value input into the service dialog element at runtime should be a
tag within this tag category. When an element of this type is processed
by the *CatalogItemInitialization* method, if either the category or tag
doesn’t currently exist, it will be created.

## CatalogBundleInitialization

The *CatalogBundleInitialization* state machine should be specified when
we create a service catalog *bundle*.

The schema for the *CatalogBundleInitialization* instance is the same as
for *CatalogItemInitialization*, except that the **pre2** stage calls
the *CatalogBundleInitialization* method.

The *CatalogBundleInitialization* method passes the service dialog
element values on to each catalog item’s *CatalogItemInitialization*
method, which is still required in order to set the miq\_request\_task’s
options hash keys for the provision of that catalog item.

## Summary

This chapter has introduced the two service provision state machines
that we can use to create service catalog items and bundles, with no
need for any Ruby scripting. We can create simple but impressive service
catalogs in minutes using these entry points, and we see a practical
example of this in [Creating a Service Catalog
Item](../creating_a_service_catalog_item/chapter.asciidoc).

### Further Reading

It is worth familiarising ourselves with the three methods that perform
the parsing and transposing of the dialog values. These are
DialogParser, CatalogItemInitialization and
CatalogBundleInitialization.

<https://github.com/ManageIQ/manageiq/blob/capablanca/db/fixtures/ae_datastore/ManageIQ/Service/Provisioning/StateMachines/Methods.class/>*methods*/dialog\_parser.rb\[DialogParser
Method\]

<https://github.com/ManageIQ/manageiq/blob/capablanca/db/fixtures/ae_datastore/ManageIQ/Service/Provisioning/StateMachines/Methods.class/>*methods*/catalogiteminitialization.rb\[CatalogItemInitialization
Method\]

<https://github.com/ManageIQ/manageiq/blob/capablanca/db/fixtures/ae_datastore/ManageIQ/Service/Provisioning/StateMachines/Methods.class/>*methods*/catalogbundleinitialization.rb\[CatalogBundleInitialization
Method\]
