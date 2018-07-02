# The Options Hash

A user starts the virtual machine provisioning workflow by clicking on
the **Lifecycle → Provision VMs** button in the **Virtual Machines**
toolbar of the WebUI. After selecting a template to provision from, the
requesting user completes the provisioning dialog and enters all of the
details required to create the virtual machine; the number of CPUs,
memory, network to connect to, and hard disk format for example. Somehow
this information collected from the WebUI must be added to the Automate
provisioning workflow.

Provisioning a virtual machine or instance is a complex operation that
as we have just seen, involves an approval stage. We saw in [Requests
and Tasks](../requests_and_tasks/chapter.asciidoc), that an automation
operation involving an approval stage is split into two parts, the
*Request* and the *Task*. In the case of a virtual machine provisioning
operation the request is represented by an *miq\_provision\_request*
object, and the task is represented by an *miq\_provision* object.

The inputs and options selected from the provisioning dialog are added
to the *miq\_provision\_request* object as key/value pairs in a data
structure known as the *options hash*. When we write our custom Ruby
methods to interact with the provisioning workflow, we frequently read
from and write to the options hash.

If the provisioning request is approved, the options hash from the
request object is propagated to the task object, but there are slight
differences between the two hashes. We’ll examine these next.

## Request Object (miq\_provision\_request)

The contents of the request object’s options hash varies slightly
between provisioning targets (VMware, OpenStack, RHEV etc) and target VM
Operating System (Linux, Windows etc.), but a typical hash for a Linux
virtual machine provision to a RHEV provider is:

``` ruby
request.options[:addr_mode] = ["static", "Static"]   (type: Array)
request.options[:cluster_filter] = [nil, nil]   (type: Array)
request.options[:cores_per_socket] = [1, "1"]   (type: Array)
request.options[:current_tab_key] = customize   (type: Symbol)
request.options[:customization_template_script] = nil
request.options[:customize_enabled] = ["disabled"]   (type: Array)
request.options[:delivered_on] = 2015-06-05 07:33:20 UTC   (type: Time)
request.options[:disk_format] = ["default", "Default"]   (type: Array)
request.options[:initial_pass] = true   (type: TrueClass)
request.options[:ip_addr] = nil
request.options[:linked_clone] = [nil, nil]   (type: Array)
request.options[:mac_address] = nil
request.options[:miqrequestdialog_name] = miq_provision_redhat_dialogs_template
request.options[:network_adapters] = [1, "1"]   (type: Array)
request.options[:number_of_sockets] = [1, "1"]   (type: Array)
request.options[:number_of_vms] = [1, "1"]   (type: Array)
request.options[:owner_email] = pemcg@bit63.com   (type: String)
request.options[:owner_first_name] = Peter   (type: String)
request.options[:owner_last_name] = McGowan   (type: String)
request.options[:pass] = 1   (type: Fixnum)
request.options[:placement_auto] = [false, 0]   (type: Array)
request.options[:placement_cluster_name] = [1000000000001, "Production"]
request.options[:placement_dc_name] = [1000000000002, "Default"]   (type: Array)
request.options[:placement_ds_name] = [1000000000001, "Data"]   (type: Array)
request.options[:placement_host_name] = [1000000000001, "rhevh12.bit63.net"]
request.options[:provision_type] = ["native_clone", "Native Clone"]
request.options[:retirement] = [0, "Indefinite"]   (type: Array)
request.options[:retirement_warn] = [604800, "1 Week"]   (type: Array)
request.options[:root_password] = nil
request.options[:schedule_time] = 2015-06-06 00:00:00 UTC   (type: Time)
request.options[:schedule_type] = ["immediately", "Immediately on Approval"]
request.options[:src_ems_id] = [1000000000001, "RHEV"]   (type: Array)
request.options[:src_vm_id] = [1000000000004, "rhel7-generic"]   (type: Array)
request.options[:start_date] = 6/6/2015   (type: String)
request.options[:start_hour] = 00   (type: String)
request.options[:start_min] = 00   (type: String)
request.options[:stateless] = [false, 0]   (type: Array)
request.options[:subnet_mask] = nil
request.options[:vlan] = ["public", "public"]   (type: Array)
request.options[:vm_auto_start] = [false, 0]   (type: Array)
request.options[:vm_description] = nil
request.options[:vm_memory] = ["2048", "2048"]   (type: Array)
request.options[:vm_name] = rhel7srv002   (type: String)
request.options[:vm_prefix] = nil
request.options[:vm_tags] = []   (type: Array)
```

