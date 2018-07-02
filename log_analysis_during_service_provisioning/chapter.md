# Log Analysis During Service Provisioning

The workflow of provisioning a virtual machine from a service catalog
involves a request, an approval stage, several tasks, and multiple
concurrently running state machines.

If we are curious to discover more about their interaction, we can
follow this workflow by examining the log lines written to
*automation.log* during the service provisioning operation. This can
reveal some interesting details about the interleaving of the various
operations and state machines.

For this example we’ve grepped for the "Following..Followed" message
pairs in *automation.log*. The service provisioning request was from a
non-admin user in a group *Bit63\_user*, so we see some group-specific
profile processing. For the purposes of brevity
*service\_template\_provision* is abbreviated to *stp* in theFollowing
outputs.

## Initial Request and Profile Lookup

We see the initial service request being created, and a *service*
provisioning profile lookup to get the auto-approval state machine

    Following [/System/Policy/request_created#create]
      Following [/System/Process/parse_provider_category#create]
      Followed [/System/Process/parse_provider_category#create]
      Following [/System/Policy/ServiceTemplateProvisionRequest_created#create]
        Following [/service/Provisioning/Profile/Bit63_user#get_auto_approval_state_machine]
        Followed [/service/Provisioning/Profile/Bit63_user#get_auto_approval_state_machine]
        Following [/service/Provisioning/StateMachines/ServiceProvisionRequestApproval/Default#create]
        Followed [/service/Provisioning/StateMachines/ServiceProvisionRequestApproval/Default#create]
      Followed [/System/Policy/ServiceTemplateProvisionRequest_created#create]
    Followed [/System/Policy/request_created#create]
    Followed [/System/Event/RequestEvent/Request/request_created#create]

## Request Processing and Approval

We see the request\_approved workflow, and the request starting. Some
operations performed in the context of the service template provisioning
*request* (service\_template\_provision\_request\_59), including the
quota checking
    workflow:

    Following [/System/Event/RequestEvent/Request/request_approved#create]
      Following [/System/Policy/request_approved#create]
        Following [/System/Process/parse_provider_category#create]
        Followed [/System/Process/parse_provider_category#create]
        Following [/System/Policy/ServiceTemplateProvisionRequest_Approved#create]
          Following [/Service/Provisioning/Email/ServiceTemplateProvisionRequest_Approved#create]
          Followed [/Service/Provisioning/Email/ServiceTemplateProvisionRequest_Approved#create]
        Followed [/System/Policy/ServiceTemplateProvisionRequest_Approved#create]
      Followed [/System/Policy/request_approved#create]
    Followed [/System/Event/RequestEvent/Request/request_approved#create]
    
    (stp_request_59) Following [/System/Event/RequestEvent/Request/request_starting#create]
      (stp_request_59) Following [/System/Policy/request_starting#create]
        (stp_request_59) Following [/System/Process/parse_provider_category#create]
        (stp_request_59) Followed [/System/Process/parse_provider_category#create]
        (stp_request_59) Following [/System/Policy/ServiceTemplateProvisionRequest_starting#create]
          (stp_request_59) Following [/System/CommonMethods/QuotaStatemachine/quota#create]
            (stp_request_59) Following [/System/CommonMethods/QuotaMethods/quota_source#create]
            (stp_request_59) Followed [/System/CommonMethods/QuotaMethods/quota_source#create]
            (stp_request_59) Following [/System/CommonMethods/QuotaMethods/limits#create]
            (stp_request_59) Followed [/System/CommonMethods/QuotaMethods/limits#create]
          (stp_request_59) Followed [/System/CommonMethods/QuotaStatemachine/quota#create]
        (stp_request_59) Followed [/System/Policy/ServiceTemplateProvisionRequest_starting#create]
      (stp_request_59) Followed [/System/Policy/request_starting#create]
    (stp_request_59) Followed [/System/Event/RequestEvent/Request/request_starting#create]
    (stp_request_59) Following [/System/Request/SERVICE_PROVISION_INFO#create]
      (stp_request_59) Following [/Service/Provisioning/Profile/Bit63_user#include_service]
      (stp_request_59) Followed [/Service/Provisioning/Profile/Bit63_user#include_service]
    (stp_request_59) Followed [/System/Request/SERVICE_PROVISION_INFO#create]
    (stp_request_59) Following [/System/Request/UI_PROVISION_INFO#create]
      (stp_request_59) Following [/Infra.../VM/Provisioning/Profile/Bit63_user#get_vmname]
        (stp_request_59) Following [/Infra.../VM/Provisioning/Naming/Default#create]
        (stp_request_59) Followed [/Infra.../VM/Provisioning/Naming/Default#create]
      (stp_request_59) Followed [/Infra.../VM/Provisioning/Profile/Bit63_user#get_vmname]
    (stp_request_59) Followed [/System/Request/UI_PROVISION_INFO#create]

Notice that this *request* processing runs the naming method, which is
therefore processed **before** *CatalogItemInitialization* (which is
processed in *task* context).

## Service Template Provisioning Tasks

Next we see two service template provisioning *tasks* created, our
top-level and child task objects
(service\_template\_provision\_task\_117 and
service\_template\_provision\_task\_118).

> **Note**
> 
> The two tasks are actually running through two separate state
> machines.
> 
> Task *service\_template\_provision\_task\_117* is running through
> */Service/Provisioning/StateMachines/ServiceProvision\_Template/CatalogItemInitialization*.
> This is the top-level or 'parent' service\_template\_provision\_task
> described in [Service Objects](../service_objects/chapter.asciidoc).
> 
> Task *service\_template\_provision\_task\_118* is running through
> */Service/Provisioning/StateMachines/ServiceProvision\_Template/clone\_to\_service*.
> This is the 'child' miq\_request\_task described in [Service
> Objects](../service_objects/chapter.asciidoc).

    (stp_task_117) Following [/Service/Provisioning/StateMachines/Methods/DialogParser#create]
    (stp_task_117) Followed [/Service/Provisioning/StateMachines/Methods/DialogParser#create]
    (stp_task_117) Following [/Service/Provisioning/StateMachines/Methods/CatalogItemInitialization#create]
    (stp_task_117) Followed [/Service/Provisioning/StateMachines/Methods/CatalogItemInitialization#create]
    (stp_task_117) Following [/Service/Provisioning/StateMachines/Methods/Provision#create]
    (stp_task_117) Followed [/Service/Provisioning/StateMachines/Methods/Provision#create]
    (stp_task_117) Following [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (stp_task_117) Followed [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    
    (stp_task_118) Following [/Service/Provisioning/StateMachines/Methods/GroupSequenceCheck#create]
    (stp_task_118) Followed [/Service/Provisioning/StateMachines/Methods/GroupSequenceCheck#create]
    (stp_task_118) Following [/Service/Provisioning/StateMachines/Methods/Provision#create]
    (stp_task_118) Followed [/Service/Provisioning/StateMachines/Methods/Provision#create]
    (stp_task_118) Following [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (stp_task_118) Followed [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]

## VM Provisioning Task

We see our grandchild miq\_provision task object
created(miq\_provision\_119), and processing the
*/Infra…​/VM/Provisioning/StateMachines* methods in the state
machine defined in our user
    profile:

    (miq_provision_119) Following [/Infra.../VM/Lifecycle/Provisioning#create]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/Profile/Bit63_user#get_state_machine]
      (miq_provision_119) Followed [/Infra.../VM/Provisioning/Profile/Bit63_user#get_state_machine]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/VMProvision_vm/template#create]
        (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/Methods/CustomizeRequest#redhat]
        (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/Methods/CustomizeRequest#redhat]
        (miq_provision_119) Following [/Infra.../VM/Provisioning/Placement/default#redhat]
        (miq_provision_119) Followed [/Infra.../VM/Provisioning/Placement/default#redhat]
        (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/Methods/PreProvision#redhat]
        (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/Methods/PreProvision#redhat]
        (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/Methods/Provision#create]
        (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/Methods/Provision#create]
        (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/Methods/CheckProvisioned#create]
        (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/Methods/CheckProvisioned#create]
      (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/VMProvision_vm/template#create]
    (miq_provision_119) Followed [/Infra.../VM/Lifecycle/Provisioning#create]
    (miq_provision_119) Following [/System/Request/UI_PROVISION_INFO#create]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/Profile/Bit63_user#get_placement]
      (miq_provision_119) Followed [/Infra.../VM/Provisioning/Profile/Bit63_user#get_placement]
    (miq_provision_119) Followed [/System/Request/UI_PROVISION_INFO#create]

## Events

We see some events being triggered and handled by the event switchboard:

    Following [/System/Event/EmsEvent/RHEVM/USER_ADD_VM_STARTED#create]
    Followed [/System/Event/EmsEvent/RHEVM/USER_ADD_VM_STARTED#create]

## Service State Machine *CheckProvisioned* States

We see both top-level and child service template provisioning tasks
running their *CheckProvisioned*
    methods:

    ([stp_task_117]) Following /Service/Provisioning/StateMachines/Methods/CheckProvisioned
    ([stp_task_117]) Followed /Service/Provisioning/StateMachines/Methods/CheckProvisioned
    ([stp_task_118]) Following /Service/Provisioning/StateMachines/Methods/CheckProvisioned
    ([stp_task_118]) Followed /Service/Provisioning/StateMachines/Methods/CheckProvisioned

## VM State Machine *CheckProvisioned* State

We see the VM provision state machine running its *CheckProvisioned*
method. We can see the entire */Infra…​/VM/Provisioning/StateMachines*
state machine being re-instantiated for each call of its
*CheckProvisioned* method, including the profile
    lookup:

    (miq_provision_119) Following [/Infra.../VM/Lifecycle/Provisioning#create]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/Profile/Bit63_user#get_state_machine]
      (miq_provision_119) Followed [/Infra.../VM/Provisioning/Profile/Bit63_user#get_state_machine]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/VMProvision_vm/template#create]
        (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/Methods/CheckProvisioned#create]
        (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/Methods/CheckProvisioned#create]
      (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/VMProvision_vm/template#create]
    (miq_provision_119) Followed [/Infra.../VM/Lifecycle/Provisioning#create]

> **Note**
> 
> Recall that if a state exits with `$evm.root['ae_result'] = 'retry'`,
> the entire state machine is re-launched after the retry interval,
> starting at the state to be retried.

We see the service and VM provisioning state machines both running their
*CheckProvisioned* methods for several minutes while the VM provision is
progressing:

    (stp_task_117) Following [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (stp_task_117) Followed [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (miq_provision_119) Following [/Infra.../VM/Lifecycle/Provisioning#create]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/Profile/Bit63_user#get_state_machine]
      (miq_provision_119) Followed [/Infra.../VM/Provisioning/Profile/Bit63_user#get_state_machine]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/VMProvision_vm/template#create]
        (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/Methods/CheckProvisioned#create]
        (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/Methods/CheckProvisioned#create]
      (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/VMProvision_vm/template#create]
    (miq_provision_119) Followed [/Infra.../VM/Lifecycle/Provisioning#create]
    (stp_task_118) Following [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (stp_task_118) Followed [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (stp_task_117) Following [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (stp_task_117) Followed [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (miq_provision_119) Following [/Infra.../VM/Lifecycle/Provisioning#create]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/Profile/Bit63_user#get_state_machine]
      (miq_provision_119) Followed [/Infra.../VM/Provisioning/Profile/Bit63_user#get_state_machine]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/VMProvision_vm/template#create]
        (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/Methods/CheckProvisioned#create]
        (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/Methods/CheckProvisioned#create]
      (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/VMProvision_vm/template#create]
    (miq_provision_119) Followed [/Infra.../VM/Lifecycle/Provisioning#create]
    (stp_task_118) Following [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (stp_task_118) Followed [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]

Once the VM creation has completed we see some more event activity,
including policy processing events. On our system we have a control
policy that requests a SmartState Analysis for every new VM that is
created:

    Following [/System/Event/EmsEvent/RHEVM/USER_ADD_VM_FINISHED_SUCCESS#create]
    Followed [/System/Event/EmsEvent/RHEVM/USER_ADD_VM_FINISHED_SUCCESS#create]
    Following [/System/Event/MiqEvent/POLICY/vm_snapshot_complete#create]
    Followed [/System/Event/MiqEvent/POLICY/vm_snapshot_complete#create]
    Following [/System/Event/MiqEvent/POLICY/vm_create#create]
    Followed [/System/Event/MiqEvent/POLICY/vm_create#create]
    Following [/System/Event/MiqEvent/POLICY/vm_provisioned#create]
    Followed [/System/Event/MiqEvent/POLICY/vm_provisioned#create]
    Following [/System/Event/MiqEvent/POLICY/request_vm_scan#create]
    Followed [/System/Event/MiqEvent/POLICY/request_vm_scan#create]

## Virtual Machine Provision State Machine Continuing

We see the *Infrastructure/VM* provisioning state machine
*CheckProvisioned* method return success, and continue with the
remainder of the state machine (starting with *PostProvision*). This
example creates a VM in a RHEV provider, and we see that within the
internal state machine the provision is actually a two-stage operation;
the initial VM clone operation, followed by a VM reconfiguration task to
set our desired VM configuration - number CPUs, memory size - and so on.

There is considerable event-related activity during a VM provision
operation, as we
    see:

    (miq_provision_119) Following [/Infra.../VM/Lifecycle/Provisioning#create]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/Profile/Bit63_user#get_state_machine]
      (miq_provision_119) Followed [/Infra.../VM/Provisioning/Profile/Bit63_user#get_state_machine]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/VMProvision_vm/template#create]
        (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/Methods/CheckProvisioned#create]
        (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/Methods/CheckProvisioned#create]
        Following [/System/Event/EmsEvent/RHEVM/USER_UPDATE_VM#create]
        Followed [/System/Event/EmsEvent/RHEVM/USER_UPDATE_VM#create]
        Following [/System/Event/MiqEvent/POLICY/vm_scan_start#create]
        Followed [/System/Event/MiqEvent/POLICY/vm_scan_start#create]
        Following [/System/Event/EmsEvent/RHEVM/NETWORK_UPDATE_VM_INTERFACE#create]
        Followed [/System/Event/EmsEvent/RHEVM/NETWORK_UPDATE_VM_INTERFACE#create]
        Following [/System/Event/MiqEvent/POLICY/vm_reconfigure#create]
          Following [/Infra.../VM/Reconfigure/Email/VmReconfigureTaskComplete#create]
          Followed [/Infra.../VM/Reconfigure/Email/VmReconfigureTaskComplete#create]
        Followed [/System/Event/MiqEvent/POLICY/vm_reconfigure#create]
    
        (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/Methods/PostProvision#redhat]
        (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/Methods/PostProvision#redhat]
        (miq_provision_119) Following [/Integration/RedHat/Methods/AddDisk#create]
          Following [/System/Event/EmsEvent/RHEVM/USER_ADD_DISK_TO_VM#create]
          Followed [/System/Event/EmsEvent/RHEVM/USER_ADD_DISK_TO_VM#create]
          Following [/System/Event/EmsEvent/RHEVM/USER_ADD_DISK_TO_VM_FINISHED_SUCCESS#create]
          Followed [/System/Event/EmsEvent/RHEVM/USER_ADD_DISK_TO_VM_FINISHED_SUCCESS#create]
        (miq_provision_119) Followed [/Integration/RedHat/Methods/AddDisk#create]
    
        Following [/System/Event/MiqEvent/POLICY/vm_scan_complete#create]
        Followed [/System/Event/MiqEvent/POLICY/vm_scan_complete#create]
    
        (miq_provision_119) Following [/Integration/RedHat/Methods/StartVM#create]
          Following [/System/Event/EmsEvent/RHEVM/USER_STARTED_VM#create]
          Followed [/System/Event/EmsEvent/RHEVM/USER_STARTED_VM#create]
          Following [/System/Event/MiqEvent/POLICY/request_vm_start#create]
          Followed [/System/Event/MiqEvent/POLICY/request_vm_start#create]
          Following [/System/Event/EmsEvent/RHEVM/USER_RUN_VM#create]
          Followed [/System/Event/EmsEvent/RHEVM/USER_RUN_VM#create]
          Following [/System/Event/MiqEvent/POLICY/vm_start#create]
          Followed [/System/Event/MiqEvent/POLICY/vm_start#create]
        (miq_provision_119) Followed [/Integration/RedHat/Methods/StartVM#create]

## Virtual Machine Provision Complete

Eventually we see the VM provision state machine
    complete:

    (miq_provision_119) Following [/Infra.../VM/Lifecycle/Provisioning#create]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/Profile/Bit63_user#get_state_machine]
      (miq_provision_119) Followed [/Infra.../VM/Provisioning/Profile/Bit63_user#get_state_machine]
      (miq_provision_119) Following [/Infra.../VM/Provisioning/StateMachines/VMProvision_vm/template#create]
        (miq_provision_119) Following [/Infra.../VM/Provisioning/Email/MiqProvision_Complete?event=vm_provisioned#create]
        (miq_provision_119) Followed [/Infra.../VM/Provisioning/Email/MiqProvision_Complete?event=vm_provisioned#create]
        (miq_provision_119) Following [/System/CommonMethods/StateMachineMethods/vm_provision_finished#create]
        (miq_provision_119) Followed [/System/CommonMethods/StateMachineMethods/vm_provision_finished#create]
      (miq_provision_119) Followed [/Infra.../VM/Provisioning/StateMachines/VMProvision_vm/template#create]
    (miq_provision_119) Followed [/Infra.../VM/Lifecycle/Provisioning#create]

## Service Provision Complete

Finally we see both of the *Service* provisioning state machine
*CheckProvisioned* methods return success, and continue with the
remainder of their state
    machines:

    (stp_task_118) Following [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (stp_task_118) Followed [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (stp_task_118) Following [/Service/Provisioning/Email/ServiceProvision_complete?event=service_provisioned#create]
    (stp_task_118) Followed [/Service/Provisioning/Email/ServiceProvision_complete?event=service_provisioned#create]
    (stp_task_118) Following [/System/CommonMethods/StateMachineMethods/service_provision_finished#create]
    (stp_task_118) Followed [/System/CommonMethods/StateMachineMethods/service_provision_finished#create]
    
    Following [/System/Event/MiqEvent/POLICY/service_provisioned#create]
    Followed [/System/Event/MiqEvent/POLICY/service_provisioned#create]
    
    (stp_task_117) Following [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (stp_task_117) Followed [/Service/Provisioning/StateMachines/Methods/CheckProvisioned#create]
    (stp_task_117) Following [/Service/Provisioning/Email/ServiceProvision_complete?event=service_provisioned#create]
    (stp_task_117) Followed [/Service/Provisioning/Email/ServiceProvision_complete?event=service_provisioned#create]
    (stp_task_117) Following [/System/CommonMethods/StateMachineMethods/service_provision_finished#create]
    (stp_task_117) Followed [/System/CommonMethods/StateMachineMethods/service_provision_finished#create]
    
    Following [/System/Event/MiqEvent/POLICY/service_provisioned#create]
    Followed [/System/Event/MiqEvent/POLICY/service_provisioned#create]

## Summary

Tracing the steps of various workflows though *automation.log* can
reveal a lot about the inner workings of the Automation Engine. All
students of automation are encouraged to investigate the
"Following..Followed" message pairs in the logs to get a feel for how
state machines sequence tasks and handle retry operations.
