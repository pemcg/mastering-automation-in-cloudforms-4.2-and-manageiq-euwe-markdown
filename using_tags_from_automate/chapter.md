# Using Tags from Automate

Tags are a very powerful feature of CloudForms and ManageIQ. They allow
us to add *Smart Management* capabilities to the objects in the WebUI
such as virtual machines, hosts, or clusters; to create tag-related
filters; and to group, sort or categorise items by tag.

For example we might assign a tag to a virtual machine to identify which
department or cost centre owns the VM. We could then create a chargeback
rate for billing purposes and assign the rate to all VMs tagged as being
owned by a particular department or cost centre.

We might also tag virtual machines with a 'Location' or 'Data Centre'
tag. We could create a filter view in the WebUI to display all VMs at a
particular location so that we instantly can see which systems might be
affected if we run a data centre failover or power test.

Tags are not only applied to virtual machines. We often tag our virtual
infrastructure components such as hosts, clusters or datastores with a
'Provisioning Scope' tag. When we provision new virtual machines, our
Automate workflow must determine where to put the new VM (a process
known as *placement*). We can use the 'Provisioning Scope' tag to
determine a 'best fit' for a particular virtual machine, based on a
user’s group membership. In this way we might, for example, place all
virtual machines provisioned by users in a development group on a
nonproduction cluster.

These are just three examples of how tags can simplify systems
administration, and help our Automate workflows. Fortunately Automate
has comprehensive support for tag-related operations.

Tags are also the foundation of the Role Based Acccess Control (RBAC)
system. We can use tags to grant access to certain objects for existing
groups of users. Discussing the concept of RBAC is out of scope for this
document, but nevertheless something very important to keep in mind.

We’ve already seen the use of a custom attribute on a virtual machine.
At first glance tags and custom attributes seem to be similar, but there
are good reasons to use one over the other.

Tags are better if we wish to categorise, sort or filter an object based
on its tag. We could for example quickly search for all items tagged
with a particular value. A tag must exist within a category before it
can be used however, and so we have to consider the manageability of tag
categories that contain many hundreds of different tags.

Custom Attributes are better if we just wish to assign a generic text
string to an object, but don’t need to sort or categorise objects by the
attribute name or string.

## Creating Tags and Categories

Tags are defined and used within the context of tag *categories*. We can
check whether a category exists, and if not create it:

``` ruby
unless $evm.execute('category_exists?', 'data_centre')
  $evm.execute('category_create',
                    :name => 'data_centre',
                    :single_value => false,
                    :perf_by_tag => false,
                    :description => "Data Centre")
end
```

We can also check whether a tag exists within a category, and if not
create it:

``` ruby
unless $evm.execute('tag_exists?', 'data_centre', 'london')
  $evm.execute('tag_create',
                    'data_centre',
                    :name => 'london',
                    :description => 'London East End')
end
```

> **Note**
> 
> Tag and category *names* must be lower-case, and optionally contain
> underscores. They have a maximum length of 30 characters. The tag and
> category *descriptions* can be free text.

## Assigning and Removing Tags

We can assign a category/tag to an object (in this case a virtual
machine):

``` ruby
vm = $evm.root['vm']
vm.tag_assign("data_center/london")
```

We can remove a category/tag from an object:

``` ruby
vm = $evm.root['vm']
vm.tag_unassign("data_center/paris")
```

## Testing Whether an Object is Tagged

We can test whether an object (in this case a user group) is tagged with
a particular tag:

``` ruby
ci_owner = 'engineering'
groups = $evm.vmdb(:miq_group).all
groups.each do |group|
  if group.tagged_with?("department", ci_owner)
    $evm.log("info", "Group #{group.description} is tagged")
  end
end
```

## Retrieving an Object’s Tags

We can retrieve the list of all tags assigned to an object using the
`tags` method:

``` ruby
group_tags = group.tags
```

This method also enables us to retrieve the tags in a particular
category (in this case using the tag name as a symbol):

``` ruby
all_department_tags = group.tags(:department)
first_department_tag = group.tags(:department).first
```

