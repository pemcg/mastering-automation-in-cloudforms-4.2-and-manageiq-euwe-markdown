# Miscellaneous Tips

We’ve reached the final chapter in the book, and our journey towards
automation mastery is almost complete. In this last chapter we’ll cover
some miscellaneous tips that can help us when we work with Automate.

## Updating the Appliance

When a minor update to CloudForms is released and installed, any changes
to the Automate code are not automatically visible to the Automate
Explorer.

Go to **Import /Export**, and **Reset all Datastore custom classes and
instances to default** to get the updates added and visible (see
[figure\_title](#i1)).

![Resetting the locked domains after an appliance
upgrade](images/ss1.png)

​  

> **Note**
> 
> This does not, as the wording might suggest, reset our custom domains,
> it merely reloads the RedHat and ManageIQ domains.

## The ManageIQ Coding Style and Standards Guide

There is a [ManageIQ Coding Style and Standards
Guide](http://manageiq.org/documentation/development/coding_style_and_standards/)
and a [Ruby Style Guide](https://github.com/ManageIQ/ruby-style-guide).
Although the guides don’t specifically refer to Automate Coding style
(it’s more a guideline for ManageIQ code development), we can adopt the
recommendations to keep our code clean and standards-compliant.

The guides recommend using snake\_case for symbols, methods and
variables, and CamelCase for classes and modules. Although this doesn’t
explicitly refer to Automate Datastore Classes and Methods, we can adopt
the same guidelines for our code as well.

> **Tip**
> 
> The style guide doesn’t currently mention whether we should name
> instances in CamelCase or snake\_case. Although a lot of the existing
> code in the Automate datastore uses CamelCase naming for instances,
> from CloudForms 4.1/ManageIQ *Darga* and onwards all new instances and
> methods should be named in snake\_case.

## Defensive Programming

The dynamic nature of the object structure means that we have to be more
careful about testing for **nil** conditions, testing whether hash keys
exist before we access them, test whether variables are enumerable
before we call each on them, and so on.

Some examples are:

``` ruby
if this_object.respond_to?(:attributes)
  if this_object.attributes.respond_to? :each
    this_object.attributes.each do |key, value|
      ...
```

``` ruby
user = $evm.root['user'] rescue nil
unless user.nil?
  ...
```

``` ruby
prov = $evm.root['miq_provision']
if prov.options.key?(:ws_values)
  ws_values = prov.options[:ws_values]
  ...
```

## Catch Exceptions

As an extension of the *Defensive Programming* tip, we should also catch
and handle exceptions wherever possible in our scripts. We have seen
several examples of this in the scripts that we’ve studied in the book,
for example:

``` ruby
begin
...
rescue RestClient::Exception => err
  unless err.response.nil?
    error = err.response
    $evm.log(:error, "The REST request failed with code: #{error.code}")
    $evm.log(:error, "The response body was:\n#{error.body.inspect}")
    $evm.root['ae_reason'] = "The REST request failed with code: #{error.code}"
  end
  $evm.root['ae_result'] = 'error'
  exit MIQ_STOP
rescue => err
  $evm.log(:error, "[#{err}]\n#{err.backtrace.join("\n")}")
  $evm.root['ae_reason'] = "Unspecified error, see automation.log for backtrace"
  $evm.root['ae_result'] = 'error'
  exit MIQ_STOP
end
```

## Use an External IDE

The built-in WebUI code editor is fairly basic. It is often easier to
develop in an external editor or IDE, and copy and paste code into the
built-in editor when complete.

## Version Control

Git integration for the Automate datastore has been introduced in
CloudForms 4.2/ManageIQ *Euwe*.

Several of Red Hat’s US consultants have also created an open source
project for handling version control and continuous integration (CI) of
CloudForms artefacts such as Automate code and dialogs, across regions.
\[1\]

The CI workflow is created using Jenkins. It provides a pipeline view
that allows us to visualize which version of any of the artefacts is in
any region at a given time. We can implement regions as lifecycle stages
in our development process, such as DEV, TEST, QA, and promote code
through the lifecycle as our testing progresses (see
[figure\_title](#i2)).

![Continuous integration workflow for ManageIQ automation
development](images/ci_workflow.png)

​  

## Use Configuration Domains

We have seen several examples in the book where system credentials have
been retrieved from an instance schema using `$evm.object['attribute']`.

When we work on larger projects and implement some kind of version
control as previously described, we will have separate ManageIQ
installations for our various automation code lifecycle environments -
DEV, TEST and QA for example. It is likely (and good practice) that the
credentials to connect to our various integration services will be
different for each lifecycle environment, but we want to be able to
'promote' our code through each environment with minimal change.

In this case it can be useful to create a separate *configuration*
domain for each lifecycle environment, containing purely the classes and
instances that define the usernames, passwords, or URLs specific to that
environment. The configuration domain typically contains no methods;
these are in the 'code' domain being tested. When a method calls
`$evm.object['attribute']`, the attribute is retrieved from the running
instance in the configuration domain, which has the highest priority.

The process of testing then becomes simpler as we cycle the code domain
through each lifecycle environment, without having to modify any
credentials; these are statically defined in the configuration domain.
The process is illustrated in
[table\_title](#promoting-code-domains-through-lifecycle-environments)

| Sprints/Environments | DEV                    | TEST                    | Q/A                   | PROD                    |
| -------------------- | ---------------------- | ----------------------- | --------------------- | ----------------------- |
| Sprint1              | Dev + Code\_v4 Domains | Test + Code\_v3 Domains | QA + Code\_v2 Domains | Prod + Code\_v1 Domains |
| Sprint2              | Dev + Code\_v5 Domains | Test + Code\_v4 Domains | QA + Code\_v3 Domains | Prod + Code\_v2 Domains |
| Sprint3              | Dev + Code\_v6 Domains | Test + Code\_v5 Domains | QA + Code\_v4 Domains | Prod + Code\_v3 Domains |

Promoting Code Domains Through Lifecycle Environments

## Summary

This completes our study of the Automate capability of CloudForms and
ManageIQ. Over the preceding chapters we have learned about the Automate
Datastore and the entities that we use to create our automation scripts.
We have taken a look behind the scenes at the objects that we work with,
and learned about their attributes, virtual columns, associations and
methods.

We discovered how these components come together to create the workflows
that provision infrastructure virtual machines and cloud instances, and
we have seen examples of how we can customise the provisioning state
machines for our own purposes.

We created service catalogs to deploy servers both singly and in
bundles, and we integrated our Automate workflows with an external Red
Hat Satellite 6.2 server.

We have seen how CloudForms and ManageIQ are able to manage our entire
virtual machine lifcycle, including retirement, and we have studied the
retirement process for virtual machines and services.

We looked at the *integration* capabilities of Automate, and saw how
easily we can integrate our automation workflows with our wider
enterprise.

Our journey toward automation mastery is complete. All that is left is
to practice, and start automating\!

1.  The project code is located
    [here](https://github.com/rhtconsulting/miq-ci)
