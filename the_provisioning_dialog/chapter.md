# The Provisioning Dialog

So far in Part II we have looked at several ways in which the virtual
machine provisioning *process* can be customised. We have seen how we
can automate the selection of virtual machine name, decide where to
place the virtual machine, and expand the state machine to insert our
own provisioning workflow steps.

This chapter will look at how the initial dialog that launched the
provisioning process can also be customised. We might want to do this to
expand the options available to us, or to preconfigure and hide other
dialog elements for certain groups of users.

The specification for the new virtual machine or instance is entered
into the *Provisioning Dialog* that is displayed to the user in the
WebUI. This dialog prompts for all of the parameters and characteristics
that will make up the new VM, such as the name, number of CPUs, and IP
address.

## Tabs and Input Fields

The provisioning dialog contains a number of tabs (**Request**,
**Purpose**, **Catalog**, **Environment**, etc.), and a number of input
fields per tab (see [figure\_title](#i1)).

![The hardware tab of the VM provisioning dialog](images/ss1.png)

​  

The provisioning dialog is context-sensitive, so a different set of
input field options will be displayed when provisioning into VMware or
OpenStack, for example.

## Dialog YAML

Each provisioning dialog is formatted from a large (900 line+) YAML
file, specifying the main tabs, dialogs and fields to be displayed, for
example:

    ---
    :buttons:
    - :submit
    - :cancel
    :dialogs:
      :requester:
        :description: Request
        :fields:
          :owner_phone:
            :description: Phone
            :required: false
            :display: :hide
            :data_type: :string
    ...
          :owner_email:
            :description: E-Mail
            :required: true
            :display: :edit
            :data_type: :string
     ...
       :purpose:
        :description: Purpose
        :fields:
          :vm_tags:
            :required_method: :validate_tags
            :description: Tags
            :required: false
            :options:
              :include: []
    ...
            :display: :edit
            :required_tags: []
            :data_type: :integer
        :display: :hide
        :field_order:
     ...
     :dialog_order:
    - :requester
    - :purpose

Dialog tabs and fields have four useful attributes that can be set:

  - Hidden (`:display: :hide`)

  - Visible (`:display: :show`)

  - Editable (`:display: :edit`)

  - Mandatory (`:required: true`)

## Selection of VM Provisioning Dialog

Several VM provisioning dialogs are supplied out-of-the-box with
CloudForms and ManageIQ, each of which provides the context sensitivity
for the particular provisioning operation. They are found under the
**Automate → Customization** menu, in the **Provisioning Dialogs**
accordion.

| Name                                                 | Description                                   |
| ---------------------------------------------------- | --------------------------------------------- |
| miq\_provision\_amazon\_dialogs\_template            | Sample Amazon Instance Provisioning Dialog    |
| miq\_provision\_azure\_dialogs\_template             | Sample Azure Instance Provisioning Dialog     |
| miq\_provision\_google\_dialogs\_template            | Sample Google Instance Provisioning Dialog    |
| miq\_provision\_microsoft\_dialogs\_template         | Sample Microsoft VM Provisioning Dialog       |
| miq\_provision\_openstack\_dialogs\_template         | Sample Openstack Instance Provisioning Dialog |
| miq\_provision\_redhat\_dialogs\_clone\_to\_vm       | Sample RedHat VM Clone to VM Dialog           |
| miq\_provision\_redhat\_dialogs\_template            | Sample RedHat VM Provisioning Dialog          |
| miq\_provision\_vmware\_dialogs\_clone\_to\_template | Sample VM Clone to Template Dialog            |
| miq\_provision\_vmware\_dialogs\_clone\_to\_vm       | Sample VM Clone to VM Dialog                  |
| miq\_provision\_dialogs\_pre\_sample                 | Sample VM Pre-Provisioning Dialog             |
| miq\_provision\_dialogs                              | Sample VM Provisioning Dialog                 |
| miq\_provision\_vmware\_dialogs\_template            | Sample VM Provisioning Dialog (Template)      |

The various dialogs contain values that are relevant to their target
provider type (Amazon, Azure, Google, OpenStack, Microsoft, VMware or
RedHat), and also to the operation type (clone from template, clone to
template, or clone to vm).

The selection of VM provisioning dialog to display to a user depends on
the **dialog\_name** attribute in the provisioning group profile. The
default **dialog\_name** value for the *.missing* and
*EvmGroup-super\_administrator* profiles is:

    ${#dialog_name_prefix}_${/#dialog_input_request_type}

The two variables are substituted at runtime, and provide the context
sensitivity. The **dialog\_name\_prefix** value is determined by the
*vm\_dialog\_name\_prefix* method, which contains the following code:

``` ruby
platform  = $evm.root['platform']
$evm.log("info", "Detected Platform:<#{platform}>")

if platform.nil?
  source_id = $evm.root['dialog_input_src_vm_id']
  source    = $evm.vmdb('vm_or_template', source_id) unless source_id.nil?
  if source
    platform = source.model_suffix.downcase
  else
    platform = "vmware"
  end
end

dialog_name_prefix = "miq_provision_#{platform}_dialogs"
$evm.object['dialog_name_prefix'] = dialog_name_prefix
```

The **dialog\_input\_request\_type** value is translated by a Rails
class *MiqRequestWorkflow* to be the instance name of the VM
provisioning state machine that we are using - that is, *template*,
*clone\_to\_vm* or *clone\_to\_template*.

So for a VM provision request from template into an RHEV provider, the
**dialog\_name** value will be substituted as follows:

    miq_provision_redhat_dialogs_template

## Group-Specific Dialogs

We can set separate provisioning dialogs for individual groups if we
wish. As an example the VMware-specific *miq\_provision\_dialogs-user*
dialog presents a reduced set of tabs, dialogs and input fields. The
hidden tabs have been given default values, and *Automatic Placement*
has been set to `true`:

``` 
      :placement_auto:
        :values:
          false: 0
          true: 1
        :description: Choose Automatically
        :required: false
        :display: :edit
        :default: true
        :data_type: :boolean
```

We can create per-group dialogs as we wish, customising the values that
are hidden or set as default.

### Example - Expanding the Dialog

In some cases it’s useful to be able to expand the range of options
presented by the dialog. For example the standard dialogs only allow us
to specify VM memory in units of 1GB, 2GB or 4GB (see
[figure\_title](#i2)).

![Default memory size options](images/ss2.png)

​  

> **Note**
> 
> CloudForms 4.2/ManageIQ *Euwe* has increased the default memory sizes
> available in this drop-down

These options come from the `:vm_memory` dialog section:

``` 
      :vm_memory:
        :values:
          '2048': '2048'
          '4096': '4096'
          '1024': '1024'
        :description: Memory (MB)
        :required: false
        :display: :edit
        :default: '1024'
        :data_type: :string
```

We sometimes need to be able to provision larger VMs, but fortunately we
can customise the dialog to our own needs.

#### Copy the existing dialog

If we identify the dialog that is being used (in this example case it is
*miq\_provision\_redhat\_dialogs\_template* as we’re provisioning into
RHEV using native clone), we can copy the dialog to make it editable
(we’ll call the new version
*bit63\_miq\_provision\_redhat\_dialogs\_template*).

We can then expand the `:vm_memory` section to match our requirements:

``` 
      :vm_memory:
        :values:
          '1024': '1024'
          '2048': '2048'
          '4096': '4096'
          '8192': '8192'
          '16384': '16384'
        :description: Memory (MB)
        :required: false
        :display: :edit
        :default: '1024'
        :data_type: :string
```

#### Create a group profile

Now we copy the */Infrastructure/VM/Provisioning/Profile* class into our
own domain, and create a profile instance for the group that we wish to
assign the new dialog to, in this case **Bit63Group-user** (see
[figure\_title](#i3)).

![Creating a new profile instance](images/ss3.png)

​  

The **dialog\_name** field in the new profile should contain the name of
our new dialog (see [figure\_title](#i4)).

![The dialog\_name schema field value changed to the new profile
name](images/ss4.png)

​  

#### Testing the provisioning dialog

To test this we login as a user who is a member of the
**Bit63Group-user** group, and provision a virtual machine. If we
navigate to the **Hardware** tab of the provisioning dialog we should
see the expanded range of memory options (see [figure\_title](#i5)).

![Expanded range of memory sizes](images/ss5.png)

​  

## Summary

In this chapter we’ve seen how the virtual machine provisioning dialog
is used, and how it can be customised.

We often create group-specific dialogs that contain a default set of
provisioning options, and we can take advantage of this when we make an
API call to provision a virtual machine as a particular user for
example. The user’s group profile will provide default values for the
virtual machine, so we need only specify override values in our API call
parameters.