> **Tip**
> 
> When called with no argument the `tags` method returns the tags as
> "category/tag" strings. When called with an argument of tag category,
> the method returns the tag name as the string.

## Searching for Specifically Tagged Objects

We can search for objects tagged with a particular tag using the
`find_tagged_with` method:

``` ruby
tag = "/managed/department/syseng"
hosts = $evm.vmdb(:host).find_tagged_with(:all => tag, :ns => "*")
```

This example shows that categories themselves are organised into
namespaces behind the scenes. In practice the only namespace that seems
to be in use is */managed* and we rarely need to specify this.

> **Note**
> 
> The `find_tagged_with` method has a slightly ambiguous past. It was
> present in ManageIQ *Anand*, but returned Active Records rather than
> MiqAeService objects. It disappeared as an Automate method in ManageIQ
> *Botvinnik*, but is thankfully back with ManageIQ *Capablanca*, and
> now returns service model objects as expected.

### Practical example

We could discover all infrastructure components tagged with
*/department/engineering*. We might wish to find out the *service model*
class name of the object, and the object’s name or example. We could
achieve this using the following code snippet:

``` ruby
tag = '/department/engineering'
[:vm_or_template, :host, :ems_cluster, :storage].each do |service_object|
  these_objects = $evm.vmdb(service_object).find_tagged_with(:all => tag,
                                                             :ns => "/managed")
  these_objects.each do |this_object|
    service_model_class = "#{this_object.method_missing(:class)}".demodulize
    $evm.log("info", "#{service_model_class}: #{this_object.name}")
  end
end
```

On a small ManageIQ *Capablanca* system this
    prints:

    MiqAeServiceManageIQ_Providers_Redhat_InfraManager_Template: rhel7-generic
    MiqAeServiceManageIQ_Providers_Redhat_InfraManager_Vm: rhel7srv010
    MiqAeServiceManageIQ_Providers_Openstack_CloudManager_Vm: rhel7srv031
    MiqAeServiceManageIQ_Providers_Redhat_InfraManager_Host: rhelh03.bit63.net
    MiqAeServiceStorage: Data

> **Note**
> 
> This code snippet shows an example of where we need to work with or
> around Distributed Ruby (dRuby). The following loop enumerates through
> *these\_objects*, substituting *this\_object* on each iteration:
> 
> ``` ruby
> these_objects.each do |this_object|
>   ...
> end
> ```
> 
> Normally this is transparent to us and we can refer to the object
> methods such as `name`, and all works as expected.
> 
> Behind the scenes however our automation script is accessing all of
> these objects remotely via its dRuby client object. We must bear this
> in mind if we also wish to find the class name of the remote object.
> 
> If we call *this\_object.class* we get the string "DRb::DRbObject",
> which is the correct class name for a dRuby client object. We have to
> tell dRuby to forward the *class* method call on to the dRuby server,
> and we do this by calling *this\_object.method\_missing(:class)*. Now
> we get returned the full module::class name of the remote dRuby object
> (such as `MiqAeMethodService::MiqAeServiceStorage`), but we can call
> the `demodulize` method on the string to strip the
> `MiqAeMethodService::` module path from the name, leaving us with
> `MiqAeServiceStorage`.

## Getting the List of Tag Categories

On versions prior to ManageIQ *Capablanca*, getting the list of tag
categories was slightly challenging. Both tags and categories are listed
in the same *classifications* table, but tags also have a non-zero
*parent\_id* value that ties them to their category. To find the
categories from the *classifications* table we had to search for records
with a parent\_id of zero:

``` ruby
categories = $evm.vmdb('classification').where(:parent_id => 0)
categories.each do |category|
  $evm.log(:info, "Found category: #{category.name} (#{category.description})")
end
```

With ManageIQ *Capablanca* we now have a `categories` association
directly from an `MiqAeServiceClassification` object, so we can say:

``` ruby
$evm.vmdb(:classification).categories.each do |category|
  $evm.log(:info, "Found category: #{category.name} (#{category.description})")
end
```

## Getting the List of Tags in a Category

We occasionally need to retrieve the list of tags in a particular
category, and for this we have to perform a double lookup - once to get
the classification ID, and again to find `MiqAeServiceClassification`
objects with that parent\_id:

