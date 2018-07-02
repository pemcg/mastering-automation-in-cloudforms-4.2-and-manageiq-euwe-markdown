# Automation Request Approval

In [Calling Automation Using the RESTful
API](../calling_automation_using_the_restful_api/chapter.asciidoc) we
looked at how external systems can make use of ManageIQ Automate
workflows by calling the RESTful API. In the examples we specified
`:auto_approve ⇒ true` in the REST call so that our requests were
immediately processed, however we can only auto-approve our own requests
if we authenticate as an admin user.

Embedding admin credentials in our external (calling) scripts is
generally considered unwise, but if we still want our automation
requests to be auto-approved, what can we do?

Fortunately by this stage in the book we have learned enough to be able
to implement our own approval workflow for automation requests. The
example in this chapter uses an access control group profile to control
which groups can submit auto-approved automation requests.

## Implementing a Custom Approval Workflow

Our automation request approval workflows will follow a very similar
pattern to those for provision request approval, and we’ll re-use and
adapt several of the components. We’ll implement two workflows; one
triggered from a **request\_created** event, and one from a
**request\_pending** event (see [figure\_title](#i10)).

![Event-triggered automation request approval
workflows](images/workflow.png)

​  

Before we implement anything we need to create some new Automate
datastore components to hold our workflow objects.

### Namespace

We’ll create a new namespace called *Automation* in our own domain.

### Group Profile

We’ll create a simple variant of the virtual machine provisioning group
profile (we can copy this from the ManageIQ domain and edit it). Our
profile class will contain two instances (profiles),
*Bit63Group\_vm\_user* and *.missing* (see [figure\_title](#i1)).

![Automation approval group profiles](images/ss1.png)

​  

The profile merely contains the name of the auto-approval state machine
instance that will be used to determine whether or not the request is
auto-approved. The profile is queried using the message
**get\_auto\_approval\_state\_machine\_instance**, and returns the
*Value* field via a *collect* as **/state\_machine\_instance**.

We’ll allow members of group *Bit63Group\_vm\_user* to have their
requests auto-approved, and everyone else (including admins who haven’t
specified `:auto_approve ⇒ true`) will require explicit approval.

The profile for the group *Bit63Group\_vm\_user* is shown in
[figure\_title](#i2).

![Profile schema for group Bit63Group\_vm\_user](images/ss3.png)

​  

The *.missing* profile for all other groups is shown in
[figure\_title](#i3).

![Profile schema for .missing](images/ss2.png)

​  

### State Machine

We’ll create a *StateMachines* namespace, and a simple variant of the VM
*ProvisionRequestApproval* class. We’ll copy the
*ProvisionRequestApproval* class from the ManageIQ domain into ours
under the new *StateMachines* namespace, and call it
*AutomationRequestApproval*. We’ll copy the associated instances and
methods as well (see [figure\_title](#i4)).

![AutomationRequestApproval instances and methods](images/ss4.png)

​  

#### Instances

The *RequireApproval* instance has an **approval\_type** value of
*require\_approval* (see [figure\_title](#i5)).

![Fields of the RequireApproval instance](images/ss5.png)

​  

The *Auto* instance is similar, but has an **approval\_type** value of
*auto*.

#### Methods

The *validate\_request* method is as follows:

``` ruby
request = $evm.root['miq_request']
resource = request.resource
raise "Automation Request not found" if request.nil? || resource.nil?

$evm.log("info", "Checking for auto_approval")
approval_type = $evm.object['approval_type'].downcase
if approval_type == 'auto'
  $evm.root["miq_request"].approve("admin", "Auto-Approved")
  $evm.root['ae_result'] = 'ok'
else
  msg =  "Request was not auto-approved"
  resource.set_message(msg)
  $evm.root['ae_result'] = 'error'
  $evm.object['reason'] = msg
end
```

The *pending\_request* method is as follows:

``` ruby
#
# This method is executed when the automation request is NOT auto-approved
#
# Get objects
msg = $evm.object['reason']
$evm.log('info', "#{msg}")

# Raise automation event: request_pending
$evm.root["miq_request"].pending
```

The method definition is also given an input parameter with Input Name
**reason** and Data Type **string**

The approve\_request method is as follows:

``` ruby
#
# This method is executed when the automation request is auto-approved
#
# Auto-Approve request
$evm.log("info", "AUTO-APPROVING automation request")
$evm.root["miq_request"].approve("admin", "Auto-Approved")
```

### Email Classes

We create an *Email* class, with an *AutomationRequest\_Pending*
instance and method (see [figure\_title](#i6)).

![Email classes and methods](images/ss6.png)

​  

The method code is copied and adapted as appropriate from the VM
*ProvisionRequest\_Pending* method. We specify as the
**to\_email\_address** a user that will act as approver for the
automation requests.

The full code for the methods is
[here](https://github.com/pemcg/mastering-automation-in-cloudforms-4.2-and-manageiq-euwe/tree/master/automation_request_approval/scripts)

## Policies

We need to generate policy instances for two AutomationRequest events,
**AutomationRequest\_created** and **AutomationRequest\_approved**. We
copy the standard */System/Policy* class to our domain, and add two
instances (see [figure\_title](#i7)).

![New policy instances](images/ss7.png)

​  

### AutomationRequest\_created

Our policy instance for *AutomationRequest\_created* has three entries;
an assertion and two relationships. We need to recognise whether an
automation request was made with the `:auto_approve ⇒ true` parameter.
If it was, we need to skip our own approval workflow.

We know (from some investigative debugging using *ObjectWalker*) that
when a request is made that specifies `:auto_approve ⇒ true`, we have an
`$evm.root['automation_request'].approval_state` attribute with a value
of **approved**. When a request is made that specifies `:auto_approve ⇒
false` this value is **pending\_approval**. We can therefore create our
assertion to look for `$evm.root['automation_request'].approval_state ==
'pending_approval'`, and continue with the instance only if the boolean
test returns **true**.

The **rel1** relationship of this instance performs a profile lookup
based on our user group, to find the auto-approval state machine
instance that should be run. The **rel2** relationship calls this state
machine instance (see [figure\_title](#i8)).

![Fields of the AutomationRequest\_created instance](images/ss8.png)

​  

### AutomationRequest\_pending

The *AutomationRequest\_pending* instance contains a single relationship
to our *AutomationRequest\_pending* email instance (see
[figure\_title](#i9)).

![Fields of the AutomationRequest\_pending instance](images/ss9.png)

​  

## Testing

We’ll submit three automation requests via the RESTful API, calling a
simple *Test* instance. The calls will be made as follows:

  - As user *admin*, specifying `:auto_approve ⇒ true`

  - As user *admin*, specifying `:auto_approve ⇒ false`

  - As a user who is a member of the group *Bit63Group\_vm\_user*

For the first call, our assertion correctly prevents our custom approval
workflow from running (the request has already been auto-approved). From
*automation.log* we see:

    Evaluating substituted assertion ["approved" == "pending_approval"]
    Assertion Failed: <"approved" == "pending_approval">
    Followed  Relationship [miqaedb:/System/Policy/AutomationRequest_created#create]
    Followed  Relationship [miqaedb:/System/Policy/request_created#create]
    Followed  Relationship [miqaedb:/System/Event/request_created#create]

For the second call we see that the assertion evaulates to **true**, but
the user *admin*'s group (*EVMGroup-super\_administrator*) doesn’t have
a group profile. The .missing profile is used, and the automation
request is not auto-approved.

The *admin* user receives an email:

    Request was not auto-approved.
    
    Please review your Request and update or wait for approval from an Administrator.
    
    To view this Request go to: https://192.168.1.45/miq_request/show/125
    
    Thank you,
    Virtualization Infrastructure Team

The *approving* user also receives an email:

    Approver,
    An automation request received from admin@bit63.com is pending.
    
    Request was not auto-approved.
    
    For more information you can go to: https://192.168.1.45/miq_request/show/125
    
    Thank you,
    Virtualization Infrastructure Team

Clicking the link takes us to an approval page, and we can approve the
request, which then continues.

For the third call we see that the assertion evaluates to **true**, but
this time we see the valid group profile being
    used:

    Evaluating substituted assertion ["pending_approval" == "pending_approval"]
    Following Relationship [miqaedb:/Automation/Profile/Bit63Group_vm_user#get_auto..

This group’s profile auto-approves the automation request, and the
*Test* instance is successfully run:

    Q-task_id([automation_task_186]) \
                              <AEMethod test> Calling the test method was successful!

Success\!

## Summary

In this chapter we’ve assembled many of the Automate components that
we’ve studied throughout the book to create our own custom approval
workflow. We’ve done it by copying and adapting slightly several
existing components in the ManageIQ domain, and adding our own pieces
where necessary.

We started off by creating our own namespace to work in, and we added an
access control group profile so that we can apply the auto-approval to
specific groups. We cloned the *ProvisionRequestApproval* class and its
methods to become our *AutomationRequestApproval* state machine, and we
created two instances, one called *Auto*, and one called
*RequireApproval*. We added an *Email* class and cloned and adapted the
*ProvisionRequest\_Pending* instance and method to become our
*AutomationRequest\_Pending* versions. Finally we added two policy
instances to handle the two Automation **request\_created** and
**request\_pending** events.

Creating an approval workflow such as this is really just a case of
putting the pieces in place and wiring it together. We know that
approval workflows start with an event, and that the event is translated
to a policy. As long as our policy instances route the workflow into the
appropriate handlers (generally a state machine or email class), all
that is left is to adapt the method code to our specific purposes, and
test.
