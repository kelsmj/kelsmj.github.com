---
layout: post
title: "Using Symlinks and Virtual hosts to serve pages outside of Sites directory using Apache on the MAC"
date: 2011-09-16 14:30
description: Step by step instructions to serve pages outside of the Sites directory on the MAC (Lion) using Symlinks and Virtual Hosts.
comments: true
categories: apache mac
---

I like keeping all my code and websites in a single "Code" directory on my machine, but it isn't straightforward, from what I can tell, to get Apache on the MAC 
to serve up pages anywhere outside of the default "Sites" directory that is created when you enable "Web Sharing" on the MAC.  
After doing a lot of searching, I finally found a solution that works for me. 

I doubt this is the only solution, or even the "proper" solution, but it seems to work.

Below are all the steps I took in order to serve pages out of directories other then those in the "Sites" directory.  For these steps, I will assume there is an existing directory of "~/Code/MyWebSite" that has a simple "Hello World" index.html file in it.

- Enable Web Sharing on the MAC by going to System Prefrences --> Sharing --> Check Enable Web Sharing

- Edit your <i>username</i>.conf file located in /private/etc/apache2/users and add the "FollowSymLinks" directive

    {% codeblock console %}
    <Directory "/Users/yourUserName/Sites/">
        Options Indexes MultiViews FollowSymLinks
        AllowOverride None
        Order allow,deny
        Allow from all
    </Directory>
    {% endcodeblock %}

- Edit the /private/etc/apache2/httpd.conf file and make sure the line under "# Virtual hosts" is not commented out, like so:

    {% codeblock console %}
    Include /private/etc/apache2/extra/httpd-vhosts.conf
    {% endcodeblock %}

- Edit the /private/etc/apache2/extra/httpd-vhosts.conf file and add:

    {% codeblock console %}
    <VirtualHost *:80>  
        <Directory /Users/yourUserName/Sites/MyWebSite.com>
            Options +FollowSymlinks +SymLinksIfOwnerMatch
            AllowOverride All
        </Directory>
      DocumentRoot /Users/yourUserName/Sites/MyWebSite
      ServerName MyWebSite.local
    </VirtualHost>
    {% endcodeblock %}

- Edit the /etc/hosts file and add this at the top:

    {% codeblock console %}
    127.0.0.1 MyWebSite.local
    {% endcodeblock %}

- Make a Symlink to link your Code directory to one in the Sites directory.

    {% codeblock console %}
    ln -s ~/Code/MyWebSite ~/Sites/MyWebSite
    {% endcodeblock %}

- Run apachectl to check config file syntax.

    {% codeblock console %}
    sudo apachectl -t
    {% endcodeblock %}

- Restart Apache

    {% codeblock console %}
    sudo apachectl restart
    {% endcodeblock %}

- Open a browser and go to MyWebSite.local and you should see the results.