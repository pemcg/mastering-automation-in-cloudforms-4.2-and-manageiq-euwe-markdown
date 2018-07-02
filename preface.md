# Preface

Red Hat CloudForms, and its "upstream" sibling ManageIQ, are powerful
cloud management platforms that allows us to efficiently manage our
virtual infrastructure and Infrastructure as a Service (IaaS) clouds. A
significant part of this efficiency comes from automating many of the
day-to-day tasks that would otherwise require manual involvement, or
time-consuming and possibly error-prone repetitive steps.

This book is an introduction and how-to guide to working with the
*Automate* feature of CloudForms 4.2 and its corresponding ManageIQ
release *Euwe*.

Automate simplifies our lives and increases our operational efficiency.
It allows us to do such things as:

  - Eliminate many of the manual decisions and operations involved in
    provisioning virtual machines and cloud instances.

  - Load-balance our virtual machines across our virtual infrastructure
    to match our organisation’s way of working, be it logical (e.g. cost
    centre, department), operational (e.g. infrastructure lifecycle
    environment), or categorical (e.g. server role or virtual machine
    characteristic).

  - Create service catalogs to allow our users to provision virtual
    machines from a single *Order* button.

  - Create auto-scalable cloud applications where new virtual machines
    are dynamically provisioned on demand.

  - Manage our complete virtual machine lifecycle.

  - Integrate our virtual machine provisioning workflow with the wider
    enterprise, for example automatically registering new virtual
    machines with a Red Hat Satellite server.

  - Implement intelligent virtual machine retirement workflows that
    de-allocate resources such as IP addresses, and unregister from
    directory services.

# A Brief Word on Terminology

This book refers to *Automate* as the CloudForms or ManageIQ capability
or product feature, and *automation* as the thing that Automate allows
us to do. The *Automation Engine* allows us to create intelligent
automation workflows, and run *automation scripts* written in Ruby, or
Ansible playbooks.

# Who Should Read This Book

This book will appeal to cloud or virtualisation administrators who are
interested in automating parts of their virtual infrastructure or cloud
computing environment. Although primarily aimed at those with some
familiarity with CloudForms or ManageIQ, many of the concepts and terms
such as *orchestration* and *automation workflows* will be easily
understood even to those unfamiliar with the product.

Automate can be one of the more challenging aspects of the tools to
master. The practitioner requires an unusual blend of skills; a
familiarity with traditional "infrastructure" concepts such as virtual
machines, hypervisors, and tenant networks, but also a flair for
scripting in Ruby and mastery of a programming object model. There is no
black magic however, and all of the skills can be learned if we are
shown the way.

