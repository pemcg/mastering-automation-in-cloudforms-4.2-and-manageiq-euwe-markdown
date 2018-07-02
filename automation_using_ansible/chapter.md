# Automation Using Ansible

CloudForms 4.1/ManageIQ *Darga* introduced the possibility of performing
automation tasks using Ansible playbooks in addition to or instead of
using native Ruby. This automation requires the presence of an Ansible
Tower server, added to CloudForms or ManageIQ as a configuration
management provider.

One of the most powerful features of Ansible as a configuration
management tool is the relative simplicity and ease of understanding of
its YAML-formatted playbooks. Adding the capability to be able to run
Ansible playbooks from Automate as well as run Ruby methods presents us
with a dilemma; when to use which?

Ruby methods allow us to access and manipulate all of the objects and
their properties within the VMDB. We have a powerful scripting language
at our disposal that allows us to make real-time decisions as part of
our automation workflow. For example we can determine how many VMs are
currently running on each hypervisor in a cluster before making
placement decisions during provisioning. The disadvantage is that we
need to be fairly comfortable with the Ruby scripting language and the
Automate object model to take full advantage and start developing our
own automation scripts.

Ansible playbooks allow us to harness the power of the Ansible module
library to interact with many systems or infrastructure components in
our enterprise that may not be natively supported by CloudForms or
ManageIQ (such as load balancers for example). They allow us to do this
using an easy to learn and well-documented YAML-based modelling
language. We are also able to take advantage of the many thousands of
existing Ansible roles that are downloadable from the Ansible Galaxy web
site \[1\]

Whether we use Ruby methods or Ansible playbooks, we can still take
advantage of some of the powerful Automate features such as state
machines to build our workflows.

## Terminology

When we use Ansible to perform our automation tasks, if helps to be
familiar of some of the Ansible Tower terminology.

### Playbooks

Playbooks are Ansible’s configuration, deployment, and orchestration
modelling language. Each playbook is composed of one or more *plays* in
a list, and a play maps a group of hosts to one or more well defined
roles or *tasks*.

Playbooks are designed to be human-readable and are expressed in YAML
format. An example snippet from a simple playbook is as follows:

{% raw %} --- hosts: webservers vars: web\_pkg: httpd firewall\_pkg:
firewalld firewall\_service: firewalld tasks: - name: Install the
required packages yum: name: - "{{ web\_pkg }}" - "{{ firewall\_pkg }}"
state: latest - name: Start and enable the {{ firewall\_service }}
service service: name: "{{ firewall\_service }}" enabled: true state:
started {% endraw %}

More advanced playbooks separate out various sections (such as variable
or handler definitions) into separate files that are included into a
main playbook file. This logical separation eases maintenance and
promotes reuse.

### Roles

Roles provide Ansible with a way to load tasks, handlers, and variables
from external files, based on a known file structure. Assigning a role
to a group of hosts (or a set of groups, or host patterns, etc.) implies
that they should implement a specific behaviour, i.e. adopt that role.
Grouping content by roles also allows easy sharing of roles with other
users.

A typical role directory structure might be as follows:

    roles/
        common/               # this hierarchy represents a "role"
            tasks/            #
                main.yml      #  <-- tasks file can include smaller files if warranted
            handlers/         #
                main.yml      #  <-- handlers file
            templates/        #  <-- files for use with the template resource
                ntp.conf.j2   #  <------- templates end in .j2
            files/            #
                bar.txt       #  <-- files for use with the copy resource
                foo.sh        #  <-- script files for use with the script resource
            vars/             #
                main.yml      #  <-- variables associated with this role
            defaults/         #
                main.yml      #  <-- default lower priority variables for this role
            meta/             #
                main.yml      #  <-- role dependencies

### Projects

A Project is a logical collection of Ansible playbooks (usually in the
form of roles), represented in Ansible Tower. Projects are often stored
in a source code management (SCM) system such as Git, and Ansible Tower
allows for the importing of projects directly from an SCM, and for the
project code to be refreshed immediately before running if required.
Projects are stored under the */var/lib/awx/projects* directory on the
Ansible Tower server.

An example project directory structure might be as follows:

    project_name
       site.yml
       webservers.yml
       dbservers.yml
       roles/
          common/
             files/
             templates/
             tasks/
             handlers/
             vars/
             defaults/
             meta/
          webservers/
             files/
             templates/
             tasks/
             handlers/
             vars/
             defaults/
             meta/
          ...

### Job Templates