``` ruby
classification = $evm.vmdb(:classification).find_by_name('cost_centre')
cost_centre_tags = {}
$evm.vmdb(:classification).where(:parent_id => classification.id).each do |tag|
  cost_centre_tags[tag.name] = tag.description
end
```

## Finding a Tag’s Name, Given its Description

Sometimes we need to add a tag to an object, but we only have the tag’s
free-text description (perhaps this matches a value read from an
external source). We need to find the tag’s snake\_case name to use with
the `tag_apply` method, but we can use more Rails syntax in our `find`
call to lookup two fields at
once:

``` ruby
department_classification = $evm.vmdb(:classification).find_by_name('department')
tag = $evm.vmdb('classification').where(["parent_id = ? AND description = ?",
                            department_classification.id, 'Systems Engineering']).first
tag_name = tag.name
```

The tag names aren’t in the *classifications* table (just the tag
description). When we call `tag.name`, Rails runs an implicit search of
the *tags* table for us, based on the tag.id:

    irb(main):051:0> tag.name
      Tag Load (0.6ms)  SELECT "tags".* FROM "tags" WHERE "tags"."id" = 44 LIMIT 1
      Tag Inst Including Associations (0.1ms - 1rows)
        => "syseng"

## Finding a Specific Tag (MiqAeServiceClassification) Object

We can just search for the tag object that matches a given category/tag:

``` ruby
tag = $evm.vmdb(:classification).find_by_name('department/hr')
```

> **Tip**
> 
> Anything returned from `$evm.vmdb(:classification)` is an
> `MiqAeServiceClassification` object, not a text string.

## Deleting a Tag or Tag Category

We can now delete a tag or category using the RESTful API:

``` ruby
require 'rest-client'
require 'json'
require 'openssl'
require 'base64'

begin

  def rest_action(uri, verb, payload=nil)
    headers = {
      :content_type  => 'application/json',
      :accept        => 'application/json;version=2',
      :authorization => "Basic #{Base64.strict_encode64("#{@user}:#{@passwd}")}"
    }
    response = RestClient::Request.new(
      :method      => verb,
      :url         => uri,
      :headers     => headers,
      :payload     => payload,
      verify_ssl: false
    ).execute
    return JSON.parse(response.to_str) unless response.code.to_i == 204
  end

  servername   = $evm.object['servername']
  @user        = $evm.object['username']
  @passwd      = $evm.object.decrypt('password')

  uri_base = "https://#{servername}/api/"
  #
  # Delete a tag category
  #
  category = $evm.vmdb(:classification).find_by_name('network_location')
  rest_action("#{uri_base}/categories/#{category.id}", :delete)
  #
  # Delete a tag
  #
  tag = "/managed/department/sales"
  reply = rest_action("#{uri_base}/tags?filter[]=name=#{tag}", :get)
  tag_href = reply['resources'][0]['href']
  rest_action(tag_href, :delete)

  exit MIQ_OK

rescue RestClient::Exception => err
  unless err.response.nil?
    $evm.log(:error, "REST request failed, code: #{err.response.code}")
    $evm.log(:error, "Response body:\n#{err.response.body.inspect}")
  end
  exit MIQ_STOP
rescue => err
  $evm.log(:error, "[#{err}]\n#{err.backtrace.join("\n")}")
  exit MIQ_STOP
end
```

In this example we define a generic method called `rest_action` that
uses the Ruby *rest-client* gem to handle the RESTful connection. We
extract the ManageIQ server’s credentials from the instance schema just
as we did in [Using Schema
Variables](../using_schema_variables/chapter.asciidoc), and we retrieve
the service model of the tag category that we wish to delete, to get its
ID.

Finally we make a RESTful *DELETE* call to the /api/categories URI,
specifying the tag category ID to be deleted.

## Summary

In this chapter we’ve seen how we can work with tags from our automation
scripts, and we’ll use these techniques extensively as we progress
through the book.

### Further Reading

[Creating and Using Tags in Red Hat
CloudForms](https://access.redhat.com/articles/421423)
