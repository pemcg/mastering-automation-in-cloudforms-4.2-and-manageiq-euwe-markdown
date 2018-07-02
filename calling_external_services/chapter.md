# Calling External Services

We saw in [Calling Automation Using the RESTful
API](../calling_automation_using_the_restful_api/chapter.asciidoc) how
external systems can make *incoming* calls to CloudForms or ManageIQ
using the RESTful API, and run Automate instances, perhaps to initiate
workflows that we’ve defined.

From Automate we can also make *outgoing* calls to external systems. We
typically use SOAP or RESTful APIs to access theses external services,
and there are several Ruby Gems that make this easy for us, including
Savon (SOAP client), RestClient, XmlSimple and Nokogiri (XML parsers),
and Fog (a Ruby Cloud Services Library).

We have already seen an example of making a RESTful API connection to
the RHEV Manager in [Customising VM
Provisioning](../../customising_vm_provisioning/chapter.asciidoc). Now
we will look at some more ways that we can integrate with external
services.\[1\]

## Calling a SOAP API Using the Savon Gem

The following snippet shows an example of making a SOAP call to an f5
BIG-IP load balancer to add an IP address to a pool (some lines have
been omitted for brevity/clarity):

``` ruby
  def call_F5_Pool(soap_action, body_hash=nil)
    servername = nil || $evm.object['servername']
    username   = nil || $evm.object['username']
    password   = nil || $evm.object.decrypt('password')

    require "rubygems"
    gem 'savon', '=2.3.3'
    require "savon"
    require 'httpi'

    # configure httpi gem to reduce verbose logging
    HTTPI.log_level = :info # changing the log level
    HTTPI.log       = false # diable HTTPI logging
    HTTPI.adapter   = :net_http # [:httpclient, :curb, :net_http]

    soap = Savon.client do |s|
      s.wsdl "https://#{servername}/iControl/iControlPortal.cgi? \
                                                    WSDL=LocalLB.Pool"
      s.basic_auth [username, password]
      s.ssl_verify_mode :none
      s.endpoint "https://#{servername}/iControl/iControlPortal.cgi"
      s.namespace 'urn:iControl:LocalLB/Pool'
      s.env_namespace :soapenv
      s.namespace_identifier :pool
      s.raise_errors false
      s.convert_request_keys_to :none
      s.log_level :error
      s.log false
    end

    response = soap.call soap_action do |s|
      s.message body_hash unless body_hash.nil?
    end

    # Convert xml response to a hash
    return response.to_hash["#{soap_action}_response".to_sym][:return]
  end
  ...
  vm.ipaddresses.each do |vm_ipaddress|
    body_hash = {}
    body_hash[:pool_names] = {:item => [f5_pool]}
    body_hash[:members] = [{:items =>
                            { :member =>
                               {:address => vm_ipaddress,
                                :port => f5_port}
                             }
                           }]
    # call f5 and return a hash of pool names
    f5_return = call_F5_Pool(:add_member, body_hash)
  end
```

This script defines a method call\_F5\_Pool that handles the connection
to the load balancer. The method first retrieves the connecting
credentials from the instance schema, and then specifies a particular
version of the *Savon* Gem to use, and sets the required HTTP logging
levels. It initialises the Savon client with the required parameters
(including a WSDL path), and then makes the SOAP call. The method
finally returns with the SOAP XML return string formatted as a Ruby
hash.

The method is called in a loop, passing an IP address into the
body\_hash argument on each iteration.

## Calling an OpenStack API Using the Fog Gem

The *fog* gem is a multipurpose cloud services library that supports
connectivity to a number of cloud providers.

