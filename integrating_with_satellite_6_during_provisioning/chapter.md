# Integrating with Satellite 6 During Provisioning

It is a fairly common requirement to register newly provisioned Red Hat
Enterprise Linux virtual machines directly with Satellite 6 as part of
the provisioning process. This ensures that the resultant VM is patched
and up to date, and optionally configured by Puppet according to a
server role.

This chapter describes the steps involved in adapting the provisioning
workflow so that Red Hat virtual machines are automatically registered
with Satellite 6 at the end of the provisioning operation. We’ll run an
Ansible Tower job template from our provisioning state machine to
perform the registration. This simple use case demonstrates the
capability of CloudForms and ManageIQ to *Integrate* with our wider
enterprise.

## Hosts and Content Hosts

Satellite 6.2 has two different types of host record. We can add the
server as a *content host*, which associates one or more Red Hat
Subscriptions with the server, and makes software package repository
content available. We can also add a new system as a plain *host*, which
registers the server as a provisioning and configuration management
client, manageable by Puppet.

If we just wish to add the newly provisioned system as a content host,
we use the **subscription-manager** tool on the new virtual machine to
register with the Satellite. If we wish to add the newly provisioned
system as both a host *and* content host, we can use the new
*bootstrap.py* script that is supplied with Satellite 6.2.

## Satellite 6 Preparation

We need to do some preparation on our Satellite 6 server. To keep the
example simple we’ll only be working with Red Hat Enterprise Linux 7,
and we’ll create some generic host groups and activation keys in
Satellite that we can use.

### Host Groups

