# Using Schema Variables

Our simple hello\_world instance in the previous chapter had a very
simple schema, containing a single field that ran a simple
self-contained method. As we become more adventurous with Automate,
we’ll find that it can be useful to associate variables or
*attributes* with an instance. Any Automate method run by the instance
can read these instance attributes, allowing us to define variables or
constants outside of our Ruby scripts. This simplifies maintenance and
promotes code reuse. Methods can also write to these instance
attributes, allowing a degree of data sharing between multiple methods
that might be run in sequence from the same instance.

In our next Automate example, we’ll add some attribute fields to our
class schema, set values for those attributes in our instance, and read
them from our method.

## Preparing the Schema

Let’s edit the schema of the *Methods* Class (see [figure\_title](#i1)).

![Editing the schema of the methods class](images/ss1.png)

​  

We’ll add three attributes, *servername*, *username* and *password*. The
*servername* and *username* attributes will be simple text strings, but
the *password* attribute will have a Data Type of **Password**, meaning
it will be stored in an encrypted form in the database (see
[figure\_title](#i2)).

![Adding attributes](images/ss2.png)

​  

Click **Save**.

We need to ensure that the schema method (our *execute* field) is listed
*after* the three new schema attributes in the field list, otherwise the
attributes won’t be visible to the method when it runs. If necessary,
run **Configuration → Edit sequence** to shuffle the schema fields up or
down (see [figure\_title](#i4)).

![Editing a class schema sequence](images/ss4.png)

​  

## The Instance

Now we’ll create a new instance in our *Methods* class as before, but
this time called *get\_credentials*. We’ll fill in some values for the
*servername*, *username* and *password* schema attributes (see
[figure\_title](#i5)).

![Entering the instance attribute details](images/ss5.png)

​  

Notice that our *password* schema value has been obfuscated.

## The Method

Each of the schema attributes will be available to our method as hash
key/value pairs from `$evm.object`, which is the Automate object
representing our currently running instance.

Our code for this example will be as follows:

``` ruby
$evm.log(:info, "get_credentials started")

servername = $evm.object['servername']
username   = $evm.object['username']
password   = $evm.object.decrypt('password')

$evm.log(:info, "Server: #{servername}, Username: #{username}, Password: \
#{password}")
exit MIQ_OK
```

We’ll create a method in our *Methods* class as we did before, but this
time called *get\_credentials*. We’ll add our code to the **Data** box,
click **Validate**, then **Save**.

## Running the Instance

Finally we’ll run the new instance through **Automate → Simulation**
again, invoking *Call\_Instance* once more with the appropriate
Attribute/Value pairs (see [figure\_title](#i7)).

![Argument name/value pairs for Call\_Instance](images/ss7.png)

​  

We check *automation.log* and see that the attributes have been
retrieved from the instance schema, and the password has been
    decrypted:

    Invoking [inline] method [/ACME/General/Methods/get_credentials] with inputs [{}]
    <AEMethod [/ACME/General/Methods/get_credentials]> Starting
    <AEMethod get_credentials> get_credentials started
    <AEMethod get_credentials> Server: myserver, Username: admin, Password: guess
    <AEMethod [/ACME/General/Methods/get_credentials]> Ending
    Method exited with rc=MIQ_OK

> **Note**
> 
> The password value is encrypted using the *v2\_key* created when the
> CloudForms/ManageIQ database is initialised, and is unique to that
> database (and hence region). If we export an Automate Datastore
> containing encrypted passwords and import it into an appliance in a
> different CloudForms/ManageIQ region, we won’t be able to decrypt the
> password.

## Summary

In this chapter we’ve seen how we can store instance variables called
*attributes* in our schema, that can be accessed by the methods run from
that instance.

Using class or instance schema variables like this is very common. One
example is when we use CloudForms or ManageIQ to provision virtual
machines. The out-of-the-box virtual machine provisioning workflow
includes an approval stage (see [Provisioning
Approval](../provisioning_approval/chapter.asciidoc)), that allows us to
define a default for the number of VMs, and their sizes (CPUs & Memory)
that can be auto-provisioned without administrative approval. The values
**max\_vms**, **max\_cpus** and **max\_memory** used at this workflow
stage are stored as schema attributes in the approval instance, and are
therefore available to us to easily customise without changing any Ruby
code.

When writing our own integration methods, we often need to specify a
valid username and password to connect to other systems outside of
CloudForms/ManageIQ, for example if making a SOAP call to a hardware
load balancer (see [Calling External
Services](../calling_external_services/chapter.asciidoc) for an
example). We can use the technique shown in this example to securely
store and retrieve credentials to connect to anything else in our
Enterprise.
