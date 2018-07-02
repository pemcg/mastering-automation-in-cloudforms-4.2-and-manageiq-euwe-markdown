# Introduction to CloudForms and ManageIQ

Welcome to this guide to mastering the Automate feature of both
CloudForms and ManageIQ. Before we begin our journey through Automate,
it’s worth taking a general tour of their capabilities to establish a
context for all that we’ll be learning. As ManageIQ is the *upstream*
open source project for Red Hat CloudForms (we’ll discuss this later in
the chapter), the two products are very similar.

## What are CloudForms and ManageIQ?

CloudForms and ManageIQ are both *cloud management platforms*, that are
also rather good at managing traditional server virtualization products
such as VMware vSphere or oVirt. This broad capability makes them ideal
as *hybrid* cloud managers, able to manage both public clouds, and
on-premise private clouds and virtual infrastructures. They provides a
single management interface into a hybrid environment, enabling
cross-platform orchestration to be achieved with relative simplicity.

Although ManageIQ was originally designed as a virtualization and
*Infrastructure as a Service* (IaaS) cloud manager, CloudForms
4.0/ManageIQ *Capablanca* introduced support for Docker Container
management, including Red Hat’s OpenShift Enterprise 3.x *Platform as a
Service* (PaaS) cloud solution. With CloudForms 4.2/ManageIQ *Euwe* this
capability has been extended to manage cloud-oriented software-defined
networking and storage, as well as configuration management and
middleware components within our enterprise.

Cloud management platforms (CMPs) are software tools that allow for the
integrated management of public, private and hybrid cloud environments.
There are several requirements that are generally considered necessary
for a product to be classified as a cloud management platform. These
are:

  - Self-service catalog-based ordering of services

  - Metering and billing of cloud workloads

  - Policy-based workload optimisation and management

  - Workflow orchestration and automation

  - The capability to integrate with external tools and systems

  - Role-based access control (RBAC) and multitenancy

## Providers

CloudForms and ManageIQ manage each cloud, container or virtual
environment using modular subcomponents called *providers*. Each
provider contains the functionality required to connect to and monitor
its specific target platform, and this provider specialization enables
common cloud management functionality to be abstracted into the core
product. In keeping with the *manager of managers* concept, providers
communicate with their respective underlying cloud or infrastructure
platform using the native APIs published for the platform manager (such
as VMware vCenter Server using the vSphere SOAP API).

The pluggable nature of the provider architecture makes it relatively
straightforward to develop new providers to support additional cloud and
infrastructure platforms. Providers are broadly divided into categories,
and in CloudForms 4.2/ManageIQ *Euwe* these are *Cloud*,
*Infrastructure*, *Container*, *Configuration Management*, *Network*,
*Middleware* and *Storage*.

### Cloud Providers

Cloud providers in both CloudForms 4.2 and ManageIQ *Euwe* enable us to
manage three *public* clouds; Amazon EC2, Google Compute Engine, and
Microsoft Azure. There is also a cloud provider that can connect to and
manage a *private* or *on-premise* Red Hat OpenStack Platform (OSP)
cloud (this is the *OverCloud* in the case that OSP is managed by the
Red Hat OpenStack Platform Director). ManageIQ *Euwe* also has the
capability to add VMware vCloud as a manageable cloud provider.

### Infrastructure Providers

Infrastructure providers enable us to manage three traditional
infrastructure products: VMware vCenter Server, Red Hat (Enterprise)
Virtualization Manager, and Microsoft System Center Virtual Machine
Manager. There is also an infrastructure provider that can connect to
and manage a private or on-premise Red Hat OpenStack Platform Director
*UnderCloud*.

### Container Providers

Container providers allow us to connect to and manage Docker container
managers. CloudForms 4.2 supports OpenShift Container Platform as a
container manager, but ManageIQ *Euwe* also adds Kubernetes and
OpenShift Origin as available managers.

### Configuration Management Providers

Configuration management providers enable us to use the capabilities of
Red Hat Satellite (or Foreman), and Ansible Tower for configuration
management. The Satellite provider enables us to use Foreman *host
groups*, and extends the provisioning capability to include *bare-metal*
(i.e. non-virtual) servers. The Ansible Tower provider allows us to run
Ansible jobs as part of an Automate workflow.

### Network Providers

