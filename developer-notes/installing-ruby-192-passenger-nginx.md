---
title: Get a local copy of this site running
layout: wikistyle
---

# Installing Ruby 1.9.2-p302, Passenger 3.0.9, and Nginx 1.0.6 on a Development Server.

This work was done to enable deployment of the rails3.0 branch of the rails-portal but can easily be extended to support additional Rails or Rack based deployments using Passenger, Ruby 1.9.2 and Nginx.

This pattern could easily be used on other servers already running Apache, Passenger, and Ruby 1.8.7.

## Create a deploy3 user and copy the authorized_keys from the existing deploy user:

    sudo adduser deploy3 --create-home --gid deploy --groups users
    sudo cp /home/deploy/.ssh/authorized_keys  /home/deploy3/.ssh/

## Check the new user:

    $ id deploy3
    uid=2930(deploy3) gid=2929(deploy) groups=2929(deploy),100(users)

## Building and Installing Ruby 1.9.2-p302

### Prerequisites for compiling Ruby 1.9.x:

#### Autoconf >= 2.60 

Autoconf >= 2.60 is needed for compiling Ruby from a git source checkout. I've got autoconf-2.68 installed here: /home/sbannasch/bin/autoconf

#### libyaml

Needed for for using the Psych YAML engine

    sudo yum install libyaml-devel.i386  --enablerepo=epel

#### libffi

To support ruby-ffi.

    sudo yum install libffi-devel --enablerepo=rpmforge

### Compile and install Ruby 1.9.2-p302 to: home/deploy3/ruby/builds/1_9_2p302.

    cd /home/sbannasch/src/ruby-git
    git checkout 493f8a1ce
    ~/bin/autoconf
    ./configure --prefix=/home/deploy3/ruby/builds/1_9_2p302
    make clean
    make
    sudo make install

### Make the installed Ruby accessible to other users:

    sudo chown deploy3:users /home/deploy3
    sudo chmod g+rx /home/deploy3

    sudo chown -R deploy3:users /home/deploy3/ruby
    sudo chmod -R g+rx /home/deploy3/ruby

Another user can use this Ruby by adding this to their PATH:

    $ export PATH=/home/deploy3/ruby/builds/1_9_2p302/bin:$PATH

## Switch to the deploy3 user:

    sudo -- su --login deploy3

## Add the following to /home/deploy3/.bash_profile so deploy3 automatically uses Ruby 1.9.2-p302

    PATH=/home/deploy3/ruby/builds/1_9_2p302/bin:$PATH:$HOME/bin
    export PATH

## Logout and log back into deploy3 and confirm that Ruby 1.9.2-p302 is available:

    [deploy3@otto ~]$ ruby -v
    ruby 1.9.2p302 (2011-08-11 revision 32916) [i686-linux]

## Install the Ruby Gems bundler, passenger, and ruby-debug19

    gem install bundler passenger
    gem install ruby-debug19 -- --with-ruby-include=/home/sbannasch/src/ruby-git

*Currently installing ruby-debug in 1.9.2-p302 requires adding the path to the source checkout for that specific Ruby. If you are using a git source checkout make sure you have the commit 493f8a1ce checked out when installing ruby-debug18.*

## Install the passenger nginx module and nginx:

    $ passenger-install-nginx-module

Tell the interactive installer to install nginx here: `/home/deploy3/nginx`

## Enable SSH user environments:

When running commands as the deploy3 user via SSH I also had to enable this in `/etc/ssh/sshd_config` so the deploy3 user uses the PATH to Ruby 1.9.2-p302 exported in `/home/deploy3/.bash_profile`:

    PermitUserEnvironment yes

and restart sshd

    sudo service sshd restart

## Create a capistrano deploy for xproject-dev here: /web/xproject3.dev.concord.org. 

## Deploy the application using Nginx and Passenger on port 8008

Apache is already listening to port 80 and is configured to use Passenger with Ruby 1.8.7 so configure Nginx to use port 8008.

Modify the nginx configuration to respond at port 8008 to serve this capistrano deploy and tell passenger to use the `deploy3` user. 

{% highlight nginx %}
server {
    listen       8008;
    server_name  localhost;

    #charset koi8-r;

    #access_log  logs/host.access.log  main;

    location / {
        root   html;

        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}

server {
   listen 8008;
   server_name xproject3.dev.concord.org;
   root /web/xproject3.dev.concord.org/current/public;
   passenger_enabled on;
   passenger_user deploy3;
}
{% endhighlight %}

*See: [nginx.conf](nginx-files/nginx.conf.html)*

## Create an nginx service script in `/etc/init.d/nginx` for starting and stopping nginx. 

See: [nginx](nginx-files/nginx.html)

## After creating the nginx script confirm that it works:

    $ sudo /etc/init.d/nginx stop
    Stopping nginx:                                            [  OK  ]

    $ curl -I http://localhost:8008/
    curl: (7) couldn't connect to host

    $ sudo /etc/init.d/nginx start
    Starting nginx:                                            [  OK  ]

    $ curl -I http://localhost:8008/
    HTTP/1.1 200 OK
    Server: nginx/1.0.6
    Date: Mon, 26 Sep 2011 19:07:09 GMT
    Content-Type: text/html
    Content-Length: 151
    Last-Modified: Mon, 26 Sep 2011 15:10:31 GMT
    Connection: keep-alive
    Accept-Ranges: bytes

## Startup nginx when the server starts

Make sure the nginx startup script run levels correspond to the active run level for the server:

    $ who -r
             run-level 3  2011-04-20 21:00                   last=S

    $ sudo /sbin/chkconfig --list nginx
    nginx          	0:off	1:off	2:on	3:on	4:on	5:on	6:off

Tell the server to start nginx when starting up:

    $ sudo /sbin/chkconfig nginx on

## Setup Apache to act as a reverse proxy for this nginx application:

    mkdir -p /web/xproject3.dev.concord.org/apache-proxy-logs
    sudo chown deploy3:users /web/xproject3.dev.concord.org/apache-proxy-logs
    sudo chmod 775 /web/xproject3.dev.concord.org/apache-proxy-logs

## Add an apache vhost, check the apache configuration, and restart apache when all is well:

{% highlight apache %}
# rails 3 version of the cross project portal running in nginx
<VirtualHost *:80>
  ServerName xproject3.dev.concord.org
  ServerAdmin scytacki@concord.org
  ErrorLog  "/web/xproject3.dev.concord.org/apache-proxy-logs/error.log"
  CustomLog "/web/xproject3.dev.concord.org/apache-proxy-logs/access_log" combined
  ProxyPass / http://xproject3.dev.concord.org:8008/
  ProxyPassReverse / http://xproject3.dev.concord.org:8008/
  ProxyTimeout 1200
</VirtualHost>
{% endhighlight %}