## Correlation with the Provisioning Dialog

The key/value pairs that make up the options hash initially come from
the provisioning dialog (see [The Provisioning
Dialog](../the_provisioning_dialog/chapter.asciidoc)). For example if we
look at an extract from one of the provisioning dialog YAML files, we
see the definition for the *number\_of\_sockets* option:

    :number_of_sockets:
      :values:
        1: '1'
        2: '2'
        4: '4'
        8: '8'
      :description: Number of Sockets
      :required: false
      :display: :edit
      :default: 1
      :data_type: :integer

In the options hash this corresponds to:

``` ruby
request.options[:number_of_sockets]
```

We can also see that the options hash values for many of these keys are
two-element lists, for example:

``` ruby
request.options[:cores_per_socket] = [1, "1"]
```

This list corresponds to one of the **value: 'display name'** pairs
listed under the **:values:** subsection in the provisioning dialog YAML
file, like so:

    :cores_per_socket:
      :values:
        1: '1'
        ...

Some of the options hash value lists contain object IDs, for
example:

``` ruby
request.options[:placement_host_name] = [1000000000001, "rhevh12.bit63.net"]
```

These options hash keys are generally populated by a dynamic method. For
example the provisioning dialog YAML for this key name doesn’t contain a
static **:values:** list, instead it specifies that the values will be
dynamically generated by the `allowed_hosts` method, as follows:

    :placement_host_name:
      :values_from:
        :method: :allowed_hosts
      :auto_select_single: false
      :description: Name
      :required: false
      :display: :edit
      :data_type: :integer
      :required_description: Host Name

The `allowed_hosts` method filters the list of hosts presented to the
user based on previously selected values for cluster, resource pool and
folder.

## Accessing the Options Hash from an Automation Script

When we work with our own methods that interact with the VM provisioning
process, we often need to get and set values in the options hash.

### Reading Values

We can read any of the options hash values using the `get_option`
method, like so:

``` ruby
request = $evm.root['miq_provision_request']
memory_in_request = request.get_option(:vm_memory).to_i
```

For options hash keys whose values are lists, the `get_option` method
returns the first value in the list (there is a corresponding method
`get_option_last` that returns the last value in the list).

### Setting Values

We can also set most options using the `set_option` method, as follows:

``` ruby
request.set_option(:subnet_mask,'255.255.254.0')
```

When setting options hash keys whose values are normally lists, we
generally only need to write a scalar value using `set_option`. This can
be an integer or string, for example:

``` ruby
request.set_option(:number_of_sockets, '2')
```

or

``` ruby
request.set_option(:number_of_sockets, 2)
```

### Set Methods

Several options hash keys have their own `set` method, listed in the
following tables, which we should use in place of `set_option`.

| Options hash key | set method             | argument type |
| ---------------- | ---------------------- | ------------- |
| `:vm_notes`      | `request.set_vm_notes` | string        |

Generic options hash keys set
methods

| Options hash key                           | set method                           | argument type        |
| ------------------------------------------ | ------------------------------------ | -------------------- |
| `:vlan`                                    | `request.set_vlan`                   | string               |
| `:dvs`                                     | `request.set_dvs`                    | string               |
| `:addr_mode`                               | `request.set_network_address_mode`   | string               |
| `:placement_host_name`                     | `request.set_host`                   | service model object |
| `:placement_ds_name`                       | `request.set_storage`                | service model object |
| `:placement_folder_name`                   | `request.set_folder`                 | service model object |
| `:placement_cluster_name`                  | `request.set_cluster`                | service model object |
| `:placement_rp_name`                       | `request.set_resource_pool`          | service model object |
| `:pxe_server_id`                           | `request.set_pxe_server`             | service model object |
| `:pxe_image_id` (Linux server provision)   | `request.set_pxe_image`              | service model object |
| `:pxe_image_id` (Windows server provision) | `request.set_windows_image`          | service model object |
| `:customization_template_id`               | `request.set_customization_template` | service model object |
| `:iso_image_id`                            | `request.set_iso_image`              | service model object |

Infrastructure-specific options hash keys set
methods