Network providers allow us to visualize and manage the software-defined
network components in our Amazon EC2, Google Compute Engine, Microsoft
Azure, or OpenStack clouds. Network providers are automatically
configured when we add any of the public or private cloud providers, or
OSP Director infrastructure provider.

### Storage Providers

Storage providers allow us to manage the software-defined storage
components in our OpenStack cloud. CloudForms 4.2/ManageIQ *Euwe* has
introduced support for the OpenStack Cinder (block) and Swift (object)
storage modules.

### Middleware Providers

CloudForms 4.2/ManageIQ *Euwe* has introduced a new Middleware provider
type as a technology preview. This provides inventory, metrics and
events for JBoss application servers and EAP databases, by connecting to
a suitably configured Hawkular instance.

### Mixing and Matching Providers

When deploying CloudForms or ManageIQ in our enterprise we often connect
to several providers. We can illustrate this with an example company.

#### Company XYZ Inc.

Our example organization has an established VMware vSphere 6.0 virtual
environment, containing many hundreds of virtual machines. This virtual
infrastructure is typically used for the stable, long-lived virtual
machines, and many of the organisation’s critical database, directory
services and file servers are situated here. Approximately half of the
VMware virtual machines run Red Hat Enterprise Linux \[1\], and to
facilitate the patching, updating, and configuration management of these
VMs, the organization has a Satellite 6 server.

Company XYZ is a large producer of several varieties of widget, and the
widget development test cycle involves creating many short-lived
instances in an OpenStack private cloud, to cycle through the test
suites. The developers like to have a service catalog from which they
can order one of the many widget test environments, and at any time
there may be several hundred instances running various test
configurations.

The web developers in the company are in the process of re-developing
the main Internet web portal as a scalable public cloud workload hosted
in Amazon EC2. The web portal provides a rich product catalog, online
documentation, knowlegebase and ecommerce shopping area to customers.

This organization uses CloudForms to manage the workflows that provision
virtual machines into the vSphere virtual infrastructure, Amazon EC2,
and OpenStack. The users have a self-service catalog to provision
individual virtual machine workloads into either VMware or Amazon, or
entire test suites into OpenStack. CloudForms orchestration workflows
help with the maintenance of an *image factory* that keeps virtual
machine images updated and published as VMware templates, Amazon Machine
Images (AMIs) and OpenStack *Glance* images.

As part of the provisoning process CloudForms also manages the
integration workflows that allow newly provisoned virtual machines to be
automatically registered with the Satellite 6 server, and an in-house
configuration management database. This ensures that newly provisioned
virtual machines are configured by Puppet according to server role,
patched with the latest updates, with a full inventory visible to the
help-desk system.

![CloudForms providers and workflows](images/cloudforms_ripicture.png)

​  

## The Capabilities

We’ve already mentioned some of the capabilities of CloudForms and
ManageIQ such as *orchestration*, a *service catalog*, and *integration
workflows*. Let’s have a look at the four main areas of capability:
Insight, Control, Automate and Integrate.

### Insight

*Insight* is the process of gathering intelligence on our virtual or
cloud infrastructure, so that we can manage it effectively. It is one of
the most fundamental but important capabilities of the product.

When we first connect a provider, CloudForms and ManageIQ begin a
process of *discovery* of the virtual or cloud infrastructure. An
infrastructure provider will collect and maintain details of the entire
virtual infrastructure, including clusters, hypervisors, datastores,
virtual machines, and the relationships between each of them. Cloud
vendors do not typically expose infrastructure details, so cloud
providers will typically gather and monitor tenant-specific information
on cloud components such as instances, images, availability zones,
networks, and security groups.

Both tools also store and process any real-time or historical
performance data that the provider exposes. They use the historical data
to calculate useful trend-based analytics such as image or VM
right-sizing suggestions, and capacity planning recommendations. They
use the real-time performance statistics and power-on/off events to give
us insight into workload utilisation, and also use this information to
calculate metering and chargeback costs.

One of the roles of a CloudForms or ManageIQ server is that of *Smart
Proxy*. A server with this role has the ability to initiate a
*SmartState Analysis* on a virtual machine, template, instance, or even
Docker container. SmartState Analysis (also known as *fleecing*) is a
patented technology that scans the container or virtual machine’s disk
image to examine its contents. The scan discovers users and groups that
have been added, applications that have been installed, and searches for
and optionally retrieves the contents of specified configuration files
or Windows Registry settings. This is an agentless operation that
doesn’t require the virtual machine to be powered on.