The following code uses the
[fog-openstack](https://github.com/fog/fog-openstack) gem to retrieve
OpenStack networks from Neutron, and present them as a dynamic drop-down
dialog list. The code filters networks that match a tenant’s name, and
assumes that the CloudForms/ManageIQ user’s group has a tenant tag
containing the same name:

``` ruby
require 'fog/openstack'
begin
  tenant = $evm.root['user'].current_group.tags(:tenant).first
  $evm.log(:info, "Tenant name: #{tenant}")

  dialog_field = $evm.object
  dialog_field["sort_by"] = "value"
  dialog_field["data_type"] = "string"
  openstack_networks = {}
  openstack_networks[nil] = '< Select >'
  ems = $evm.vmdb('ems').find_by_name("OpenStack DC01")
  raise "ems not found" if ems.nil?

  neutron_service =  Fog::Network::OpenStack.new({
    :openstack_api_key  => ems.authentication_password,
    :openstack_username => ems.authentication_userid,
    :openstack_auth_url => "https://#{ems.hostname}:#{ems.port}/v2.0/tokens",
    :openstack_tenant   => tenant,
    :connection_options => {:ssl_verify_peer => false}
  })

  keystone_service = Fog::Identity::OpenStack.new({
    :openstack_api_key  => ems.authentication_password,
    :openstack_username => ems.authentication_userid,
    :openstack_auth_url => "https://#{ems.hostname}:#{ems.port}/v2.0/tokens",
    :openstack_tenant   => tenant,
    :connection_options => {:ssl_verify_peer => false}
  })

  tenant_id = keystone_service.current_tenant["id"]
  $evm.log(:info, "Tenant ID: #{tenant_id}")
  networks = neutron_service.networks.all
  networks.each do |network|
    $evm.log(:info, "Found network #{network.inspect}")
    if network.tenant_id == tenant_id
      network_id = $evm.vmdb('CloudNetwork').find_by_ems_ref(network.id)
      openstack_networks[network_id] = network.name
    end
  end

  dialog_field["values"] = openstack_networks
  exit MIQ_OK

rescue => err
  $evm.log(:error, "[#{err}]\n#{err.backtrace.join("\n")}")
  exit MIQ_STOP
end
```

This example first retrieves the value of a tenant tag applied to the
current user’s access control group. It then makes a fog connection to
both Neutron and Keystone, using the Fog::Network.new and
Fog::Identity.new calls, specifying a :provider type of 'OpenStack', the
credentials defined for the ManageIQ OpenStack provider, and the tenant
name retrieved from the tag.

The script iterates though all of the Neutron networks, matching those
with a tenant\_id that matches our tenant tag. If a matching network is
found it retrieves the 'CloudNetwork' service model object ID for the
network and uses that as the key for the hash that populates the dynamic
drop-down list. The corresponding hash value is the network name
retrieved from Neutron.

## Reading from a MySQL Database Using the MySQL Gem

We can add gems to our ManageIQ appliance if we wish. The following code
snippet uses the *mysql* gem to connect to a MySQL-based CMDB to extract
project codes and create tags from them:

``` ruby
require 'rubygems'
require 'mysql'

begin
  server   = $evm.object['server']
  username = $evm.object['username']
  password = $evm.object.decrypt('password')
  database = $evm.object['database']

  con = Mysql.new(server, username, password, database)

  unless $evm.execute('category_exists?', "project_code")
    $evm.execute('category_create', :name => "project_code",
                                    :single_value => true,
                                    :description => "Project Code")
  end
  con.query('SET NAMES utf8')
  query_results = con.query('SELECT description,code FROM projectcodes')
  query_results.each do |record|
    tag_name = record[1]
    tag_display_name = record[0].force_encoding(Encoding::UTF_8)

    unless $evm.execute('tag_exists?', 'project_code', tag_name)
      $evm.execute('tag_create', "project_code", :name => tag_name,
                                                :description => tag_display_name)
    end
  end
end
rescue Mysql::Error => e
  puts e.errno
  puts e.error
ensure
  con.close if con
end
```

This example first makes a connection to the MySQL database, using
credentials stores in the instance schema. It then checks that the tag
category exists, before specifying `'SET NAMES utf8'` \[2\], and making
a SQL query to the database to retrieve a list of project codes and
descriptions. Finally the script iterates through list of project codes
returned, creating a tag for each corresponding code.

## Summary

These examples show the flexibility that we have to integrate with other
enterprise components. We have called a load balancer API as part of a
provisioning operation to add new IP addresses to its pool. This enables
us to completely automate the auto-scaling of our application workload.
We have called two OpenStack components to populate a dynamic drop-down
list in a service dialog, and we have made a SQL call to a MySQL
database to extract a list of project codes and create tags from them.

### Further Reading

[Heavy metal SOAP client](https://github.com/savonrb/savon)

[The Ruby cloud services library](https://github.com/fog/fog)

[MySQL API module for Ruby](https://rubygems.org/gems/mysql/)

1.  There are more and complete examples of integration code on
    [GitHub](https://github.com/ramrexx)

2.  This is required if the database contains "non-English" strings with
    character marks such as umlauts