| Options hash key               | set method                          | argument type        |
| ------------------------------ | ----------------------------------- | -------------------- |
| `:availability_zone`           | `request.set_availability_zone`     | service model object |
| `:instance_type`               | `request.set_instance_type`         | service model object |
| `:security_groups`             | `request.set_security_group`        | service model object |
| `:floating_ip_address`         | `request.set_floating_ip_address`   | service model object |
| `:cloud_network`               | `request.set_cloud_network`         | service model object |
| `:cloud_subnet`                | `request.set_cloud_subnet`          | service model object |
| `:guest_access_key_pair`       | `request.set_guest_access_key_pair` | service model object |
| `:cloud_tenant`                | `request.set_cloud_tenant`          | service model object |
| `:resource_group` (Azure only) | `request.set_resource_group`        | service model object |

Cloud-specific options hash keys set methods

The set methods that take a service model object as an argument, perform
a validity check that the value we’re setting is an eligible resource
for the provisioning instance. We use one of these methods in the
following way:

``` ruby
cloud_network = $evm.vmdb('CloudNetwork').find_by_name('private_3')
unless cloud_network.nil?
  prov.set_cloud_network(cloud_network)
  ...
```

> **Tip**
> 
> Use one of the techniques discussed in [Investigative
> Debugging](../investigative_debugging/chapter.asciidoc) to find out
> what key/value pairs are in the options hash to manipulate.

## Task Object (miq\_provision)

The options hash from the request object is propagated to each task
object, where it is subsequently extended by task-specific methods such
as those handling VM naming and placement:

``` ruby
miq_provision.options[:dest_cluster] = [1000000000001, "Default"]
miq_provision.options[:dest_host] = [1000000000001, "rhelh03.bit63.net"]
miq_provision.options[:dest_storage] = [1000000000001, "Data"]
miq_provision.options[:vm_target_hostname] = rhel7srv002
miq_provision.options[:vm_target_name] = rhel7srv002
```

Some options hash keys such as `:number_of_vms` have no effect if
changed in the task object; they are relevant only for the request.

### Adding Network Adapters

There are two additional methods that we can call on an `miq_provision`
object, to add further network adapters. These are `set_network_adapter`
and `set_nic_settings`.

``` ruby
idx = 1
miq_provision.set_network_adapter(idx,
                         {
                          :network => 'VM Network',
                          :devicetype => 'VirtualVmxnet3',
                          :is_dvs => false
                         })

miq_provision.set_nic_settings(idx,
                          {
                           :ip_addr => '10.2.1.23',
                           :subnet_mask => '255.255.255.0',
                           :addr_mode => ['static', 'Static']
                          })
```

## Adding Our Own Options: The ws\_values Hash

Sometimes we wish to add our own custom key/value pairs to the request
or task object, so that they can be used in a subsequent stage in the VM
provision state machine for custom processing. An example might be the
size and mount point for a secondary disk to be added as part of the
provisioning workflow. Although we could add our own key/value pairs
directly to the option hash, we risk overwriting a key defined in the
core provisioning code (or one added in a later release of ManageIQ).

There is an existing options hash key that is intended to be used for
this, called `ws_values`. The value of this key is itself a hash,
containing our key/value pairs that we wish to
save.

``` ruby
miq_provision.options[:ws_values] = {:disk_dize_gb=>100, :mountpoint=>"/opt"}
```

The `ws_values` hash is also used to store custom values that we might
supply if we provision a VM programmatically from either the RESTful
API, or from `create_provision_request`. One of the arguments for a
programmatic call to create a VM is a set of key/value pairs called
`additional_values` (it was originally called `additionalValues` in the
SOAP call). Any key/value pairs supplied with this argument for the
automation call will automatically be added to the `ws_options` hash.

By using the `ws_options` hash to store our own custom key/value pairs,
we make our code compatible with the VM provision request being called
programmatically.

## Summary

The options hashes in the *miq\_provision\_request* and *miq\_provision*
objects are some of the most important data structures that we work
with. They contain all of the information required to create the new
virtual machine or instance, and by setting their key values
programmatically we can influence the outcome of the provisioning
operation.

As discussed in [Requests and
Tasks](../requests_and_tasks/chapter.asciidoc), the challenge is
sometimes knowing whether we should access the options hash in the
*miq\_provision\_request* or *miq\_provision* objects, particularly when
setting values. We need to apply our knowledge of requests and tasks to
determine which context we’re working in.

We also need to be aware of which options hash keys have their own 'set'
method, as these keys typically require an array formatted in a
particular way.
