---
layout:         post
title:          "How to force Docker not to bypass the UFW rules on Ubuntu 16.04"
date:           2017-09-05 12:00:00
description:    "See how you can make the Docker obey the rules of UFW on Ubuntu 16.04 (and others) with simple iptables modification."
categories:     docker ufw ubuntu firewall rules iptables network
---

It is not easy to understand what's beneath the mighty Docker everyone has been talking about for the last couple of years with growing fascination. So what is the fuss all about? Well, the fact is that Docker is kind of a revolutionary thing in IT. It might not be the one and only solution to the world's most pressing problems, but it still provides a better way of preparing and deploying applications around the globe. Everyone is trying their hands at it, but the struggle starts where the majority give up on exploring it, assuming that they know everything they would ever need. 

That's the point where people like us come to the game, trying to understand something more than just `docker inspect` which is often seen as something bordering on black magic. So let's just go a bit deeper and face one of the most commonly occurring problems I tried to solve a few months ago as well. If you have ever tried to make the Docker work with the UFW, then you probably know what's the said struggle. Let's examine it!


## Use case

### Setting up test environment

Here I'm going to use Vagrant as my Ubuntu 16.04 server, but you might use whatever you want, we just start together with the plain Ubuntu Xenial x64.

Let's create a `Vagrantfile` with the following content:

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.provision "shell", inline: <<-SHELL
    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    apt-get install -y docker-ce
    groupadd docker
    usermod -aG docker $(whoami)
  SHELL
end
{% endhighlight %}

What it provides is just the Ubuntu 16.04 with some basic packages and the Docker CE itself. The UFW we are about to use is already there, as it is being shipped with each Ubuntu version at the moment.

Run `sudo su` so we can easily omit `sudo` part for every command below.

Let's define some basic rules for the UFW and enable it.

{% highlight shell %}
$ ufw default deny incoming
$ ufw default allow outgoing
$ ufw allow ssh
$ ufw enable
{% endhighlight %}

and run an example Nginx container that exposes port 80 by default. With networking set as `host` the Docker will publish the port directly to the main interface without creating additional interfaces called `bridge`s.

{% highlight shell %}
$ docker run -it --net=host nginx
{% endhighlight %}

If you take a closer look at the `Vagrantfile` we defined above, you will see that the machine we're using has the IP of `192.168.33.10`. It's pretty convenient to have this set up explicitly in the configuration. Let's access this IP with our browser. Open it and access `192.168.33.10`.

What's the result? Well... Nothing. The UFW has blocked the traffic on port 80 because its default policy is to block incoming traffic. Nothing shocking. But what if we ran the Nginx container using port mapping and with the networking mode being set up as bridge?


### The unexpected and unwritten behavior

{% highlight shell %}
$ docker run -it -p 8080:80 nginx
{% endhighlight %}

and try it in your browser with `192.168.33.10:8080`. Please notice that we're using `8080` port here explicitly, so we can easily tell the difference within the incoming parts. Do you see the default response with `Welcome to nginx!`? That's because Docker has bypassed the UFW settings of the default block policy. And we definitely don't want that to be happening anymore, so let's dig a bit and try to understand what's actually being added when we're running any container.

{% highlight shell %}
$ iptables -L -n -t nat
{% endhighlight %}

gives us a lot in return, but the most important part for us is this one:

{% highlight shell %}
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0           
MASQUERADE  tcp  --  172.17.0.2           172.17.0.2           tcp dpt:80