Both CloudForms and ManageIQ allow us to apply tags to infrastructure or
cloud components to help us identify and classify our workloads or
resources in a way that makes sense to our organisation. These tags
might specify an owning department, cost centre, operating system type,
location, or workload classification for example. We can create powerful
filters in the WebUI that allow us to display managed components such as
VMs along organisational and business lines, rather than physical
placement or characteristic.

To round off the summary of their Insight ability, CloudForms and
ManageIQ also have a powerful reporting capability that can be used to
create online or exportable CSV or PDF reports.

### Control

We can use the *control* functionality of CloudForms and ManageIQ to
enforce security and configuration policies, using the information
retrieved from insight. For example the SmartState Analysis of a virtual
machine might discover a software package containing a known critical
security vulnerability. We could implement a *control policy* to shut
down the VM, or migrate it to a hypervisor in a quarantined network so
that it can be patched.

Using real-time performance statistics we might configure alerts to warn
us when critical virtual machines are running at unusually high
utilisation levels. Many monitoring tools can do this, but with ManageIQ
we could also use such an alert to trigger an Automate workflow to
dynamically scale out the application workload by provisioning more
servers.

We can monitor for compliance with corporate security policies, by
gathering and intelligently processing the contents of selected
configuration files. In this way we might detect if SELinux has been
disabled for example, or that sshd is running with an insecure
configuration. We can run such compliance rules automatically, and mark
a virtual machine as *noncompliant*, whereupon its status will be
immediately visible in the WebUI.

### Automate

One of the most powerful features of CloudForms and ManageIQ are their
ability to *automate* the orchestration of workloads and resources in
our virtual infrastructure or cloud. Automate allows us to create and
use powerful workflows using the Ruby scripting language or Ansible
jobs, and features provided by the *Automation Engine* such as *state
machines* and *service models*.

CloudForms and ManageIQ come preconfigured with a large number of
out-of-the-box workflows \[2\], to orchestrate such things as:

  - Provisioning or scaling out of *workloads*, such as virtual machines
    or cloud instances

  - Provisioning or scaling out of *infrastructure*, such as bare-metal
    hypervisors or *compute nodes*

  - Scaling back or retirement of virtual machine or cloud instances

Each of these is done in the context of comprehensive role-based access
control (RBAC), with administrator-level approval of selected Automate
operations required where appropriate.

We can extend or enhance these default workflows and create whole new
orchestration workflows to meet our specific requirements.

#### Service Catalog

We can create self-service catalogs to permit users to order our
orchestration workflows with a single button click. Automate comes with
an interactive service dialog designer that we use to build rich
dialogs, containing elements such as text boxes, radio buttons or
drop-down lists. These elements can be dynamically prepopulated with
values that are specific and relevant to the logged-in user or workload
being ordered.

### Integrate

As an extension of its Automate capability, CloudForms and ManageIQ are
able to connect to and *Integrate* with many Enterprise tools and
systems. Both systems come with Ruby Gems to enable automation scripts
to connect to both RESTful and SOAP APIs, as well as libraries to
connect to several SQL and LDAP databases, and the ability to run remote
PowerShell scripts on Windows servers.

Typical integration actions might be to extend the virtual machine
provisioning workflow to retrieve and use an IP address from a corporate
IP address management (IPAM) solution; to create a new configuration
item (CI) record in the central configuration management database
(CMDB), or to create and update tickets in the enterprise Service
Management tool, such as ServiceNow.

## The Appliance

To simplify installation, both CloudForms are ManageIQ are distributed
as fully installed virtual machine templates, often just referred to as
*Appliances* for convenience. An appliance comes pre-configured with
everything we need. A CloudForms 4.2 appliance runs RHEL 7.3 (CentOS 7.3
in the case of ManageIQ *Euwe*), with PostgreSQL 9.5, Rails 5.0.0.1, the
CloudForms/ManageIQ application, and all associated Ruby gems installed.
Appliances are downloadable as a virtual machine image template in
formats suitable for VMware, Red Hat Enterprise Virtualization,
OpenStack, Amazon EC2, Microsoft’s System Center Virtual Machine Manager
or Azure cloud, and Google Compute Engine. They are also available as a
Docker container image.

### Ruby and Rails

