---
layout: post
title: "Managing Files with Puppet"
date: 2014-03-29 11:23:55 -0700
comments: true
categories: 
---
Working with files is one of the most common tasks when managing systems and therefore a routine task when working with a tool like Puppet. There are many to manage files with Puppet and last year I hosted a [webinar](http://puppetlabs.com/webinars/using-puppet-forge-manage-configuration-files) about some of the options. I've been thinking a lot about them lately and thought others would find a written version useful. 

For the purpose of comparison, all I'm looking to do is manage the configuration of /etc/ssh/sshd_config on a single Red Hat server. More Puppet code would be necessary to fully manage an ssh server and accommodate for multiple operating systems. Thankfully, the [Puppet Forge](http://forge.puppetlabs.com) offers plenty of options. 


### Static File Serving

{% codeblock lang:puppet sshd_config.rb %}
file { '/etc/ssh/sshd_config':
  ensure => file,
  mode   => '0600',
  owner  => 'root',
  group  => 'root',
  source => 'puppet:///modules/my_module/my_sshd_config',
}
{% endcodeblock %}

This is the most blunt instrument you can wield with Puppet. We're instructing Puppet to take a file we have (the source line) and deliver that file to the machine we're managing with a particular mode, owner and group. Puppet will first compare checksums and if a change is needed, it will overwrite the file, reporting back a diff of what changed.

It's quick and easy but dirty. This technique forces everyone to manage that file through the same Puppet-sourced file (hope you're using version control) and every machine under management will get the same file, making it more complicated to account for differences between systems.


### Puppet Templates

{% codeblock lang:puppet sshd_config.rb %}
file { '/etc/ssh/sshd_config':
  ensure   => file,
  mode     => '0600',
  owner    => 'root',
  group    => 'root',
  content  => template('my_module/sshd_template.erb'),
}
{% endcodeblock %}

Templates are much more practical. The [documentation on Puppet templates](http://docs.puppetlabs.com/guides/templating.html) will explain it in more detail but effectively, you're able to use information about the machine you're managing to tweak the configuration you actually manage on it. An example for sshd_config is the [ListenAddress](http://www.openssh.org/cgi-bin/man.cgi?query=sshd_config) keyword. If we want it to be set to the ip address assigned to our machines eth0 adaptor, we simply need to specify the following in our sshd_config modeled as an erb template.

`ListenAddress <%= @ipaddress_eth0 %>`

Whenever Puppet runs on the machine you're managing, it delivers the value for ipaddress_eth0 (run `facter ipaddress_eth0` to see it now) to the Puppet Master which builds the appropriate sshd_config file for the machine to manage, similarly to how static files are managed by Puppet. It's much more flexible but templating still expects you to manage the entire file with Puppet, using the same source file. 


### Concatenating File Fragments

{% codeblock lang:puppet concat.rb %}
$file_to_manage = '/etc/ssh/sshd_config'
	
concat{ $file_to_manage:
  owner => 'root',
  group => 'root',
  mode  => '0600',
}
	
concat::fragment { 'Disclaimer':
  ensure  => present,
  target  => $file_to_manage,
  order   => '01',
  content => "# This file is managed by Puppet - your edits will be overwritten\n",
}
	
concat::fragment { 'Listen':
  ensure  => present,
  target  => $file_to_manage,
  order   => '02',
  content => "ListenAddress ${ipaddress_eth0}\n",
}
	
concat::fragment { 'Base':
  ensure => present,
  target => $file_to_manage,
  order  => '10',
  source => 'puppet:///modules/my_module/sshd_config_base',
}
{% endcodeblock %}

This pattern involves many more lines of Puppet code but it's infinitely more flexible than what you've seen so far. These concat resources come from the [puppetlabs-concat](http://forge.puppetlabs.com/puppetlabs/concat) module from the [Puppet Forge](http://forge.puppetlabs.com). If you boil it down, it's an abstraction above static file serving and Puppet template creation. You tell Puppet that you care about individual fragments of a file which don't have to be defined in the same spot in your Puppet code. Those fragments are put together on the machine you're managing. You can even configure it such that local users on the system can add their own fragments that won't be removed by Puppet, if that's what works for you. 


### Scalpel Precision with Augeas

{% codeblock lang:puppet augeas.rb %}
sshd_config { 'ListenAddress':
  ensure => present,
  value  => $ipaddress_eth0
}
	
sshd_config { 'PermitRootLogin':
  ensure => present,
  value  => 'no',
}
{% endcodeblock %}

Finally, the pattern I'm most excited about. These Puppet resources are provided by the [domcleal-augeasproviders](http://forge.puppetlabs.com/domcleal/augeasproviders) Forge module. The foundation is the rock-solid [Augeas](http://augeas.net/) file manipulation tool. It specializes in understanding every element of a configuration file which allows it to make very specific modifications without requiring the entire file to be managed. Purists may use the [built-in augeas type](http://docs.puppetlabs.com/references/latest/type.html#augeas) but for the rest of us, the Forge module combines the simplicity of the Puppet DSL with the flexibility and power of Augeas. 

It's so elegant that you can even ask Puppet to tell you about the state of every keyword in sshd_config in it's own language. 

{% codeblock lang:puppet resource.pp %}
[root@rhel ~]# puppet resource sshd_config
sshd_config { 'ChallengeResponseAuthentication':
  ensure => 'present',
  value  => ['no'],
}
sshd_config { 'ClientAliveInterval':
  ensure => 'present',
  value  => ['420'],
}
sshd_config { 'GSSAPIAuthentication':
  ensure => 'present',
  value  => ['yes'],
}
…
{% endcodeblock %}

If you combine the sshd_config resources you want to manage with a special resource named [resources](http://docs.puppetlabs.com/references/latest/type.html#resources), you can instruct Puppet to include _only_ the things you've expressed in your managed machines sshd_config. Surgical precision or a nuke from orbit, it's your choice. 

Try this stuff by installing [Puppet](http://docs.puppetlabs.com/guides/installation.html) or try [Puppet Enterprise](http://puppetlabs.com/puppet/puppet-enterprise) with everything ready to go for free on up to 10 machines. Learn much more about using Puppet at http://puppetlabs.com/learn.