A job template is a definition and set of parameters for running an
Ansible Tower job. Job templates allow us to execute the same job many
times, by pre-defining such items as the playbook to run, extra
variables to pass, inventory to be managed, and credentials that should
be used. A typical job template definition in Ansible Tower is shown in
[figure\_title](#i1).

![Typical job template in Ansible Tower](images/ss1.png)

​  

Job templates are significant when we discuss CloudForms/ManageIQ
integration with Ansible Tower, because a job template is the entity
that we run from Automate. [figure\_title](#i2) shows the list of
Ansible Tower job templates displayed in the CloudForms WebUI.

![Ansible job templates visible in CloudForms](images/ss3.png)

​  

#### Extra Variables

Ansible playbook variables can be defined in a number of places, but
there is an established precedence to determine which value is used when
the playbook is run. If a variable of the same name is defined in
multiple places, the occurrence defined with the highest precedence will
be used (See [table\_title](#table27a.1) for the precedence list \[2\]).

| Precedence         | where defined                        |
| ------------------ | ------------------------------------ |
| lowest precedence  | role defaults                        |
| \-                 | inventory vars                       |
| —                  | inventory group\_vars                |
| \---               | inventory host\_vars                 |
| \----              | playbook group\_vars                 |
| \-----             | playbook host\_vars                  |
| \------            | host facts                           |
| \-------           | play vars                            |
| \--------          | play vars\_prompt                    |
| \---------         | play vars\_files                     |
| \----------        | registered vars                      |
| \-----------       | set\_facts                           |
| \------------      | role and include vars                |
| \-------------     | block vars (only for tasks in block) |
| \--------------    | task vars (only for the task)        |
| highest precedence | extra vars                           |

Ansible variable precedence

We see that extra variables have the highest precedence, and we can
define defaults for extra variables in the job template. If the **Prompt
on launch** option is checked then we can also override these default
values from CloudForms/ManageIQ when we launch the job template. The
precedence ensures that our dynamically defined variables are the ones
that are used by the playbook.

### Jobs

A job is an instance of Ansible Tower launching a playbook against an
inventory of hosts.

### Inventories

An inventory defines a list of managed hosts that Ansible jobs can be
run against. Inventories can contain *groups* which further sub-divide
hosts into logical collections of systems. Groups and their contents can
be dynamically generated using an Ansible Tower inventory script (see
[figure\_title](#i3)).

![Definition of an "All Servers" inventory group in Ansible
Tower](images/ss2.png)

​  

We can define several different inventories, and use them in our various
job template definitions.

#### Update on Launch

The **Update on launch** update option is particularly important when we
define dynamic inventory groups to be referenced from CloudForms or
ManageIQ Automate. We often wish to call Ansible Tower jobs as part of
our provisioning workflow, and so we need an up-to-date inventory that
contains our newly provisioned virtual machine. The **Update on launch**
setting ensures that the inventory defined in the job template is always
refreshed immediately before the job is run.

#### The Limit Variable

Many Ansible job templates contain playbooks that have a `hosts` key
defined as `all`. When we execute a job from CloudForms or ManageIQ, we
usually wish to override this and the run the job on one or more
specific systems, and the built-in `limit` variable enables us to to
this.

The `limit` variable is automatically defined for us by Automate and
passed to Ansible Tower with a new job request if either of the
following two Automate attributes contain valid non-nil values:

``` ruby
$evm.root['vm'].name
```

or

``` ruby
$evm.root['miq_provision'].destination.name
```

These values will be set if we are calling an Ansible Tower job template
either from a button on a VM object, or as part of a VM provisioning
workflow (after the virtual machine has been created). For these two
common use-cases we don’t have to worry about defining the limit
ourselves.

## Adding the ansible-remote User with a cloud-init Script

As Ansible uses ssh to connect to managed servers and run playbooks, we
must ensure that our newly provisioned virtual machines are configured
with the ssh credentials required to perform the actions. It is
generally considered good practice not to connect at the root user, so
the examples described in this book use an account called
'ansible-remote'.

If we are provisioning from 'fat' template we can create the
ansible-remote user using a CloudForms/ManageIQ CloudInit-type
customization template, called from the **Customize** tab of the
provisioning dialog.

An example cloud-init script to setup the newly provisioning virtual
machine as an Ansible Tower managed host is as follows:

    #cloud-config
    
    ssh_pwauth: true
    disable_root: false
    
    users:
      - default
      - name: ansible-remote
        shell: /bin/bash
        sudo: ['ALL=(ALL) NOPASSWD:ALL']
        ssh_authorized_keys:
          - ssh-rsa AAAAB3N...bit63.net
    
    chpasswd:
      list: |
        root:<%= MiqPassword.decrypt(evm[:root_password]) %>
      expire: false
    
    preserve_hostname: false
    manage_etc_hosts: true
    fqdn: <%= evm[:hostname] %>

We create an Ansible Tower machine credential containing the private key
that matches this public key, and we can specify this machine credential
when we define our job templates.

> **Note**
> 
> We should also ensure that our virtual machine templates are prepared
> with the cloud-init package. For Red Hat Enterprise Linux this is
> installed from the **rhel-7-server-rh-common-rpms** repository.

## Summary

This chapter has introduced some of the concepts and terminology that we
encounter when we use the powerful capabilities of Ansible Tower. In the
next chapter we’ll take a look at the new features of Automate that
allow us to create Ansible Tower jobs as part of our automation
workflows.

### Further Reading

[Ansible Documentation](https://docs.ansible.com)

1.  <https://galaxy.ansible.com>

2.  See
    <http://docs.ansible.com/ansible/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable>