Chain DOCKER (2 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:80
{% endhighlight %}

What can be seen here is `POSTROUTING chain` with `MASQUERADE` which maps and exposes the Nginx default port for our Nginx container with the specific IP in range of the CIDR block for our Docker. How do I know that? Use {% raw %}`docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' CONTAINER_ID_HERE`{% endraw %}. This outputs `172.17.0.2` in my case.

Then there's a second part of the `DOCKER chain`, which uses the `DNAT` to publish our `172.17.0.2:80` that stands for the Nginx container to `0.0.0.0:8080` implicitly. That is why we can still access the welcome page in spite of having the UFW enabled with the default block policy for the incoming traffic.

What if we modified our container command and explicitly defined the IP we want it to bind to?

{% highlight shell %}
$ docker run -d -p 127.0.0.1:8080:80 nginx
{% endhighlight %}

and for `$ iptables -L -n -t nat` we get:

{% highlight shell %}
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0           
MASQUERADE  tcp  --  172.17.0.2           172.17.0.2           tcp dpt:80

Chain DOCKER (2 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
DNAT       tcp  --  0.0.0.0/0            127.0.0.1            tcp dpt:8080 to:172.17.0.2:80
{% endhighlight %}

Do you see the difference? Destination for the `DNAT` rule in the `DOCKER chain` instead of giving us `0.0.0.0` returns `127.0.0.1` which binds to the exactly one interface on our machine. If you test it in your browser now, you won't access any welcome page but you would still be able to access it on the instance with simple `curl` command. It doesn't solve the issue, because we're still not able to publish anything to the world, so let's modify the `127.0.0.1` with the IP address of the instance we're doing all this black magic for?

{% highlight shell %}
$ docker run -d -p 192.168.33.10:8080:80 nginx
{% endhighlight %}

The command `$ iptables -L -n -t nat` shows nothing more than what we expected, because we’ve exposed the container directly to the public interface everyone is accessing with a simple browser request. Can we as well? For sure. Website is about to be served as you would expect and there's still a way to block the traffic easily with the UFW, because the Uncomplicated Firewall controls the traffic on the level of this public interface too.
But what if this does not satisfy us and we still want to get to the bottom of it? You may not want to define the IP explicitly. In terms of cloud scalability for example, that might be a pain. So how do we proceed?


### The solution

Let's start with setting up our Docker to the mode where it doesn't modify iptables rules:

{% highlight shell %}
$ echo "{
\"iptables\": false
}" > /etc/docker/daemon.json
{% endhighlight %}

Now we need to restart our instance. Exit the instance and run `$ vagrant reload && vagrant ssh`. So what do we get with `$ iptables -L -n -t nat`? Basically, nothing. Rules have been removed and Docker is kind of isolated right now. But in a negative sense. With `$ docker run -d -p 8080:80 nginx` you can still access the welcome page on your browser, but if you try simple `$ docker exec -it CONTAINER_ID apt-get update` on any running container, you will immediately see that there is no connection to the Internet. That's an undesirable behavior.

So if we fix this issue, do we have the setup we wanted from the very beginning? We need to tweak the UFW a bit then with

{% highlight shell %}
$ sed -i -e 's/DEFAULT_FORWARD_POLICY="DROP"/DEFAULT_FORWARD_POLICY="ACCEPT"/g' /etc/default/ufw
$ ufw reload
{% endhighlight %}

This allows the UFW to NAT the connections from the external interface to the internal one.
Then, with a simple assumption that your Docker has the IP of `172.17.0.1` (can be found easily with `ifconfig` for `docker0` interface), we run

{% highlight shell %}
$ iptables -t nat -A POSTROUTING ! -o docker0 -s 172.17.0.0/16 -j MASQUERADE
{% endhighlight %}

and... that's it! From this point on, you can access the Internet within a container and have it covered with UFW rules so there is no access leak. That `iptables` rule is nothing more than just the same rule Docker is adding implicitly with `iptables: true` in `/etc/docker/daemon.json`. So by dismantling this through the whole article, we get to the bottom of it and we find a solution that is both secure, because we have control over the traffic we had not before, and explicit, so we know what's going on under the bonnet.


## Summary

It wasn't easy. We touched the untouched. We named Voldemort out loud. We said `Bloody Mary` three times in front of the mirror. And we survived! That big ol’ whale we all see each day isn't as intimidating as they describe it to be. But it definitely has some drawbacks that can be beaten and that's one of them for sure. Hope I helped you a bit during your way to the core of the unwritten and untold of Docker. See you down there!
