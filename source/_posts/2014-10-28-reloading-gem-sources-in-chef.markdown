---
layout: post
title: Reloading gem sources in Chef
date: 2014-10-28 11:39:11 -0700
comments: true
categories: chef ruby gem rubygems
---
I had a situation where we had private gems hosted on an internal repo that needed to be installed on our servers.  To solve this I decided to write a [gemrc][gem-env] configuration file.  This file and its contents seem to always be in flux, depending on the version of Rubygems that is installed.

[gem-env]: http://guides.rubygems.org/command-reference/#gem-environment

To solve this problem I wrote the following cookbook (only relevant snippets shown):

In `recipes/default.rb`:

{% highlight ruby %}
gem_sources = ["http://rubygems.org/", "http://custom_server:12345/"]

template '/etc/gemrc' do
  source 'gemrc.erb'
  variables {(
    :gem_sources => gem_sources
  )}
  notifies :create, "ruby_block[refresh_gemrc]", :immediately
end

# Some tools don't read /etc/gemrc and only look at the user one
home_dir = `echo ~`.strip
template "#{home_dir}/.gemrc" do
  source "gemrc.erb"
  variables {(
    :gem_sources => gem_sources
  )}
  owner File.stat(home_dir).uid
  group File.stat(home_dir).god
  notifies :create, "ruby_block[refresh_gemrc]", :immediately
end

# This updates the current ruby context.  It is not ideal because
# it uses internal APIs that are not guaranteed
ruby_block "refresh_gemrc" do
  action :nothing
  block do
    Gem.configuration = Gem::ConfigFile.new []
  end
end
{% endhighlight %}

In `templates/default/gemrc.erb`:

{% highlight yaml %}
---
:sources:
<% @gem_sources.each do |source| %>
- <%= source %>
<% end %>
:verbose: true
gem: --no-doc
{% endhighlight %}

By specifying the `/etc/gemrc` file manually (and then reloading it on the same chef-run) I was able to install gems from our custom gem server on bootstraping a node.  Gem installs also went MUCH faster since I added the `--no-doc` argument which prevented download RI and RDoc for each gem.