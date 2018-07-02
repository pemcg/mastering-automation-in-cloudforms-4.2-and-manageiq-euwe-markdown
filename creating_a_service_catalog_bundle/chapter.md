# Creating a Service Catalog Bundle

We learnt in [Creating a Service Catalog
Item](../creating_a_service_catalog_item/chapter.asciidoc) how to create
service catalog items that enable our users to provision fully
configured virtual machines from a single **Order** button.

We can populate our service catalog with useful items (see
[figure\_title](#i1)).

![Service catalog containing three services](images/ss1.png)

​  

In these examples the virtual machines are provisioned from fully
installed VMware templates, preconfigured with the application packages.
The service dialog purely prompts for the Service and VM Names (see
[figure\_title](#i2)).

![Service dialog for each catalog item](images/ss2.png)

​  

The next logical step is to be able to provision several items together
as a single *service catalog bundle*.

## Creating the Service Dialog for the Bundle

When we create a service catalog bundle, we handle the dialog input for
each of the catalog items in a single service dialog that we create for
the bundle. For our Web, Middleware and Database Server items, we must
prompt for the VM name of each, but we’ll also prompt for a service name
(see [figure\_title](#i3)).

![Service dialog for a catalog bundle](images/ss3.png)

​  

We name the dialog elements according to the sequence in which we want
our individual items provisioned. Our sequence will be:

1.  Database Server

2.  Middleware Server

3.  Web Server

Our four dialog elements are therefore constructed as follows. We’ll
create a text box element to prompt for Service Name (see
[figure\_title](#i4)).

![Dialog element to prompt for service name](images/ss4.png)

​  

We add a second text box element to prompt for Web Server Name (see
[figure\_title](#i5)).

![Dialog element to prompt for web server name](images/ss5.png)

​  

We add a third text box element to prompt for Middleware Server Name
(see [figure\_title](#i6)).

![Dialog element to prompt for middleware server name](images/ss6.png)

​  

Finally we add a fourth text box element to prompt for Database Server
Name (see [figure\_title](#i7)).

![Dialog element to prompt for database server name](images/ss7.png)

​  

The number in the element name reflects the sequence number, and the
*CatalogItemInitialization* and *CatalogBundleInitialization* methods
use this sequence number to pass the dialog value to the correct
grandchild miq\_request\_task (see [The Service Provisioning State
Machine](../the_service_provisioning_state_machine/chapter.asciidoc)).

The value **option\_\<n\>\_vm\_name** is recognised and special-cased by
*CatalogItemInitialization*, which sets both the `:vm_target_name` and
`:vm_target_hostname` keys in the miq\_request\_task’s options hash to
the value input.

The `:vm_target_name` key sets the name of the resulting virtual
machine.

The `:vm_target_hostname` key can be used to inject a Linux *hostname*
(i.e. FQDN) into a VMware Customization Specification, which can then
set this in the virtual machine using VMware Tools on firstboot.

## Preparing the Service Catalog Items

As we will be handling dialog input when the bundle is ordered, we need
to edit each catalog item to set the **Catalog** to **\<Unassigned\>**,
and the **Dialog** to **\<No Dialog\>**. We also *deselect* the
**Display in Catalog** option as we no longer want this item to be
individually orderable (see [figure\_title](#i8)).

![Preparing the existing service catalog items](images/ss8.png)

​  

Once we’ve done this, the items will appear as **Unassigned** (see
[figure\_title](#i9)).

![Unassigned catalog items](images/ss9.png)

​  

## Creating the Service Catalog Bundle

Now we can go ahead and create our catalog bundle. Highlight a catalog
name, and select **Configuration → Add a New Catalog Bundle** (see
[figure\_title](#i10)).

![Adding a new catalog bundle](images/ss10.png)

​  

Enter a name and description for the bundle, then select the **Display
in Catalog** checkbox. Select an appropriate catalog, and the newly
created bundle dialog, from the appropriate drop-downs.

For the Provisioning Entry Point, navigate to
*ManageIQ/Service/Provisioning/StateMachines/ServiceProvision\_Template/CatalogBundleInitialization*
(see [figure\_title](#i12)).

![Service bundle basic info](images/ss11.png)

​  

Click on the **Details** tab, and enter some HTML-formatted text to
describe the catalog item to anyone viewing in the catalog.

``` html
<h1>Three Tier Intranet Server Bundle</h1>
<hr>
<p>Deploy a <strong>Web, Middleware</strong> and <strong>Database</strong>
                 server together as a single service</p>
```

Click on the **Resources** tab, and select each of the three unassigned
catalog items to add them to the bundle (see [figure\_title](#i13)).

![Adding resources to the bundle](images/ss12.png)

​  

Change the **Action Order** and **Provisioning Order** according to our
desired sequence ('3' won’t be visible until '2' is set for an option)
see [figure\_title](#i14). The sequence should match the
**option\_\<n\>\_vm\_name** sequence that we gave our dialog elements.

![Setting the action and provision orders](images/ss13.png)

​  

Finally click the **Add** button.

Select a suitable sized icon for a custom image, and save.

## Ordering the Catalog Bundle

Navigate to the **Service Catalogs** section in the accordion, expand
the **Intranet Services** catalog, and highlight the **Three Tier
Intranet Server Bundle** catalog item (see [figure\_title](#i16)).

![Ordering the catalog bundle](images/ss14.png)

​  

Click **Order**, and fill out the service dialog values (see
[figure\_title](#i17)).

![Entering the service and server names in the service
dialog](images/ss15.png)

​  

Click **Submit**

After a new minutes, the new service should be visible in **My
Services**, containing the new VMs (see [figure\_title](#i18)).

![The completed service](images/ss16.png)

​  

If we weren’t watching the order that the VMs were created in, we could
look in the database to check that our desired provisioning sequence was
followed:

    vmdb_production=# select id,name from vms order by id asc;
          id       |                     name
    ---------------+----------------------------------------------
    ...
     1000000000177 | jst-db01
     1000000000178 | jst-mid01
     1000000000179 | jst-web01

Here we see that the VMs were created (and named) in the correct order.

## Summary

This has been a useful example that shows the flexibility of service
catalogs to deploy entire application bundles. When we link this concept
to a configuration management tool such as Puppet running from Red Hat
Satellite 6, we start to really see the power of automation in our
enterprise. We can deploy complex workloads from a single button click.

One of the cool features of service bundles is that we can mix and match
catalog items that provision into different providers. For example we
may have a Bimodal IT \[1\] infrastructure comprising RHEV for our
traditional Mode 1 workloads, and an in-house OpenStack private cloud
for our more cloud-ready Mode 2 workloads. Using service bundles we
could provision our relatively static servers into RHEV, and our
dynamically scalable mid-tier and frontend servers into OpenStack.

### Further Reading

[Filtering out service catalog items during
deployment](http://talk.manageiq.org/t/filtering-out-service-catalog-items-during-deployment/725)

1.  <http://www.gartner.com/it-glossary/bimodal/>
