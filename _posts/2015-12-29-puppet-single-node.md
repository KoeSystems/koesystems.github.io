---
layout: post
title: Install puppet master and agent in single node
description: "How to install puppet in single node for development environments"
date: 2015-12-29 00:00:00
tags:
- puppet
categories:
- puppet
twitter_text: 'puppet'
---

If we need to write, modify or test a puppet module, we can do it in several ways: we can modify our code in our puppet master directly (obviously not recommended) or we can create a new environment based in a git branch and use [Directory Environments](https://docs.puppetlabs.com/puppet/3.8/reference/environments_configuring.html) or we can configure puppet master and agent in our development workstation.

I think that this last option is particularly helpful every time I start a new puppet project, making the puppet development faster at the beginning of the project.

So, let's install and configure both in one **single node**.

###Cookbook

We will use Ubuntu 14.04 LTS (trusty) and Puppet 3.8.4

First of all we need to install puppetlabs repositories, but don't worry, puppetlabs did most of the work for us.

{% highlight bash %}
wget https://apt.puppetlabs.com/puppetlabs-release-trusty.deb
dpkg -i puppetlabs-release-trusty.deb
{% endhighlight %}

Then we need to install puppet master and puppet agent, in this case we will specify the version in both packages.

{% highlight bash %}
apt-get update
apt-get install puppetmaster=3.8.4-1puppetlabs1
apt-get install puppet=3.8.4-1puppetlabs1
{% endhighlight %}

Also and optionally, I like to install some tools in development environment that will help us in development process.

{% highlight bash %}
apt-get install vim-puppet puppet-lint
{% endhighlight %}

Now we need to configure both puppet services, we can do it using the same **/etc/puppet/puppet.conf** config file, just defining specific sections.

1. **[main]** is the global section used by all commands and services. It can be overridden by the other sections.
2. **[master]** is used by the puppet master service.
3. **[agent]** is used by the puppet agent service.

{% highlight ini %}
[main]
confdir          = /etc/puppet
logdir           = /var/log/puppet
vardir           = /var/lib/puppet
ssldir           = /var/lib/puppet/ssl
rundir           = /var/run/puppet
environmentpath  = $confdir/environments
factpath         = $vardir/lib/facter
pluginsource     = puppet:///plugins
pluginsync       = true
srv_domain       = localhost
strict_variables = true
parser           = future

[master]
certname                 = localhost
report                   = true
reports                  = log
ssl_client_header        = HTTP_X_CLIENT_DN
ssl_client_verify_header = HTTP_X_CLIENT_VERIFY
	  
[agent]
certname   = localhost
server     = localhost
pluginsync = true
report     = true
summarize  = true
{% endhighlight %}

Note that we enable reports and summaries in the agent section, this will help us during the module development. Also we enable the future parser to be ready for puppet 4.x versions.

Now we can restart puppet master

{% highlight bash %}
service puppetmaster restart
{% endhighlight %}

Because we used the **environmentpath** variable in the main section we will configure puppet with Directory Environments, making our puppet environment more flexible.

We need to create a new environment directory **environments** in /etc/puppet and then create a new environment with any name, in our case will be **development**.

{% highlight bash %}
mkdir -p /etc/puppet/environments/development
{% endhighlight %}

We can leave the **auth.conf** and **fileserver.conf** files by default.

If you want to use hieradata integrated with the environments you need to create a **hiera.yaml** file with the proper configuration, for example...

{% highlight yaml %}
---
:backends:
  - yaml
:yaml:
  :datadir: /etc/puppet/environments/%{::environment}/hieradata 
:hierarchy:
  - "node/%{::fqdn}"
  - common
{% endhighlight %}

Each environment should have the standard tree directory with **hieradata**, **manifests** and **modules**, and also an **environment.conf** file where we need to define/overwrite some puppet variables.

{% highlight bash %}
# cat environment.conf 
modulepath = modules
{% endhighlight %}

Now we can define our **default.pp** and **localhost.pp** nodes in the manifests directory

{% highlight puppet %}
# cat default.pp 
node default {
}
{% endhighlight %}

{% highlight puppet %}
# cat localhost.pp 
node 'localhost' {
  notify {"Loding localhost node configuration":}
}
{% endhighlight %}

And finally we can test our puppet configuration running puppet agent

{% highlight bash %}
puppet agent -t

Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for localhost
Info: Applying configuration version '1451264390'
Notice: Loding localhost node configuration
Notice: /Stage[main]/Main/Node[localhost]/Notify[Loding localhost node configuration]/message: defined 'message' as 'Loding localhost node configuration'
Notice: Finished catalog run in 0.48 seconds
Changes:
            Total: 1
Events:
          Success: 1
            Total: 1
Resources:
          Changed: 1
      Out of sync: 1
            Total: 8
Time:
         Schedule: 0.00
           Notify: 0.00
   Config retrieval: 0.08
            Total: 0.08
         Last run: 1451264391
       Filebucket: 0.00
Version:
           Config: 1451264390
           Puppet: 3.8.4
{% endhighlight %}

Now you can start to write your puppet modules inside /etc/puppet/environments/development/modules.

That's all!