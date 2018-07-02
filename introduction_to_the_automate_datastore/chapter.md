# Introduction to the Automate Datastore

When we use the Automate capability of CloudForms/ManageIQ, we write
scripts in the Ruby language, and use objects that the *Automation
Engine* makes available to us. The CloudForms/ManageIQ Web User
Interface (WebUI) allows us to access the Automate functionality via the
**Automate** top-level menu (see [figure\_title](#i1)).

![Automate top-level menu for CloudForms 4.2](images/ss1.png)

​  

## The Automate Explorer

The first menu item that we see takes us to the *Explorer*. This is our
visual interface into the *Automate Datastore*, and it contains the
various kinds of Automate objects that we’ll use throughout this book
(see [figure\_title](#i2)).

![Automate Explorer](images/ss2.png)

​  

Before we start our journey into learning CloudForms/ManageIQ Automate,
we’ll take a tour of the Automate Datastore to familiarise ourselves
with the objects that we’ll find there.

## The Automate Datastore

The Automate Datastore has a directory-like structure, consisting of
several types of organisational unit arranged in a hierarchy (see
[figure\_title](#i3)).

![Automate datastore icon styles](images/datastore.png)

​  

We can look at each of these types of object in more detail.

## Domains

A *domain* is a collection of namespaces, classes, instances and
methods. The ManageIQ project provides a single *ManageIQ* domain for
all supplied automation code, whilst Red Hat adds the supplemental
*RedHat* domain containing added-value code for the CloudForms product.
The ManageIQ and RedHat domains are locked, indicating their read-only
nature, however we can create new domains for our own custom automation
code. [figure\_title](#i2) shows the default domains, two additional
manually-created domains: *Bit63* and *Debug*, and a Git-imported domain
*Investigative\_Debugging*.

Organising our own code into custom domains greatly simplifies the task
of exporting and importing code (simplifying code portability and
reuse). It also leaves Red Hat or the ManageIQ project free to update
the locked domain through minor releases without fear of overwriting our
customisations.

> **Note**
> 
> If we are logged into CloudForms/ManageIQ as an account in a child
> tenant, we may see domains created by a parent tenant in the Automate
> Datastore, but they will also appear as locked.

### Domain Priority

User-added domains can be individually enabled or disabled, and can be
ordered by priority such that if code exists in the same path in
multiple domains (for example */Cloud/VM/Provisioning/StateMachines*),
the code in the highest priority enabled domain will be executed. We can
change the priority order of our user-added domains using the
**Configuration → Edit Priority Order of Domains** menu (see
[figure\_title](#i4)).

![Editing the priority order of domains](images/ss3.png)

​  

### Importing and Exporting Domains

We can export domains using *rake* from the command line, and import
them either using rake or from the WebUI. (Using rake enables us to
specify more import and export options). A typical rake import line is
as
    follows:

    bin/rake evm:automate:import YAML_FILE=bit63.yaml IMPORT_AS=Bit63 SYSTEM=false \
    ENABLED=true DOMAIN=Export PREVIEW=false

#### Importing Domains from a Git Repository

CloudForms 4.2/ManageIQ *Euwe* has introduced the capability to be able
to import domains directly from a git repository. We can specify the
repository URL and optional credentials at the Import / Export menu,
whereupon the repository contents is downloaded to the appliance with
the *Git Repositories Owner* server role set. We can then choose to
import from a repository branch or tag (see [figure\_title](#i5)).

![Selecting a branch or tag to import](images/ss12.png)

​  

A domain imported from a git repository has a github icon (see
[figure\_title](#i2)), and will be locked. The recommended way of
updating code in such a domain is to commit and push an update to the
git repository as a new tag or branch, and then update the domain using
the **Configuration → Refresh with a new branch or tag** menu.

### Copying Objects Between Domains

We frequently need to customise code in one of the the locked domains,
for example when implementing our own custom VM Placement method.
Fortunately we can easily copy any object from a locked domain into our
own, using **Configuration → Copy this …​** (see [figure\_title](#i6)).

![Copying a class](images/ss4.png)

​  

When we copy an object such as a class, we are prompted for the **From**
and **To** domains. We can optionally deselect **Copy to same path** and
specify our own destination path for the object (see
[figure\_title](#i7)).

![Specifying the destination domain and path](images/ss5.png)

​  

### Importing Old Format Exports

Domains were a new feature of the Automate Datastore in CloudForms
3.1/ManageIQ *Anand*. Prior to this release all factory-supplied and
user-created automation code was contained in a common structure, which
made updates difficult when any user-added code was introduced (the
user-supplied modifications needed exporting and reimporting/merging
whenever an automation update was released).

To import a Datastore backup from a CloudForms 3.0 and prior format
Datastore, we must convert it to the new Datastore format first, like
so:

    cd /var/www/miq/vmdb
    bin/rake evm:automate:convert FILE=database.xml DOMAIN=SAMPLE \
    ZIP_FILE=/tmp/sample_converted.zip

## Namespaces

A *namespace* is a folder-like container for classes, instances and
methods, and is used purely for organisational purposes. We create
namespaces to arrange our code logically and namespaces often contain
other namespaces (see [figure\_title](#i8)).

![Namespaces](images/ss6.png)

​  

## Classes

A *class* is similar to a template, it contains a generic definition for
a set of automation operations. Each class has a schema, that defines
the variables, states, relationships or methods that instances of the
class will use.

> **Note**
> 
> The Automate Datastore uses object-oriented terminology for these
> objects. A *class* is a generic definition for a set of automation
> operations, and these classes are *instantiated* as specific
> instances. The classes that we work with in the Automate Datastore are
> not the same as Ruby classes that we work with in our automation
> scripts.

### Schemas

A *schema* is made up of a number of elements, or *fields*, that
describe the properties of the class. A schema often has just one entry
- to run a single method - but in many cases it has several components.
[figure\_title](#i9) shows the schema for a *placement* class, which has
several different field types.

![A more complex schema](images/ss7.png)

​  

### Adding or Editing a Schema

We add or edit each schema field in the schema editor by specifying the
**Type** from a drop-down list (see [figure\_title](#i10)).

![Schema field type](images/ss8.png)

​  

Each field type has an associated **Data Type** which is also selectable
from a drop-down list (see [figure\_title](#i11)).

![Schema field data type](images/ss9.png)

​  

CloudForms 4.2/ManageIQ *Euwe* has introduced several new data types:
*Null Coalescing*, *Host*, *Vm*, *Storage*, *Ems*, *Policy*, *Server*,
*Request*, *Provision* and *User*. These will be discussed in a later
chapter.

#### Default Values

We can define default values for fields in a class schema. These will be
inherited by all instances created from the class, but can be optionally
overridden in the schema of any particular instance.

### Relationships

One of the schema field types is a *relationship*, which links to other
instances elsewhere in the Automate Datastore. We often use
relationships as a way of chaining instances together, and relationship
values can accept variable substitutions for flexibility (see
[figure\_title](#i12)).

![Relationship fields showing variable substitutions](images/ss10.png)

​  

## Instances

An *instance* is a specific *instantiation* or "clone" of the generic
class, and is the entity run by the Automation Engine. An instance
contains a copy of the class schema but with actual values of the fields
filled in (see [figure\_title](#i13)).

![Single class definition with three instances](images/ss11.png)

​  

## Methods

A *method* is a self-contained block of Ruby code that gets executed
when we run any automation operation. A typical method looks like this:

``` ruby
###################################################################
#
# Description: select the cloud network
#              Default availability zone is provided by Openstack
#
###################################################################

# Get variables
prov  = $evm.root["miq_provision"]
image = prov.vm_template
raise "Image not specified" if image.nil?

if prov.get_option(:cloud_network).nil?
  cloud_network = prov.eligible_cloud_networks.first
  if cloud_network
    prov.set_cloud_network(cloud_network)
    $evm.log("info", "Image=[#{image.name}] Cloud Network=[#{cloud_network.name}]")
  end
end
```

Methods can have one of three *Location* values: **inline**,
**builtin**, or **URI**. In practice most of the methods that we create
are **inline** methods, which means they run as a separate Ruby process
outside of Rails.

## Summary

In this chapter we’ve learned about the fundamental objects or
organisational units that we work with in the Automate Datastore:
domains, namespaces, classes, instances and methods.

We are now ready to use this information to write our first automation
script.

### Further Reading

[Scripting Actions in
CloudForms](https://access.redhat.com/documentation/en-us/red_hat_cloudforms/4.2/html/scripting_actions_in_cloudforms/)

[CloudForms 3.1 Exporting Automate
Domains](https://access.redhat.com/solutions/1225313)

[CloudForms 3.1 Importing Automate
Domains](https://access.redhat.com/solutions/1225383)

[CloudForms 3.1 Automate Model
Conversion](https://access.redhat.com/solutions/1225413)
