# This repository has been archived.

[![Build Status](https://secure.travis-ci.org/dcrosta/cookbook-simple-iptables.png?branch=master)](http://travis-ci.org/dcrosta/cookbook-simple-iptables)

Description
===========

Simple cookbook with LWRPs for managing iptables rules and policies.

Requirements
============

None, other than a system that supports iptables.


Platforms
=========

The following platforms are supported and known to work:

* Debian (6.0 and later)
* RedHat (5.8 and later)
* CentOS (5.8 and later)

Other platforms that support `iptables` and the `iptables-restore` script
are likely to work as well; if you use one, please let me know so that I can
update the supported platforms list.

Attributes
==========

This cookbook uses node attributes to track internal state when generating
the iptables rules and policies. These attributes _should not_ be overridden
by roles, other recipes, etc.

Usage
=====

Include the recipe `simple_iptables` somewhere in your run list, then use
the LWRPs `simple_iptables_rule` and `simple_iptables_policy` in your
recipes.

`simple_iptables_rule` Resource
-------------------------------

Defines a single iptables rule, composed of a rule string (passed as-is to
iptables), and a jump target. The name attribute defines an iptables chain
that this rule will live in (and, thus, that other rules can jump to). For
instance:

    # Allow SSH
    simple_iptables_rule "ssh" do
      rule "--proto tcp --dport 22"
      jump "ACCEPT"
    end

For convenience, you may also specify an array of rule strings in a single
LWRP invocation:

    # Allow HTTP, HTTPS
    simple_iptables_rule "http" do
      rule [ "--proto tcp --dport 80",
             "--proto tcp --dport 443" ]
      jump "ACCEPT"
    end

Additionally, if you want to declare a module (such as log) you can define jump as false:

    # Log
    simple_iptables_rule "system" do
      rule "--match limit --limit 5/min --jump LOG --log-prefix \"iptables denied: \" --log-level 7"
      jump false
    end

By default rules are added to the filter table but the nat and mangle tables are also supported. For example:

    # Tomcat redirects
    simple_iptables_rule "tomcat" do
      table "nat"
      direction "PREROUTING"
      rule [ "--protocol tcp --dport 80 --jump REDIRECT --to-port 8080",
             "--protocol tcp --dport 443 --jump REDIRECT --to-port 8443" ]
      jump false
    end

    #mangle example
    #NOTE: set jump to false since iptables expects the -j MARK --set-mark in that order
    simple_iptables_rule "mangle" do
      table "mangle"
      direction "PREROUTING"
      jump false
      rule "-i eth0 -j MARK --set-mark 0x6
    end

    #reject all outbound connections attempts to 10/8 on a dual-homed host
    simple_iptables_rule "reset_10slash8_outbound" do
      direction "OUTPUT"
      jump false
      rule "-p tcp -o eth0 -d 10/8 --jump REJECT --reject-with tcp-reset"
    end

`simple_iptables_policy` Resource
---------------------------------

Defines a default action for a given iptables chain. This is usually used to
switch from a default-accept policy to a default-reject policy. For
instance:

    # Reject packets other than those explicitly allowed
    simple_iptables_policy "INPUT" do
      policy "DROP"
    end

As with the `simple_iptables_rules` resource, policies are applied to the filter table
by default. You may change the target table to nat as follows:

    # Reject packets other than those explicitly allowed
    simple_iptables_policy "INPUT" do
      table "nat"
      policy "DROP"
    end

Example
=======

Suppose you had the following `simple_iptables` configuration:

    # Reject packets other than those explicitly allowed
    simple_iptables_policy "INPUT" do
      policy "DROP"
    end
    
    # The following rules define a "system" chain; chains
    # are used as a convenient way of grouping rules together,
    # for logical organization.
    
    # Allow all traffic on the loopback device
    simple_iptables_rule "system" do
      rule "--in-interface lo"
      jump "ACCEPT"
    end
    
    # Allow any established connections to continue, even
    # if they would be in violation of other rules.
    simple_iptables_rule "system" do
      rule "-m conntrack --ctstate ESTABLISHED,RELATED"
      jump "ACCEPT"
    end
    
    # Allow SSH
    simple_iptables_rule "system" do
      rule "--proto tcp --dport 22"
      jump "ACCEPT"
    end
    
    # Allow HTTP, HTTPS
    simple_iptables_rule "http" do
      rule [ "--proto tcp --dport 80",
             "--proto tcp --dport 443" ]
      jump "ACCEPT"
    end
    
    # Tomcat redirects
    simple_iptables_rule "tomcat" do
      table "nat"
      direction "PREROUTING"
      rule [ "--protocol tcp --dport 80 --jump REDIRECT --to-port 8080",
             "--protocol tcp --dport 443 --jump REDIRECT --to-port 8443" ]
      jump false
    end

This would generate a file `/etc/iptables-rules` with the contents:

    # This file generated by Chef. Changes will be overwritten.
    *nat
    :PREROUTING ACCEPT [0:0]
    :INPUT ACCEPT [0:0]
    :OUTPUT ACCEPT [0:0]
    :POSTROUTING ACCEPT [0:0]
    :tomcat - [0:0]
    -A PREROUTING --jump tomcat
    -A tomcat --protocol tcp --dport 80 --jump REDIRECT --to-port 8080
    -A tomcat --protocol tcp --dport 443 --jump REDIRECT --to-port 8443
    COMMIT
    # Completed
    # This file generated by Chef. Changes will be overwritten.
    *filter
    :INPUT DROP [0:0]
    :FORWARD ACCEPT [0:0]
    :OUTPUT ACCEPT [0:0]
    :system - [0:0]
    :http - [0:0]
    -A INPUT --jump system
    -A system --in-interface lo --jump ACCEPT
    -A system -m conntrack --ctstate ESTABLISHED,RELATED --jump ACCEPT
    -A system --proto tcp --dport 22 --jump ACCEPT
    -A INPUT --jump http
    -A http --proto tcp --dport 80 --jump ACCEPT
    -A http --proto tcp --dport 443 --jump ACCEPT
    COMMIT
    # Completed

Which results in the following iptables configuration:

    # iptables -L
    Chain INPUT (policy DROP)
    target     prot opt source               destination         
    system     all  --  anywhere             anywhere            
    http       all  --  anywhere             anywhere            
    
    Chain FORWARD (policy ACCEPT)
    target     prot opt source               destination         
    
    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination         
    
    Chain http (1 references)
    target     prot opt source               destination         
    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:http
    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:https
    
    Chain system (1 references)
    target     prot opt source               destination         
    ACCEPT     all  --  anywhere             anywhere            
    ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
    ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:ssh

    #iptables -L -t nat
    Chain PREROUTING (policy ACCEPT)
    target     prot opt source               destination         
    tomcat     all  --  anywhere             anywhere            
    
    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination         
    
    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination         
    
    Chain POSTROUTING (policy ACCEPT)
    target     prot opt source               destination         
    
    Chain tomcat (1 references)
    target     prot opt source               destination         
    REDIRECT   tcp  --  anywhere             anywhere             tcp dpt:http redir ports 8080
    REDIRECT   tcp  --  anywhere             anywhere             tcp dpt:https redir ports 8443

Changes
=======

* 0.4.0 (May 9, 2013)
    * Added support for mangle table (#? - Michael Hart)
    * Updated Gemfile to 11.4.4 (#? - Michael Hart)
* 0.3.0 (March 5, 2013)
    * Added support for nat table (#10 - Nathan Mische)
    * Updated Gemfile for Travis-CI integration (#10 - Nathan Mische)
* 0.2.4 (Feb 13, 2013)
    * Fixed attribute precedence issues in Chef 11 (#9 - Warwick Poole)
    * Added `name` to metadata to satisfy recent foodcritic versions
* 0.2.3 (Nov 10, 2012)
    * Fixed a warning in Chef 11+ (#7 - Hector Castro)
* 0.2.2 (Oct 13, 2012)
    * Added support for logging module and other non-jump rules (#6 - phoolish)
* 0.2.1 (Aug 5, 2012)
    * Fixed a bug using `simple_iptables` with chef-solo (#5)
* 0.2.0 (Aug 1, 2012)
    * Allow an array of rules in `simple_iptables_rule` LWRP (Johannes Becker)
    * RedHat/CentOS compatibility (David Stainton)
    * Failing `simple_iptables_rule`s now fail with a more helpful error message
* 0.1.2 (July 24, 2012)
    * Fixed examples in README (SchraderMJ11)
* 0.1.1 (May 22, 2012)
    * Added Travis-CI integration (Nathen Harvey)
    * Fixed foodcritic warnings (Nathen Harvey)
* 0.1.0 (May 12, 2012)
    * Initial release

