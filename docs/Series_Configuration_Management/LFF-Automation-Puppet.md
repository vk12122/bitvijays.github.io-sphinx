# Puppet

Puppet code is a declarative language, (describes the desired state for your systems, not the steps needed to get there). Once specified in code those aspects of a system you want Puppet to manage, an agent service running in the background makes any changes needed to bring the system into compliance with the desired state.

Puppet to describe a file resource:

```shell
 puppet resource file /tmp/test
```

What you see is the Puppet code representation of a resource. In this case, the resource's type is file, and its path is /tmp/test:

```ruby
 file { '/tmp/test':
   ensure => 'absent',
 }
```

The resource syntax can be broken down into its components so its anatomy can be understood a little more clearly.

```ruby
 type { 'title':
   parameter => 'value',
 }
```

The type is the kind of thing the resource describes. It can be a core type like a user, file, service, or package, or a custom type that you have implemented yourself or installed from a module.

The body of a resource is a list of parameter value pairs that follow the pattern `parameter => value`. The parameters and possible values vary from type to type. They specify the state of the resource on the system. Documentation for resource parameters is provided in the [Resource Type Reference](https://puppet.com/docs/puppet/latest/types/)

## The agent/master architecture

Puppet is typically used in what's called an agent/master (client/server) architecture.

In this architecture, each managed node in your infrastructure runs a Puppet agent service. One or more servers (depending on the size and complexity of your infrastructure) act as Puppet master(s) and run the Puppet server service to handle communication with your agents.

By default, the Puppet agent service initiates a Puppet run every half-hour. This periodic run ensures that your system stays in the desired state you described in your Puppet code. Any configuration drift that occurs between runs is remediated the next time the agent runs.

1. The Puppet agent begins a Puppet run by sending a catalog request to the Puppet master. This request includes some information about the agent system provided by a tool called Facter.
2. The Puppet master uses this information from Facter along with your Puppet code to compile a catalog that tells the agent exactly how the resources on its system should be configured.
3. Puppet master keeps a copy of the Puppet codebase you've created to define the desired state for systems in your infrastructure. This Puppet code is primarily made up of resource declarations. The master parses this code to create a catalog. The catalog is the final list of system resources that define the desired state for an agent node.
4. The Puppet master sends this catalog back to the Puppet agent, which then uses its providers to check if the desired state of each resource defined in the catalog matches the actual state of the resource on the system. If any differences are found, the providers help Puppet implement whatever changes are necessary to bring the actual state of the system into line with the desired state defined in the catalog.
5. Finally, the Puppet agent generates a report that includes information about unchanged resources, successful changes, and any errors it may have encountered during the run. It sends this report back to the Puppet master, which stores it in PuppetDB and makes it available via the PE console's web GUI.

## Certificates

Puppet requires any system contacting the Puppet master to authenticate with a signed certificate. The first time a Puppet agent contacts the Puppet master, it will submit a certificate signing request (CSR). A Puppet administrator can then validate that the system sending the CSR should be allowed to request catalogs from the master before deciding to sign the certificate.

Use the puppetserver ca tool to list unsigned certificates:

```shell
puppetserver ca list
```

Sign the cert for agent.puppet.vm:

```shell
 puppetserver ca sign --certname agent.puppet.vm
```

First time `puppet agent -t` is executed on the agent system; PluginSync takes place (you'll see a lot of text scroll by). During pluginsync, any extensions installed on the master (such as custom facts, resource types, or providers) are copied to the Puppet agent before the Puppet run continues. This ensures that the agent has all the tools it needs to correctly apply the catalog.

```shell
 Info: Loading facts
 Info: Caching catalog for agent.puppet.vm
 Info: Applying configuration version '1464919481'
```

The above three lines inform that the agent has received a catalog because it tells you when it caches a copy of the new catalog. (The Puppet agent can be configured to fail over to this cached catalog if it is unable to connect to the master.) Finally, the Puppet agent applies the catalog. Normally after this step, you would see a list of all the changes made by the agent.

## Classification

When the Puppet server service on the Puppet master receives a catalog request with a valid certificate, it begins a process called node classification to determine what Puppet code will be compiled to generate a catalog for the agent making the request.

There are three different ways to handle node classification.

- The site.pp manifest is a special file on the master where you can write node definitions.
- The PE console includes a GUI node classifier that makes it easy to manage node groups and classification without editing code directly.
- Finally, if you want to customize node classification, you can create your own external node classifier. An external node classifier can be any executable that takes the name of a node as an argument and returns a YAML file describing the Puppet code to be applied to that node.

### The site.pp manifest

When a Puppet agent contacts the Puppet master, the master checks for any node definitions in the site.pp manifest that match the agent system's name. In the Puppet world, the term "node" is used to mean any system or device in your infrastructure, so a node definition defines how Puppet should manage a given system.

```shell
 vim /etc/puppetlabs/code/environments/production/manifests/site.pp
```

A class defines a group of related resources, allowing them to be declared as a single unit. Using classes in your node definitions keeps them simple and well organized and encourages the reuse of code.

Puppet manifest is Puppet code saved to a file with the .pp extension. This code is written in the Puppet domain specific language (DSL).

The Puppet DSL includes resource declarations, along with a set of other language features, that let you control which resources are applied on a system and what values are set for those resources' parameters.

## Classes and modules

A class is named block of Puppet code. Defining a class combines a group of resources into single reusable and configurable unit. Once defined, a class can then be declared to tell Puppet to apply the resources it contains.

A module is a directory structure that lets Puppet keep track of where to find the manifests that contain your classes. A module also contains other data a class might rely on, such as the templates for configuration files. When you apply a class to a node, the Puppet master checks in a list of directories called a modulepath for a module directory matching the class name. The master then looks in that module's manifests subdirectory to find the manifest containing the class definition. It reads the Puppet code contained in that class definition and uses it to compile a catalog defining a desired state for the node.

### Configured modulepath

```shell
 puppet config print modulepath
 /etc/puppetlabs/code/environments/production/modules:/etc/puppetlabs/code/modules:/opt/puppetlabs/puppet/modules
```

- First directory listed in the modulepath includes modules specific to the production environment.
- Second directory contains modules used across all environments,
- Third is modules that PE uses to configure itself.

..Tip: :: In a real production environment, however, you would likely want to keep your Puppet code in a version control repository such as Git and use Puppet's code manager tool to deploy it to your master.

### Writing a Module

Use `cd` to navigate to the modules directory.

```shell
 cd /etc/puppetlabs/code/environments/production/modules
```

Create a directory structure for a new module called cowsay. (The -p flag allows you to create the cowsay parent directory and manifests subdirectory at once.)

```shell
 mkdir -p cowsay/manifests
```

To include files, we need to have a folder named files inside the module folder like

```shell
 mkdir -p pasture/{manifests,files}
```

With this directory structure in place, it's time to create the manifest where we'll write the cowsay class.

Use vim to create an init.pp manifest in your module's manifests directory.

```shell
 vim cowsay/manifests/init.pp
```

If this manifest is going to contain the cowsay class, you might be wondering why we're calling it init.pp instead of cowsay.pp. Most modules contain a main class like this whose name corresponds with the name of the module itself. This main class is always kept in a manifest with the special name init.pp.

Enter the following class definition, then save and exit (:wq):

```ruby
 class cowsay {
   package { 'cowsay':
     ensure   => present,
     provider => 'gem',
   }
 }
```

It's always good practice to validate your code before you try to apply it. Use the puppet parser tool to check the syntax of your new manifest.

```shell
 puppet parser validate cowsay/manifests/init.pp
```

The parser will return nothing if there are no errors. If it does detect a syntax error, open the file again and fix the problem before continuing. Be aware that this validation can only catch simple syntax errors and won't let you know about other possible errors in your manifests.

Now that the cowsay class is defined in your module's init.pp manifest, your Puppet master knows exactly where to find the appropriate Puppet code when you apply the cowsay class to a node.

In the setup for this quest, the quest tool prepared a cowsay.puppet.vm node. Let's apply the cowsay class to this node. First, open your site.pp manifest.

```shell
 vim /etc/puppetlabs/code/environments/production/manifests/site.pp
```

At the end of the site.pp manifest, insert the following code:

```ruby
 node 'cowsay.puppet.vm' {
   include cowsay
 }
```

Save and exit.

This include cowsay line tells the Puppet master to parse the contents of the cowsay class when it compiles a catalog for the cowsay.puppet.vm node.

Before applying any changes to your system, it's always a good idea to use the `--noop` flag to do a practice run of the Puppet agent. This will compile the catalog and notify you of the changes that Puppet would have made without actually applying any of those changes to your system. It's a good way to catch issues the puppet parser validate command can't detect, and gives you a chance to validate that Puppet will be making the changes you expect.

```shell
 sudo puppet agent -t --noop
```

You should see an output like the following:

```shell
 Notice: Compiled catalog for cowsay.puppet.vm in environment production in
 0.62 seconds
 Notice: /Stage[main]/Cowsay/Package[cowsay]/ensure: current_value
 absent, should be present (noop)
 Notice: Class[Cowsay]: Would have triggered 'refresh' from 1
 events
 Notice: Stage[main]: Would have triggered 'refresh' from 1 events
 Notice: Finished catalog run in 1.08 seconds
```

If your dry run looks good, go ahead and run the Puppet agent again without the --noop flag.

```shell
 sudo puppet agent -t
```

### Composed classes and class scope

A module often includes multiple components that work together to serve a single function. Cowsay alone is great, but many users don't have the time to write out a custom high-quality message on each execution of the command. For this reason, cowsay is often used in conjunction with the fortune command, which provides you and your cow with a database of sayings and wisdom to draw on.

Create a new manifest for your fortune class definition.

```shell
 vim cowsay/manifests/fortune.pp
```

Write your class definition here:

```ruby
 class cowsay::fortune {
   package { 'fortune-mod':
     ensure => present,
   }
 }
```

Note that unlike the main init.pp manifest, the filename of the manifest shows us the name of the class it defines. In fact, because this class is contained in the cowsay module, its full name is cowsay::fortune. The two colons that connect cowsay and fortune are pronounced "scope scope" and indicate that this fortune class is contained in the cowsay module scope. Notice how the fully scoped name of the class tells Puppet exactly where to find it in your module path: the fortune.pp manifest in the cowsay module's manifests directory. This naming pattern also helps avoid conflicts among similarly named classes provided by different modules.

In this case, use a class declaration to pull the cowsay::fortune class into our main cowsay class.

```shell
 vim cowsay/manifests/init.pp
```

Add an include statement for your cowsay::fortune class to the cowsay class.

```ruby
 class cowsay {
   package { 'cowsay':
     ensure   => present,
     provider => 'gem',
   }
   include cowsay::fortune
 }
```

### The package, file, service pattern

A package resource manages the software package itself, a file resource allows you to customize a related configuration file, and a service resource starts the service provided by the software you've just installed and configured.

Next, because Pasture reads from a configuration file, we're going to use a file resource to manage its settings.

Finally, we want Pasture to run as a persistent background process, rather than running once like the cowsay command line tool. It needs to listen for incoming requests and serve out cows as needed. To set this up, we'll first have to create a file resource to define the service, then use a service resource to ensure that it's running.

#### File

Packages you install with Puppet often have configuration files that let you customize their behavior. We've written the Pasture gem to use a simple configuration file. Once you understand the basic concept, it will be easy to extend to more complex configurations.

You already created a files directory inside the pasture module directory. Just like placing manifests inside a module's manifests directory allows Puppet to find the classes they define, placing files in the module's files directory makes them available to Puppet.

Create a pasture_config.yaml file in your module's files directory.

```shell
 vim pasture/files/pasture_config.yaml
```

Include a line here to set the default character to elephant.

```yaml
  :default_character: elephant
```

With this source file saved to your module's files directory, you can use it to define the content of a file resource.

The file resource takes a source parameter, which allows you to specify a source file that will define the content of the managed file. As its value, this parameter takes a URI. While it's possible to point to other locations, you'll typically use this to specify a file in your module's files directory. Puppet uses a shortened URI format that begins with the puppet: prefix to refer to these module files kept on your Puppet master. This format follows the pattern `puppet:///modules/<MODULE NAME>/<FILE PATH>`. Notice the triple forward slash just after `puppet:`. This stands in for the implied path to the modules on your Puppet master.

Return to your `init.pp` manifest.

```shell
 vim pasture/manifests/init.pp
```

And add a file resource declaration.

```ruby
  file { '/etc/pasture_config.yaml':
    source => 'puppet:///modules/pasture/pasture_config.yaml',
   }
```

#### Service

While the cowsay command you installed in the previous quest runs once and exits, Pasture is intended to be run as a service. A Pasture process will run in the background and listen for any incoming requests on its designated port. Because our agent node is running CentOS 7, we'll use the systemd service manager to handle our Pasture process. Although some packages set up their own service unit files, Pasture does not. It's easy enough to use a file resource to create your own. This service unit file will tell systemd how and when to start our Pasture application.

First, create a file called pasture.service.

```shell
 vim pasture/files/pasture.service
```

Include the following contents:

```vi
 [Unit]
 Description=Run the pasture service

 [Service]
 Environment=RACK_ENV=production
 ExecStart=/usr/local/bin/pasture start

 [Install]
 WantedBy=multi-user.target
```

If you're not familiar with the format of a systemd unit file, don't worry about the details here. The beauty of Puppet is that the basic principles you learn with this example will be easily portable to whatever system you work with.

Now open your init.pp manifest again.

```shell
vim pasture/manifests/init.pp
```

First, add a file resource to manage your service unit file.

```ruby
  file { '/etc/systemd/system/pasture.service':
    source => 'puppet:///modules/pasture/pasture.service',
  }
```

Next, add the service resource itself. This resource will have the title pasture and a single parameter to set the state of the service to running.

```ruby
  service { 'pasture':
    ensure => running,
  }
```

#### Resource ordering

There's one more step before this class is complete. We need a way to ensure that the resources defined in this class are managed in the correct order. The "package, file, service" pattern describes the common dependency relationships among these resources: we want to install a package, write a configuration file, and start a service, in that order. Furthermore, if we make changes to the configuration file later, we want Puppet to restart our service so it can pick up those changes.

Though Puppet code will default to managing resources in the order they're written in a manifest, we strongly recommend that you make dependencies among resources explicit through relationship metaparameters. A metaparameter is a parameter, that affects how Puppet handles a resource rather than directly defining its desired state. Relationship metaparameters tell Puppet about ordering relationships among your resources.

For our class, we'll use two relationship metaparameters: before and notify. before tells Puppet that the current resource must come before the target resource. The notify metaparameter is like before, but if the target resource is a service, it has the additional effect of restarting the service whenever Puppet modifies the resource with the metaparameter set. This is useful when you need Puppet to restart a service to pick up changes in a configuration file.

Relationship metaparameters take a resource reference as a value. This resource reference points to another resource in your Puppet code. The syntax for a resource reference is the capitalized resource type, followed by square brackets containing the resource title: Type['title'].

In your init.pp

```shell
 vim pasture/manifests/init.pp
```

Add relationship metaparameters to define the dependencies among your package, file, and service resources. Take a moment to think through what each of these relationship metaparameters does and why.

```ruby
 class pasture {

  package { 'pasture':
    ensure   => present,
    provider => 'gem',
    before   => File['/etc/pasture_config.yaml'],
  }

  file { '/etc/pasture_config.yaml':
    source  => 'puppet:///modules/pasture/pasture_config.yaml',
    notify  => Service['pasture'],
  }

  file { '/etc/systemd/system/pasture.service':
    source  => 'puppet:///modules/pasture/pasture.service',
    notify  => Service['pasture'],
  }

  service { 'pasture':
    ensure => running,
  }

 }
```

### Variables and templates

We introduce variables and templates. Once you assign a value to a variable in a Puppet manifest, you can use that variable throughout the manifest to yield the assigned value. Through templates, you can incorporate these variables into the content of any files your Puppet manifest manages.

A variable name is prefixed with a $ (dollar sign), and a value is assigned with the = operator. Assigning a short string to a variable, for example, looks like this:

```ruby
 $my_variable = 'look, a string!'
```

Once you have defined a variable, you can use it anywhere in your manifest where you want to use the assigned value.

Technically, Puppet variables are actually constants from the perspective of the Puppet parser as it parses your Puppet code to create a catalog. Once a variable is assigned, the value is fixed and cannot be changed. The variability, here, refers to the fact that a variable can have a different value set across different Puppet runs or across different systems in your infrastructure.

#### Templates

Many of the tasks involved in system configuration and administration come down to managing the content of text files. The most direct way to handle this is through a templating language. A template is similar to a text file but offers a syntax for inserting variables as well as some more advanced language features like conditionals and iteration. This flexibility lets you manage a wide variety of file formats with a single tool.

The limitation of templates is that they're all-or-nothing. The template must define the entire file you want to manage. If you need to manage only a single line or value in a file because another process or Puppet module will manage a different part of the file, you may want to investigate Augeas, concat, or the file_line resource type.

##### Embedded Puppet templating language

Puppet supports two templating languages, Embedded Puppet (EPP) and Embedded Ruby (ERB).

EPP templates were released in Puppet 4 to provide a Puppet-native templating language that would offer several improvements over the ERB templates that had been inherited from the Ruby world. Because EPP is now the preferred method, it's what we'll be using in this quest. Once you understand the basics of templating, however, you can easily use the ERB format by referring to the documentation.

An EPP template is a plain-text document interspersed with tags that allow you to customize the content.

First, you'll need to create a templates directory in your pasture module.

```shell
 mkdir pasture/templates
```

Next, create a pasture_config.yaml.epp template file.

``` shell
 vim pasture/templates/pasture_config.yaml.epp
```

Best practice is to begin your EPP template with a parameter tag. This declares which parameters your template will accept and allows you to set their default values. A template will work without this tag, but explicitly declaring your variables here makes your template more readable and easier to maintain.

It's also good practice to add a comment to the beginning of the file so people who might come across it know that it's managed by Puppet and any manual changes they make will be reverted the next time Puppet runs. This comment is intended to be included directly in the final file, so remember to use the comment syntax native to the file format you're working with.

The beginning of your template should look like the following. We'll explain the details of the syntax below.

```ruby
 <%- | $port,
       $default_character,
       $default_message,
 | -%>
 # This file is managed by Puppet. Please do not make manual changes.
```

The bars `(|)` surrounding the list of parameters are a special syntax that define the parameters tag. The `<% and %>` are the opening and closing tag delimiters that distinguish EPP tags from the body of the file. Those hyphens `(-)` next to the tag delimiters will remove indentation and whitespace before and after the tag. This allows you to put this parameter tag at the beginning of the file, for example, without the newline character after the tag creating an empty line at the beginning of the output file.

Next, we'll use the variables we set up to define values for the port and character configuration options.

```ruby
 <%- | $port,
       $default_character,
       $default_message,
 | -%>
 # This file is managed by Puppet. Please do not make manual changes.
 ---
 :default_character: <%= $default_character %>
 :default_message:   <%= $default_message %>
 :sinatra_settings:
 :port: <%= $port %>
```

The `<%= ... %>` tags we use to insert our variables into the file are called expression-printing tags. These tags insert the content of a Puppet expression, in this case the string values assigned to our variables.

Now that this template is set up, let's return to our `init.pp` manifest and see how to use it to define the content of a file resource.

```shell
 vim pasture/manifests/init.pp
```

The file resource type has two different parameters that can be used to define the content of the managed file: source and content.

- source takes the URI of a source file like the ones we've placed in our module's files directory.
- content parameter takes a string as a value and sets the content of the managed file to that string.

To set a file's content with a template, we'll use Puppet's built-in epp() function to parse our EPP template file and use the resulting string as the value for the content parameter.

This epp() function takes two arguments:

- First, a file reference in the format `<MODULE>/<TEMPLATE_NAME>` that specifies the template file to use.
- Second, a hash of variable names and values to pass to the template.

To avoid cramming all our variables into the epp() function, we'll put them in a variable called $pasture_config_hash just before the file resource.

```ruby
class pasture {

  $port                = '80'
  $default_character   = 'sheep'
  $default_message     = ''
  $pasture_config_file = '/etc/pasture_config.yaml'

  package { 'pasture':
    ensure   => present,
    provider => 'gem',
    before   => File[$pasture_config_file],
  }
  $pasture_config_hash = {
    'port'              => $port,
    'default_character' => $default_character,
    'default_message'   => $default_message,
  }
  file { $pasture_config_file:
    content => epp('pasture/pasture_config.yaml.epp', $pasture_config_hash),
    notify  => Service['pasture'],
  }
  file { '/etc/systemd/system/pasture.service':
    source => 'puppet:///modules/pasture/pasture.service',
    notify  => Service['pasture'],
  }
  service { 'pasture':
    ensure    => running,
  }
}
```

Now that that's set, we can repeat the process for the service unit file.

```shell
 vim pasture/templates/pasture.service.epp
```

Add your parameters tag and comment to the beginning of the file. Set the `--config_file` argument of the start command to the value of `$pasture_config_file`

```ruby
 <%- | $pasture_config_file = '/etc/pasture_config.yaml' | -%>
 # This file is managed by Puppet. Please do not make manual changes.
 [Unit]
 Description=Run the pasture service

 [Service]
 Environment=RACK_ENV=production
 ExecStart=/usr/local/bin/pasture start --config_file <%= $pasture_config_file %>

 [Install]
 WantedBy=multi-user.target
```

Now return to your init.pp manifest.

```shell
 vim pasture/manifests/init.pp
```

And modify the file resource for your service unit file to use the template you just created.

```ruby
 class pasture {

  $port                = '80'
  $default_character   = 'sheep'
  $default_message     = ''
  $pasture_config_file = '/etc/pasture_config.yaml'

  package { 'pasture':
    ensure   => present,
    provider => 'gem',
    before   => File[$pasture_config_file],
  }
  $pasture_config_hash = {
    'port'              => $port,
    'default_character' => $default_character,
    'default_message'   => $default_message,
  }
  file { $pasture_config_file:
    content => epp('pasture/pasture_config.yaml.epp', $pasture_config_hash),
    notify  => Service['pasture'],
  }
  $pasture_service_hash = {
    'pasture_config_file' => $pasture_config_file,
  }
  file { '/etc/systemd/system/pasture.service':
    content => epp('pasture/pasture.service.epp', $pasture_service_hash),
    notify  => Service['pasture'],
  }
  service { 'pasture':
    ensure    => running,
  }
}
```

When you're finished, use the puppet parser tool to check your syntax.

## Class Parameters

A well-written module in Puppet should let you customize all its important variables without editing the module itself. This is done with class parameters. Writing parameters into a class allows you to declare that class with a set of parameter-value pairs similar to the resource declaration syntax. This gives you a way to customize all the important variables in your class without making any changes to the module that defines it.

### Writing a parameterized class

A class's parameters are defined as a comma-separated list of parameter name and default value pairs ($parameter_name = default_value,). These parameter value pairs are enclosed in parentheses ((...)) between the class name and the opening curly bracket ({) that begins the body of the class. For readability, multiple parameters should be listed one per line, for example:

```ruby
 class class_name (
   $parameter_one = default_value_one,
   $parameter_two = default_value_two,
 ){
  ...
 }
```

Notice that this list of parameters must be comma-separated, while variables set within the body of the class itself are not. This is because the Puppet parser treats these parameters as a list, while variable assignments in the body of your class are individual statements. These parameters are available as variables within the body of the class.

```ruby
 class pasture (
   $port                = '80',
   $default_character   = 'sheep',
   $default_message     = '',
   $pasture_config_file = '/etc/pasture_config.yaml',
 ){

   package { 'pasture':
     ensure   => present,
     provider => 'gem',
     before   => File[$pasture_config_file],
   }
```

### Resource-like class declarations

Now that your class has parameters, let's see how these parameters are set.

Until now, you've been using include to declare the class as part of your node classification in the site.pp manifest. This include function declares a class without explicitly setting any parameters, allowing any parameters in the class to use their default values. Any parameters without defaults take the special undef value.

To declare a class with specific parameters, use the resource-like class declaration. As the name suggests, the syntax for a resource-like class declaration is very similar to a resource declaration. It consists of the keyword class followed by a set of curly braces ({...}) containing the class name with a colon (:) and a list of parameters and values. Any values left out in this declaration are set to the defaults defined within the class, or undef if no default is set.

```ruby
 class { 'class_name':
   parameter_one => value_one,
   parameter_two => value_two,
 }
```

Unlike the include function, which can be used for the same class in multiple places, resource-like class declarations can only be used once per class. Because a class declared with the include uses defaults, it will always be parsed into the same set of resources in your catalog. This means that Puppet can safely handle multiple include calls for the same class. Because multiple resource-like class declarations are not guaranteed to lead to the same set of resources, Puppet has no unambiguous way to handle multiple resource-like declarations of the same class. Attempting to make multiple resource-like declarations of the same class will cause the Puppet parser to throw an error.

Though we won't go into detail here, you should know that external data-sources like facter and hiera can give you a lot of flexibility in your classes even with the include syntax. For now, you should be aware that though the include function uses defaults, there are ways to make those defaults very intelligent.

Now let's go ahead and use a resource-like class declaration to customize the pasture class from the site.pp manifest. Most of the defaults will still work well, but for the sake of this example, let's set this instance of our Pasture application to use the classic cow character instead of the sheep we had set as the parameter default.

Open your site.pp manifest.

```shell
 vim /etc/puppetlabs/code/environments/production/manifests/site.pp
```

And modify your node definition for pasture.puppet.vm to include a resource-like class declaration. We'll set the default_character parameter to the string 'cow', and leave the other two parameters unset, letting them take their default values.

```ruby
 node 'pasture.puppet.vm' {
   class { 'pasture':
     default_character => 'cow',
   }
 }
```

Notice that with your class parameters set up, all the necessary configuration for all the components of the Pasture application can be handled with a single resource-like class declaration.

## Facter

You can access a standard set of facts with the facter command. Adding the -p flag will include any custom facts that you may have installed on the Puppet master and synchronized with the agent during the pluginsync step of a Puppet run. We'll pass this facter -p command to less so you can scroll through the output in your terminal.

```shell
facter -p | less
facter -p os
facter -p os.family
```

All facts are automatically made available within your manifests. You can access the value of any fact via the $facts hash, following the `$facts['fact_name']` syntax. To access structured facts, you can chain more names using the same bracket indexing syntax. For example, the os.family fact you accessed above is available within a manifest as `$facts['os']['family']`.

```ruby
 class motd {

 $motd_hash = {
    'fqdn'       => $facts['networking']['fqdn'],
    'os_family'  => $facts['os']['family'],
    'os_name'    => $facts['os']['name'],
    'os_release' => $facts['os']['release']['full'],
  }

  file { '/etc/motd':
    content => epp('motd/motd.epp', $motd_hash),
  }

 }
```

The `$facts` hash and top-level (unstructured) facts are automatically loaded as variables into any template.

### Conditional Statements

Conditional statements let you write Puppet code that will behave differently in different contexts.

Because Puppet manages configurations on a variety of systems fulfilling a variety of roles, great Puppet code means flexible and portable Puppet code. Though Puppet's types and providers can translate between Puppet's resource syntax and the native tools on a wide variety of systems, there are still a lot questions that you as a Puppet module developer will need to answer yourself.

A good rule of thumb is that the resource abstraction layer answers how questions, while the Puppet code itself answers what questions.

As an example, let's take a look at the puppetlabs-apache module. While this module's developers rely on Puppet's providers to determine how the Apache package is installed—whether it's handled by yum, apt-get or another package manager—Puppet doesn't have any way of automatically determining the name of the package that needs to be installed. Depending on whether the module is used to manage Apache on a RedHat or Debian system, it will need to manage either the httpd or apache2 package.

Simplified to show only the values we're concerned with, the conditional statement looks like this:

```ruby
 if $::osfamily == 'RedHat' {
   ...
   $apache_name = 'httpd'
   ...
 } elsif $::osfamily == 'Debian' {
   ...
   $apache_name = 'apache2'
   ...
 }
```

Here, the $apache_name variable is set to either httpd or apache2 depending on the value of the $::osfamily fact. Elsewhere in the module, you'll find a package resource that uses this $apache_name variable to set its name parameter.

```ruby
 package { 'httpd':
   ensure => $ensure,
   name   => $::apache::apache_name,
   notify => Class['Apache::Service'],
 }
```

Conditional statements return different values or execute different blocks of code depending on the value of a specified variable.

Puppet supports a few different ways of implementing conditional logic:

- if statements,
- unless statements,
- case statements, and
- selectors.

Because the same concept underlies these different forms of conditional logic available in Puppet, we'll only cover the if statement in the tasks for this quest. Once you have a good understanding of how to implement if statements, we'll leave you with descriptions of the other forms and some notes on when you may find them useful.

An if statement includes a condition followed by a block of Puppet code that will only be executed if that condition evaluates as true. Optionally, an if statement can also include any number of elsif clauses and an else clause.

If the if condition fails, Puppet moves on to the elsif condition (if one exists).
If both the if and elsif conditions fail, Puppet will execute the code in the else clause (if one exists).
If all the conditions fail, and there is no else block, Puppet will do nothing and move on.

```ruby
   if ($sinatra_server == 'thin') or ($sinatra_server == 'mongrel')  {
    package { $sinatra_server:
      provider => 'gem',
      notify   => Service['pasture'],
    }
```

## Puppet Forge

The Puppet Forge is a public repository for Puppet modules. The Forge gives you access to community maintained modules. Using existing modules from the Forge allows you to manage a wide variety of applications and systems without spending extensive time on custom module development.

For those who put a high premium on vetted code and active maintenance, the Forge maintains lists of modules that have been checked against standards set by Puppet's Forge team. Modules in the Puppet Approved category are audited to ensure quality, reliability, and active development. Modules in the Puppet Supported list are covered by Puppet Enterprise support contracts and are maintained across multiple platforms and versions over the course of the Puppet Enterprise release lifecycle.

## Puppet Master

- Best way to learn puppet is to download [Puppet Learning VM](https://puppet.com/download-learning-vm) and perform the quests.

- Enable the [Puppet platform on APT](https://puppet.com/docs/puppet/latest/puppet_platform.html#task-383)

- Basically visit [Puppet Repo](https://apt.puppetlabs.com/) and choose latest puppet server release for your operating system. I currently run Debian 9 stretch, so for me it's [Xenial](https://apt.puppetlabs.com/puppet6-release-xenial.deb)

- Setup the permanent IP address using /etc/network/interfaces

## Install PuppetServer

```shell
 cd /tmp; wget https://apt.puppetlabs.com/puppet6-release-xenial.deb --no-check-certificate; dpkg -i /tmp/puppet6-release-xenial.deb; apt-get update; apt-get install puppetserver; /opt/puppetlabs/bin/puppetserver ca setup; systemctl start puppetserver; apt-get install locate man vim
```

We would utilise `razor <https://github.com/puppetlabs/razor-server>`_to do most of the automatic provision.

`Installation Step of razor-server <https://github.com/puppetlabs/razor-server/wiki/Installation>`_

1. Database setup : Install puppetlabs-postgresql

::

 class { 'postgresql::globals':
   manage_package_repo => true,
   version             => '9.2',
 }->

 class { 'postgresql::server': }

 postgresql::server::db { 'razor':
   user     => 'razor',
   password => postgresql_password('razor', 'secret'),
 }

## Bolt

Bolt is an open-source remote task runner; connects directly to remote nodes with SSH or WinRM without any agent installation on the target box. Can be used to automate tasks that you need to perform on your infrastructure on an ad hoc basis, such as troubleshooting, deploying an application, stopping and starting services, and upgrading a database schema.

```shell
 bolt --help | more
 Usage: bolt <subcommand> <action> [options]

 Available subcommands:
  bolt command run <command>       Run a command remotely
  bolt file upload <src> <dest>    Upload a local file
  bolt script run <script>         Upload a local script and run it remotely
  bolt apply <manifest>            Apply Puppet manifest code

 where [options] are:
     -n, --nodes NODES                Identifies the nodes to target.
 Authentication:
     -u, --user USER                  User to authenticate as
     -p, --password [PASSWORD]        Password to authenticate with. Omit the value to prompt for the password.
 Escalation:
         --run-as USER                User to run as using privilege escalation
         --sudo-password [PASSWORD]   Password for privilege escalation. Omit the value to prompt for the password.
 Display:
         --format FORMAT              Output format to use: human or json
```