We’ll create some simple host groups in Satellite 6 (see
[figure\_title](#i1)).

![Satellite 6.2 host groups](images/ss1.png)

​  

Although the host groups will define default values for architecture,
operating system, media, partition table, domain and root password,
these will have no effect in the newly registered virtual machine as
they are parameters used during kickstart. The host groups will however
define the Puppet environment that the newly attached Satellite client
will use for configuration. The *Generic\_RHEL7\_Servers* hostgroup will
install the *motd* and *ntp* Puppet modules.

In this example we’ll be using two configuration management providers;
Ansible Tower and Satellite 6. Each has its own configuration management
tool: Tower uses Ansible, whereas Satellite 6.2 uses Puppet. Each tool
has its strengths (and passionate advocates), and although there is a
degree of overlap in their capabilities, using both together is often a
valid choice in a modern enterprise.

### Activation Keys

To register a newly provisioned system with Satellite as a *content
host*, we can include an activation key name as an argument to
**subscription-manager** or the *bootstrap.py* script. The activation
key determines the lifecycle environment, content view, Red Hat
subscriptions, and repository content sets to which the newly registered
server has visibility. It can be useful to create activation keys for
different types of server that require specific combinations of product
repositories.

We’ll create a small selection of activation keys in Satellite 6 (see
[figure\_title](#i2)).

![Satellite 6.2 activation keys](images/ss2.png)

​  

The **RHEL7-Generic** activation key will be used for this example. The
Satellite client will:

  - Be assigned to the **RHEL 7.2 Q4 2016** content view in the
    **Production** lifecycle environment

  - Be granted an enterprise Red Hat subscription

  - Have the following product content repositories enabled:
    
      - Red Hat Enterprise Linux 7 Server (RPMs)
    
      - Red Hat Enterprise Linux 7 Server - RH Common (RPMs)
    
      - Red Hat Satellite Tools 6.2 (for RHEL 7 Server) (RPMs)
    
      - PuppetForge Modules

## Ansible Tower Preparation

We must do some preparation on our Ansible Tower server.

### Inventory

We’ll use the same inventory credential and inventory that we created in
[Running an Ansible Tower Job From a
Button](../running_an_ansible_tower_job_from_a_button/chapter.asciidoc).

### Machine Credential

We’ll make a slight change to the **SSH Key (ansible-remote)** machine
credential. We’ll add the vault password for the vault file in which
we’ll store the Satellite 6 Admin user’s password. When decrypted this
will be used for the the playbook variable `vault_admin_pass`.

### Job Template

From the project that we created in [Running an Ansible Tower Job From a
Button](../running_an_ansible_tower_job_from_a_button/chapter.asciidoc),
we’ll create a job template called **Satellite 6 Client**. It will be a
**Run** job type; use the **CloudForms VMs** inventory and **SSH Key
(ansible-remote)** machine credential that we defined previously, and
will execute the *satellite\_client.yaml* project playbook (see
[figure\_title](#i5)).

![Job template](images/ss5.png)

​  

We’ll define some defaults for the extra variables that will be passed
to the playbook:

  - sat6\_ip (Satellite 6 server IP address)

  - sat6\_fqdn (Satellite 6 server fully-qualified domain name)

  - admin\_user (Admin-level user for registration with Satellite 6)

  - organization (Satellite 6 organization to join)

  - location (Satellite 6 location to join)

  - hostgroup (Satellite 6 configuration hostgroup to use, or 'false'
    for no Puppet configuration)

  - activationkey (Satellite 6 activation key to use)

  - updatehost ('true' or 'false')

We must also ensure that **Prompt on Launch** is checked, to allow the
variables to be overridden from CloudForms/ManageIQ if we wish (see
[figure\_title](#i7)).

![Default extra variables](images/ss7.png)

​  

This playbook uses an additional variable called `admin_pass`. The value
for this should be stored as an encrypted string, and so we defined it
in a vault file as the encrypted variable `vault_admin_pass`. We can
create this using the following commands on the Tower server:

    su - awx
    cd /var/lib/awx/projects/<project_dir>/roles/satellite_client/group_vars/all
    ansible-vault create vault
    New Vault password:
    Confirm New Vault password:
    vault_admin_pass: secret_password
    ~
    ~
    ~

We then add the vault password to the machine credential that we created
earlier.

> **Note**
> 
> Adding our own local vault file to the project directory will prevent
> the project from cleanly performing an SCM update unless we enable the
> 'Clean' SCM update project option.

## CloudForms/ManageIQ Preparation

We must also do some preparation of our CloudForms or ManageIQ
appliances.

### cloud-init Customization Template

We need our newly provisioned virtual machine to be configured as an
Ansible manageable host, so we’ll use the cloud-init template described
in [Automation Using
Ansible](../automation_using_ansible/chapter.asciidoc). We’ll specify
this template in the provisioning dialog when we provision our VM.

### Service Dialog and Button

Before we integrate the new playbook into our virtual machine
provisioning workflow, it is useful to be able to test its functionality
from a button on a VM object as we did in [Running an Ansible Tower Job
From a
Button](../running_an_ansible_tower_job_from_a_button/chapter.asciidoc).
This will allow us to troubleshoot its operation, and will also add
useful functionality to our VM-related button group.

Once again we’ll create a service dialog from the Ansible job template.
We’ll give the new service dialog the name "Satellite 6 Client" so that
we can identify it as coming from the job template. We can delete the
**Options** box and its **Limit** element as before, and this time we’ll
also edit the **hostgroup** element to change 't' to 'true' and untick
the 'Read only' checkbox. Similarly we’ll edit the **updatehost**
element to change 'f' to 'false'.

Having created the dialog, we can add a button to our VM button group.
Our button will use the new "Satellite 6 Client" dialog, and will call
the ansible\_tower\_job instance.

Once defined we can use this button to test the integration with Ansible
Tower.

![Button added to button group](images/ss13.png)

​  

### JobTemplate Instance

We’ll clone the
*/ConfigurationManagement/AnsibleTower/Operations/JobTemplate* class
into our domain, and add a new instance of this class called
*satellite\_6\_client*. We’ll add the value "Satellite 6 Client" as the
job template name, and for our first test we’ll leave all of the
**param** fields blank. By not passing any of these parameters to Tower,
we ensure that the default job template extra variables that we’ve
defined within Ansible Tower are used for the running job.

![Fields of the satellite\_6\_client instance](images/ss10.png)

​  

### register\_satellite

We only want to register a new virtual machine with our Satellite 6
server if it’s running the Red Hat Enterprise Linux (RHEL) operating
system. Fortunately we can use a template property called
`operating_system.distribution` to determine whether our template is
true RHEL, a clone (such as CentOS), or another distribution or
operating system entirely.

> **Note**
> 
> We must run a SmartState Analysis on all of our templates for the
> `operating_system.distribution` property to be populated.

We’ll create a new class */Integration/Satellite/AnsibleMethods* in our
domain, and a new instance of this class called *register\_satellite*.
We can put an assertion in our *register\_satellite* instance to
evaluate the `operating_system.distribution` property and compare it
with the string "redhat". The execution of the Ansible job template will
only proceed if this assertion evaluates to `true`.

The schema of *register\_satellite* is shown in [figure\_title](#i11).

![Fields of the register\_satellite instance](images/ss11.png)

​  

### Modify the Provisioning Workflow

We must add an additional state to the *VMProvision\_VM* state machine
schema at some point after the VM has been provisioned, called
**RegisterSatellite**. We’ll edit a cloned copy of the *template*
instance of this state machine in our domain, to add our
*/Integration/Satellite/AnsibleMethods/register\_satellite* instance to
the **RegisterSatellite** state (see [figure\_title](#i12)).

![Fields of the VMProvision\_VM/template state machine
instance](images/ss12.png)

​  

## Testing the Integration

We’ll test the integration changes that we’ve made in three ways.

### Test 1 - Registering a RHEL 7.2 Server for Content Management

Our first test is to provision a new RHEL 7 VM called "ralsrv001", and
have it register with Satellite 6 purely for package content management.
We’ll use a fully provisioned 'fat' RHEL 7.2 template called
'rhel72-generic' as our source to provision from, and we’ll select a
**Provision Type** of 'Native Clone'. The template has the cloud-init
package installed and configured.

To ensure that the new server is automatically provisioned as an Ansible
managed host, we’ll select the **Setup for Ansible Tower Management**
cloud-init script in the **Customize** tab of the provisioning dialog
(see [figure\_title](#i16)).

![Selecting the cloud-init template](images/ss16.png)

​  

We’ll also complete the **Root Password** and **Host Name** fields as
these values are passed to the cloud-init script (see
[figure\_title](#i15)).

![Customize tab of the provisioning dialog](images/ss15.png)

​  

The absence of any overridden parameters in our initial
*satellite\_6\_client* instance means that the default value of 'false'
for the **hostgroup** extra variable will be used. When this value is
passed to the Ansible playbook, the server is registered with Satellite
6 as a content host using **subscription-manager**.

If we examine *automation.log* while the server is provisioning, we see
our assertion being evaluated to **true** and the Ansible job template
being called:

    Evaluating substituted assertion ["redhat" == "redhat"]
    Q-task_id([miq_provision_183]) Following Relationship [miqaedb: \
      /ConfigurationManagement/AnsibleTower/Operations/JobTemplate/satellite_6_client#create]

On the Tower server we can see the progress of the
    job:

    Identity added: /tmp/ansible_tower_xeMhte/credential (/tmp/ansible_tower_xeMhte/credential)
    Vault password:
    
    PLAY [all] *********************************************************************
    
    TASK [setup] *******************************************************************
    ok: [ralsrv001]
    
    TASK [satellite_client : Workaround for a non working DNS] *********************
    changed: [ralsrv001]
    
    TASK [satellite_client : Download bootstrap.py from satellite01.bit63.net] *****
    skipping: [ralsrv001]
    
    TASK [satellite_client : Copy bootstrap.py script to /usr/local/sbin and make it executable] ***
    skipping: [ralsrv001]
    
    TASK [satellite_client : Register to Satellite 6 with puppet enabled and add it to the correct hostgroup] ***
    skipping: [ralsrv001]
    
    TASK [satellite_client : Install the katello-ca-consumer-latest.noarch.rpm from satellite01.bit63.net] ***
    changed: [ralsrv001]
    
    TASK [satellite_client : Register to Satellite 6 only for content] *************
    changed: [ralsrv001]
    
    TASK [satellite_client : Install the katello-agent] ****************************
    changed: [ralsrv001]
    
    TASK [satellite_client : Update the host to latest errata within the attached content view] ***
    changed: [ralsrv001]
    
    RUNNING HANDLER [satellite_client : Start katello-agent] ***********************
    ok: [ralsrv001]
    
    RUNNING HANDLER [satellite_client : Enable katello-agent] **********************
    ok: [ralsrv001]
    
    PLAY RECAP *********************************************************************
    ralsrv001                  : ok=8    changed=5    unreachable=0    failed=0

We see that the playbook completed successfully, and that the
bootstrap-related tasks were skipped. The new server is registered to
Satellite 6 as a content
host.

### Test 2 - Registering a RHEL 7.2 Server for both Content and Configuration Management

For this test we’ll provision another RHEL 7.2 VM (called "ralsrv002"),
also from the 'rhel72-generic' template. We’ll use the same provisioning
dialog settings as before.

Before we start the provisioning process however, we’ll edit the
*satellite\_6\_client* instance to add a value for param1. We’re going
to override the default 'hostgroup' extra variable, and pass the value
"Generic\_RHEL7-Servers" (see [figure\_title](#i18)).

![Content host in Satellite 6](images/ss18.png)

​  

If we follow the provisioning process in *automation.log* we once again
we see our assertion evaluate to **true**, and the Ansible job template
being called. On the Tower server we can follow the progress of the
    job:

    Identity added: /tmp/ansible_tower_NzAfZi/credential (/tmp/ansible_tower_NzAfZi/credential)
    Vault password:
    
    PLAY [all] *********************************************************************
    
    TASK [setup] *******************************************************************
    ok: [ralsrv002]
    
    TASK [satellite_client : Workaround for a non working DNS] *********************
    changed: [ralsrv002]
    
    TASK [satellite_client : Download bootstrap.py from satellite01.bit63.net] *****
    changed: [ralsrv002]
    
    TASK [satellite_client : Copy bootstrap.py script to /usr/local/sbin and make it executable] ***
    changed: [ralsrv002]
    
    TASK [satellite_client : Register to Satellite 6 with puppet enabled and add it to the correct hostgroup] ***
    changed: [ralsrv002]
    
    TASK [satellite_client : Install the katello-ca-consumer-latest.noarch.rpm from satellite01.bit63.net] ***
    skipping: [ralsrv002]
    
    TASK [satellite_client : Register to Satellite 6 only for content] *************
    skipping: [ralsrv002]
    
    TASK [satellite_client : Install the katello-agent] ****************************
    skipping: [ralsrv002]
    
    TASK [satellite_client : Update the host to latest errata within the attached content view] ***
    changed: [ralsrv002]
    
    PLAY RECAP *********************************************************************
    ralsrv002                  : ok=6    changed=5    unreachable=0    failed=0

This time we see that the *bootstrap.py* script was copied to the newly
provisioned server and used to register the host as both content host
and Puppet client.

In Satellite we can see the two new hosts added. We can verify that the
second host, ralsrv02 was added to the host group
"Generic\_RHEL7\_Servers", and has the
"KT\_Bit63\_Production\_RHEL\_7\_2\_Q4\_2016\_9" Puppet environment
assigned (see [figure\_title](#i19)).

![New hosts added to Satellite inventory](images/ss19.png)

​  

### Test 3 - Provisioning a CentOS 7.2 Server

To confirm the operation of our assertion in the *register\_satellite*
instance when provisioning a non-RHEL server, we’ll provision a CentOS
7.2 server from 'fat' template.

If we follow the provisioning progress in *automation.log* we see that
the assertion evaluates to **false**, and our Ansible job template is
not called.

    Evaluating substituted assertion ["centos" == "redhat"]
    Q-task_id([miq_provision_184]) Assertion Failed: <"centos" == "redhat">

## Summary

This chapter shows how we can integrate our virtual machine provisioning
workflow with our wider enterprise, in this case by registering new VMs
with a Satellite 6 server. It also illustrates how we can dynamically
enable or block states in our workflow depending on attributes that we
can test for using an assertion.