The core "evmserverd" application is witten in Ruby on Rails, and uses
PostgreSQL as its database. When we use the Automate capability of
CloudForms or ManageIQ we work extensively with the Ruby language, and
write scripts that interact with a Ruby object model defined for us by
the Automation Engine. We certainly don’t need to be Rails developers
however (we don’t really *need* to know anything about Rails), but as
we’ll see in [Peeping Under the
Hood](../../peeping_under_the_hood/chapter.asciidoc), some understanding
of Rails concepts can make it easier to understand the object model, and
what happens behind the scenes when we run our scripts.

> **Note**
> 
> Why Rails? Ruby on Rails is a powerful development framework for
> database-centric web-based applications. It is popular for open source
> product development, for example *Foreman*, one of the core components
> of Red Hat’s *Satellite 6.x* product, is also a Rails application.

## Projects, Products and Some History

Red Hat is an open source company, and its *products* are derived from
one or more "upstream" open source projects. ManageIQ is the upstream
project for Red Hat CloudForms.

### ManageIQ (the *Project*)

The ManageIQ project releases a new version every six months
(approximately). Each version is named alphabetically after a chess
Grand Master, and so far these have been Anand, Botvinnik, Capablanca,
Darga and Euwe. At the time of writing, Euwe is the current stable
release, and Fine is in development.

### Red Hat CloudForms (the *Product*)

Red Hat CloudForms 1.0 was originally a suite of products comprising
CloudForms System Engine, CloudForms Cloud Engine and CloudForms Config
Server, each with its own upstream project.

When Red Hat acquired ManageIQ (a privately held company) in late 2012,
it decided to discontinue development of the original CloudForms 1.0
projects \[3\], and base a new version, CloudForms 2.0, on the much more
capable and mature ManageIQ Enterprise Virtualization Manager (EVM) 5.x
product. EVM 5.1 was re-branded as CloudForms Management Engine 5.1.

It took Red Hat approximately 18 months from the time of the ManageIQ
acquisition to make the source code ready to publish as an open source
project. Once completed, the ManageIQ project was formed and development
was started on the *Anand* release.

### CloudForms Management Engine (the *Appliance*)

*CloudForms Management Engine* is the name of the CloudForms virtual
appliance that we download from redhat.com. The most recent versions of
CloudForms Management Engine have been based on corresponding ManageIQ
project releases. The relative versions and releases are summarised in
the following
table:

| ManageIQ project release | ManageIQ sprints | CloudForms Management Engine version | CloudForms version |
| ------------------------ | ---------------- | ------------------------------------ | ------------------ |
|                          |                  | 5.1                                  | 2.0                |
|                          |                  | 5.2                                  | 3.0                |
| Anand                    | 1 - 12           | 5.3                                  | 3.1                |
| Botvinnik                | 13 - 22          | 5.4                                  | 3.2                |
| Capablanca               | 23 - 33          | 5.5                                  | 4.0                |
| Darga                    | 34 - 42          | 5.6                                  | 4.1                |
| Euwe                     | 43 - 51          | 5.7                                  | 4.2                |

Summary of the relative project and product versions

## Summary

This chapter has introduced both CloudForms and ManageIQ at a fairly
high level, but has hopefully established a product context in the mind
of the reader. The remainder of the book focuses specifically on the
Automate functionality of the two tools. Let’s roll up our sleeves and
get started\!

### Further Reading

[Red Hat
CloudForms](https://www.redhat.com/en/technologies/cloud-computing/cloudforms)

[A Technical Overview of Red Hat Cloud Infrastructure
(RHCI)](https://allthingsopen.com/2015/04/09/a-technical-overview-of-red-hat-cloud-infrastructure-rhci/)

[The Forrester Wave™: Hybrid Cloud Management Solutions,
Q1 2016](https://www.forrester.com/report/The+Forrester+Wave+Hybrid+Cloud+Management+Solutions+Q1+2016/-/E-RES122813)

[ManageIQ Architecture Guides - Provider
Overview](https://github.com/manageiq/guides/blob/master/architecture/providers_overview.md)

1.  CloudForms and ManageIQ are virtual machine operating system
    neutral; they can manage Windows, Red Hat, Fedora, Debian, Ubuntu or
    SUSE VMs (or their derivatives) with equal ease

2.  CloudForms actually ships with supplementary automation scripts that
    are not in ManageIQ

3.  CloudForms System Engine didn’t completely disappear. It was based
    on the upstream *Katello* project, which now forms a core part of
    Red Hat’s Satellite 6.x product
