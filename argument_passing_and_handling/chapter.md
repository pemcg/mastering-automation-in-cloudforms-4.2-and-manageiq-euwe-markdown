# Argument Passing and Handling

Over the preceding chapters we have discovered several ways of calling
Automate instances. In some cases we need to pass arguments into the
instance’s method, but the way that we pass arguments into methods, and
receive them from inside the method varies depending on how the instance
is called. We need to consider this if we’re writing code that can be
called in several ways, such as from a button and/or from an API call.

In this chapter we’ll look at how we pass arguments into instances, and
how we retrieve them from inside the method. We will call the same
instance (*object\_walker*) four ways, passing two arguments each time,
'lunch' and 'dinner'. We can use object\_walker\_reader to show us where
the arguments can be read from inside our called method.

> **Note**
> 
> If using version 1.8 or later of object\_walker, we need to set
> `$print_evm_object = true` in the script so that it prints the
> attributes of $evm.object

## Case 1 - Calling from a Button

For this first case we call *object\_walker* (via
*/System/Process/Request/Call\_Instance*) from a button. We create a
button dialog that prompts for two text box fields (see
[figure\_title](#i1)).

![Simple dialog to prompt for input values](images/ss1.png)

​  

We then add the button to a button group anywhere.

If we click on the button, and enter the values 'salad' and 'pasta' into
the dialog boxes, we see the dialog values appear in `$evm.root` in the
receiving method, indexed by the key name prefixed by *dialog\_*

    ~/object_walker_reader.rb | grep -P "lunch|dinner"
         |    $evm.root['dialog_dinner'] = salad   (type: String)
         |    $evm.root['dialog_lunch'] = pasta   (type: String)

## Case 2 - Calling from the RESTful API

For this use-case we have an external Ruby script that calls our
internal ManageIQ instance via the REST API:

``` ruby
url = "https://#{server}"

post_params = {
  :version => '1.1',
  :uri_parts => {
    :namespace => 'Discovery',
    :class     => 'ObjectWalker',
    :instance  => 'object_walker'
  },
  :parameters => {
    :lunch  => "sandwich",
    :dinner => "steak"
  },
  :requester => {
    :auto_approve => true
  }
}.to_json
query = "/api/automation_requests"

rest_return = RestClient::Request.execute(method: :post, url: url + query,
                                          :user => username,
                                          :password => password,
                                          :headers => {:accept => :json},
                                          :payload => post_params,
                                          verify_ssl: false)
result = JSON.parse(rest_return)
```

In the called method we see the arguments visible in several places; in
the task’s options hash as the `attrs` key; under `$evm.root` because
this is the instance that we launched when entering Automate, and under
`$evm.object` because this is also our current object.

    ~/object_walker_reader.rb | grep -P "lunch|dinner"
        |    object_walker:  $evm.root['automation_task'].options[:attrs] = {:lunch=>"sandwich", :dinner=>"steak", :userid=>"admin"}  (type: Hash)
        object_walker:  $evm.root['dinner'] = steak  (type: String)
        object_walker:  $evm.root['lunch'] = sandwich  (type: String)
        object_walker:  $evm.object['dinner'] = steak  (type: String)
        object_walker:  $evm.object['lunch'] = sandwich  (type: String)

## Case 3 - Calling from a Relationship or Automate Datastore URI

When we call instances via a relationship (such as from a state
machine), we specify the full URI of the instance. We can append
arguments to this URI using standard web form query string syntax.

For this use case we’ll call *object\_walker* from an already running
automation script using `$evm.instantiate`. The argument to
`$evm.instantiate` is the full URI of the instance to be launched, as
follows:

``` ruby
$evm.instantiate("/Discovery/ObjectWalker/object_walker?lunch=salad&dinner=spaghetti")
```

When instantiated in this way, the receiving method retrieves the
arguments from `$evm.object` (one of our (grand)parent instances is
`$evm.root`, our immediate caller is `$evm.parent`).

    ~/object_walker_reader.rb | grep -P "lunch|dinner"
         object_walker:   $evm.object['dinner'] = spaghetti   (type: String)
         object_walker:   $evm.object['lunch'] = salad   (type: String)

## Case 4 - Passing Arguments via the ws\_values Hash During a VM Provision

We can pass our own custom values into the virtual machine provisioning
process so that they can be interpreted by any method in the *Provision
VM from Template* state machine.

The facility to do this is provided by the **additional\_values** field
in an **/api/provision\_requests** REST call (**additionalValues** in
the original SOAP `EVMProvisionRequestEx` call), or from the sixth
element in the argument list to an
`$evm.execute('create_provision_request',…​)` call (see [Creating
Programming Requests
Programmatically](../creating_provisioning_requests_programmatically/chapter.asciidoc)).

For this use case we’ve edited the *Provision VM from Template* state
machine to add a few extra stages (see [figure\_title](#i3)).

![Calling object\_walker from the VmProvision\_VM state
machine](images/ss3.png)

​  

These stages could modify the provisioning process if required based on
the custom values passed in. An example of this might be to specify the
disk size for an additional disk to be added by the AddDisk stage.

For this example we’re using a simple automation method to call
`$evm.execute('create_provision_request',…​)` to provision a new virtual
machine. We specify the custom values in **arg6**:

``` ruby
# arg1 = version
args = ['1.1']

# arg2 = templateFields
args << {'name'         => 'rhel7-generic',
         'request_type' => 'template'}

# arg3 = vmFields
args << {'vm_name' => 'test10',
         'vlan'    => 'rhevm'}

# arg4 = requester
args << {'owner_email'      => 'pemcg@bit63.com',
         'owner_first_name' => 'Peter',
         'owner_last_name'  => 'McGowan'}

# arg5 = tags
args << nil

# arg6 = Web Service Values (ws_values)
args << {'lunch'  => 'soup',
         'dinner' => 'chicken'}

# arg7 = emsCustomAttributes
args << nil

# arg8 = miqCustomAttributes
args << nil

request_id = $evm.execute('create_provision_request', *args)
```

When we call this method and the virtual machine provisioning process
begins, we can retrieve the custom values at any stage from the
`miq_provision_request` or `miq_provision` options hash using the
`ws_values` key…​

    ~/object_walker_reader.rb | grep -P "lunch|dinner"
         |    $evm.root['miq_provision'].options[:ws_values] = \
                                {:lunch=>"soup", :dinner=>"chicken"}   (type: Hash)
         |    |    miq_provision_request.options[:ws_values] = \
                                {:lunch=>"soup", :dinner=>"chicken"}   (type: Hash)

## Passing Arguments When Calling a Method in the Same Class

When an instance (such as a state machine) calls a method in the same
class as itself, it can pass key/value argument pairs in parentheses as
input parameters with the call. We see the *VMProvision\_VM* state
machine do this when it calls *update\_provision\_status* during the
processing of the **On Entry**, **On Exit** and **On Error** (see
[figure\_title](#i4)).

![Text Arguments Passed to update\_provision\_status](images/ss4.png)

​  

When we create a method that accepts input parameters in this way, we
need to specify the name and data type of each parameter in the method
definition (see [figure\_title](#i5)).

![Specifying Input Parameters](images/ss5.png)

​  

The method then reads the parameters from `$evm.inputs`:

``` ruby
update_provision_status(status => 'pre1',status_state => 'on_entry')

 # Get status from input field status
 status = $evm.inputs['status']

 # Get status_state ['on_entry', 'on_exit', 'on_error'] from input field
 status_state = $evm.inputs['status_state']
```

## Summary

This chapter shows how we can send arguments when we call instances, and
how we process them inside the method. The way that a method retrieves
an argument depends on how the instance has been called, but we can use
`$evm.root['vmdb_object_type']` as before to determine this, and access
the argument in a appropriate manner.