The book assumes a reasonable level of competence with the Ruby language
on the part of the reader. There are many good on-line Ruby tutorials
available, including Codecademy’s [Learn to program in
Ruby](http://www.codecademy.com/tracks/ruby).

The book also presumes a comfortable level of working experience and
familiarity of the Web User Interface (WebUI) features of either
CloudForms or ManageIQ, particularly Insight, Control, tagging, and
provisioning VMs via the **Lifecycle → Provision VMs** entry point. Many
of these features will be automated as we follow the examples in the
book, and so an understanding of why tagging is useful (for example) is
helpful.

> **Note**
> 
> Both CloudForms and ManageIQ are web applications, so interaction is
> predominantly via the browser-based WebUI. We only use a command line
> terminal when we initially configure a CloudForms appliance, or when
> troubleshooting or examining logfiles.

# Navigating This Book

The book is divided into seven parts.

## Part I "Working With Automate"

Chapter 1, *Introduction to CloudForms and ManageIQ*, sets the scene and
describes the capabilities of CloudForms and ManageIQ as cloud
management platforms.

Chapter 2, *Introduction to the Automate Datastore*, takes us on a tour
of the objects that we work with when we use the Automate capabilities
of CloudForms and ManageIQ.

Chapter 3, *Writing and Running Our Own Automate Scripts*, introduces us
to writing automation scripts in Ruby, with a simple "Hello, World\!"
example.

Chapter 4, *Using Schema Variables*, shows how we can use our instance’s
schema to store and retrieve variables.

Chapter 5, *Working with Virtual Machines*, demonstrates how to work
with an Automation Engine virtual machine object, and how to run an
automation script from a custom button in the Web User Interface.

Chapter 6, *Peeping Under the Hood*, introduces some background theory
about Rails *models*, and how CloudForms and ManageIQ abstract these as
*Service Models* that we work with when we write our automation scripts.

Chapter 7, *$evm and the Workspace*, takes us on a tour of the useful
`$evm` methods that we frequently use when scripting, such as
`$evm.vmdb` and `$evm.object`.

Chapter 8, *A Practical Example: Enforcing Anti-Affinity Rules*, is a
real-world full-script example of how we could use the techniques learnt
so far to implement anti-affinity rules in our virtual infrastructure,
based on tags.

Chapter 9, *Using Tags from Automate*, describes in detail how we can
create, assign, read, and work with tags from our Ruby automation
scripts.

Chapter 10, *Investigative Debugging*, discusses the ways that we can
discover which Automate objects are available to us when scripting. This
is useful both from an investigative viewpoint when developing scripts,
but also for debugging our scripts when things are not working as
expected.

Chapter 11, *Ways of Entering Automate*, shows us the various workflow
entry points into the Automate Datastore. It also illustrates how we can
determine programmatically the way that our automation script has been
called, so enabling us to create re-usable scripts.

Chapter 12, *Requests and Tasks*, illustrates how more advanced Automate
operations are separated into a *Request* stage, which requires
administrative approval to progress into the *Task* stage. The
corresponding request and task objects are described, and their usage
compared.

Chapter 13, *State Machines*, introduces us to state machines, and how
we can use them to intelligently sequence our workflows.

Chapter 14, *More Advanced Schema and Instance Features*, discusses the
more advanced but less frequently used schema and instance features;
Messages, Assertions, Collections, and the .missing instance.

Chapter 15, *Event Processing*, describes the way that CloudForms and
ManageIQ respond to external events such as a virtual machine shutting
down, and traces the event handling sequence through the Automate
Datastore. It also shows how Automate manages its own internal events
such as `request_started`.

## Part II "Provisioning Virtual Machines"

Chapter 16, *Provisioning a Virtual Machine*, introduces concept of
virtual machine provisioning, the most complex out-of-the-box Automate
operation that is performed by CloudForms and ManageIQ.

Chapter 17, *The Provisioning Profile*, describes how the provisioning
profile is referenced to determine the initial group-specific processing
that is performed at the start of a virtual machine provisioning
operation.

Chapter 18, *Approval*, shows how the approval workflow operates, and
how we can adjust the auto-approval criteria such as the number of
virtual machines to be provisioned, or the amount of storage, to suit
our needs.

Chapter 19, *Quota Management*, gives details of the CloudForms and
ManageIQ quota handling mechanism, and how it enables us to establish
quotas for tenants or groups.

Chapter 20, *The Options Hash*, explains the importance of a data
structure called the *options hash*, and how we can use it to retrieve
and store variables to customise the virtual machine provisioning
operation.

Chapter 21, *The Provisioning State Machine*, discusses the stages in
the state machine that governs the sequence of operations involved in
provisioning a virtual machine.

Chapter 22, *Customising Virtual Machine Provisioning*, is a practical
example showing how we can customise the state machine and include our
own Methods to add a second hard disk during the virtual machine
provisioning operation.

Chapter 23, *Virtual Machine Naming During Provisioning*, explains how
we can customise the *naming* logic that determines the name given to
the newly provisioned virtual machine.

Chapter 24, *Virtual Machine Placement During Provisioning*, explains
how we can customise the *placement* logic that determines the host,
cluster and datastore locations for our newly provisioned virtual
machine.

Chapter 25, *The Provisioning Dialog*, describes the WebUI dialogs that
prompt for the parameters that are required before a new virtual machine
can be provisioned. The chapter also explains how the dialogs can be
customised to expand optional ranges for items like size of memory, or
to present a cut down bespoke dialog to certain user groups.

Chapter 26, *Virtual Machine Provisioning Objects*, details the four
main objects that we work with when we write Ruby scripts to interact
with the virtual machine provisioning process.

Chapter 27, *Creating Provisioning Requests Programmatically*, shows how
we can initiate a virtual machine provisioning operation from an
automation script, instead of the Web User Interface.

## Part III "Automation using Ansible Tower"

Chapter 28, *Automation using Ansible*, introduces some Ansible concepts
and describes the Tower features that we use when automating using
Ansible, such as inventories, roles and job templates.

Chapter 29, *Tower-Related Automate Components*, describes the Automate
datastore components and service models that are used when using Ansible
for automation tasks.

Chapter 30, *Running an Ansible Tower Job from a Button*, is a practical
example of creating a job template on an Ansible Tower server, and then
running it on a VM from a WebUI button in CloudForms. Chapter 31,
*Integrating with Satellite 6 During Provisioning*, is an example
showing how an Ansible playbook can be used to register a newly created
virtual machine with Red Hat Satellte 6, either as a *host* or *content
host* (or both).

## Part IV "Working with Services"

Chapter 32, *Service Dialogs*, introduces the components that make up a
*service dialog*, including elements that can be dynamically populated
by Ruby methods.

Chapter 33, *The Service Provisioning State Machine*, discusses the
stages in the state machine that governs the sequence of operations
involved in creating a service.

Chapter 34, *Catalog{Item,Bundle}Initialization*, describes two specific
instances of the service provisioning state machine, that have been
designed to simplify the process of creating service catalog *items* and
*bundles*.

Chapter 35, *Approval and Quota*, shows the approval workflow for
services, and how the new consolidated quota handling mechanism also
applies to services.

Chapter 36, *Creating a Service Catalog Item*, is a practical example
showing how to create a service catalog item to provision a virtual
machine.

Chapter 37, *Creating a Service Catalog Bundle*, is a practical example
showing how to create a service catalog bundle of three virtual
machines.

Chapter 38, *Ansible Tower Services*, describes the Automate datastore
components and service models that are used when we create services that
run Ansible Tower job templates.

Chapter 39, *Creating an Ansible Tower Service Catalog Item and Bundle*,
illustrates how to create a service catalog bundle that provisions a new
virtual machine, and runs an Ansible Tower job template on it afterwards
to configure the LAMP stack software components.

Chapter 40, *Service Objects*, is an exposé of the various objects that
work behind the scenes when a service catalog item is provisioned.

Chapter 41, *Log Analysis During Service Provisioning*, is a
step-by-step walk-through, tracing the lines written to *automation.log*
at various stages of a service provision operation. This can help our
understanding of the several levels of concurrent state machine activity
taking place.

Chapter 42, *Service Hierarchies*, illustrates how services can contain
other services, and we can arrange our service groups into hierarchies
for organisational and management convenience.

Chapter 43, *Service Reconfiguration*, describes how we can create
reconfigurable services. These are capable of accepting configuration
parameters at order time via the service dialog, and can later be
reconfigured with new configuration parameters via the same service
dialog.

Chapter 44, *Service Tips and Tricks*, mentions some useful tips to
remember when developing services.

## Part V "Retirement"

Chapter 45, *Virtual Machine and Instance Retirement*, discusses the
retirement process for virtual machines and instances.

Chapter 46, *Service Retirement*, discusses the retirement process for
services.

## Part VI "Integration"

Chapter 47, *Calling Automate from the RESTful API*, shows how we can
make external calls *into* CloudForms or ManageIQ to run Automate
Instances via the RESTful API. We can also return results to our caller
in this way, enabling us to create our own pseudo-API endpoints within
the two platforms.

Chapter 48, *Automation Request Approval*, explains how to customise the
default approval behaviour for automation requests, so that
nonadministrators can submit RESTful API requests without needing
administrative approval.

Chapter 49, *Calling External Services*, shows the various ways that we
can call *out* from Automate to integrate with our wider enterprise.
This includes making outbound REST and SOAP calls, connecting to MySQL
databases, and interacting with OpenStack using the *fog* gem.

## Part VII "Miscellaneous"

Chapter 50, *Tenancy and Automate*, describes the CloudForms/ManageIQ
tenancy model, and how we sometimes need to adapt our automation
scripting to work with the tenancy concept.

Chapter 51, *Distributed Automation Processing*, describes how Automate
has been designed to be horizontally scalable. The chapter describes the
mechanism by which automation requests are distributed between multiple
appliances in a Region.

Chapter 52, *Argument Passing and Handling*, explains how arguments are
passed to, and handled internally by Automate methods for each of the
different ways that we’ve called them up to this point in the book.

Chapter 53, *Miscellaneous Tips*, closes the book with some useful tips
for Automate Method development.

# Online Resources

There are several online resources that any student of CloudForms or
ManageIQ Automate should be aware of.

## Official Documentation

The official documentation for CloudForms is here:
<https://access.redhat.com/documentation/en/red-hat-cloudforms/>

The official documentation for ManageIQ is here:
<http://manageiq.org/documentation/>

## Code Repositories

One of the best sources of reference material is the excellent
*CloudForms\_Essentials* code collection maintained by Kevin Morey from
Red Hat (<https://github.com/ramrexx/CloudForms_Essentials>). This
contains a wealth of useful code samples, and many of the examples in
this book have originated from this source.

There is also the very useful Red Hat Consulting
(<https://github.com/rhtconsulting>) GitHub repository maintained by
several Red Hat consultants.

## Fora

The ManageIQ project hosts the *ManageIQ Talk* forum at
<http://talk.manageiq.org>

## Blogs

There are several blogs that have good CloudForms and ManageIQ-related
articles, including some useful *notes from the field*. These include:

  - CloudForms NOW (<http://cloudformsblog.redhat.com/>)

  - Christian’s Blog (<http://www.jung-christian.de>)

  - Laurent Domb OSS Blog (<http://blog.domb.net/>)

  - ALL THINGS OPEN (<http://allthingsopen.com/>)

  - TigerIQ (<http://www.tigeriq.co/>)

# Conventions Used in This Book

The following typographical conventions are used in this book:

  - *Italic*  
    Indicates new terms, URLs, email addresses, filenames, and file
    extensions, path and object names within the Automate Datastore,
    Schema field values

  - **Bold**  
    Indicates WebUI components, event names, Schema field names

  - Constant width  
    Used for program listings, as well as within paragraphs to refer to
    program elements such as variable or function names, databases, data
    types, environment variables, statements, and keywords.

  - **`Constant width bold`**  
    Shows commands or other text that should be typed literally by the
    user.

  - *Constant width italic*  
    Shows text that should be replaced with user-supplied values or by
    values determined by context.

> **Note**
> 
> This icon signifies a general note.

> **Tip**
> 
> This icon signifies a tip or suggestion

> **Warning**
> 
> This icon indicates a warning or caution.
