# fail2mtblock

FUNCTION

This script will parse secure/auth logs from local (or remote) system and
based on fail login attempts will:
- create unwanted IP list,
- download already blocked list from MikroTik, 
- diff both lists
- add new unwanted IP to Mikrotik block list

Naturally initial run of this script will require some time and resources
to parse, whoever each next one with addition of logrotate will require 
minimum amount of effort to finish.

Additionally because blocking is maintained by MikroTik router all servers
are protected.


BACKGROUND

In many cases simple fail2ban-like application will do just fine, whoever
some small or even home made environments based on ARM boards can choke 
on parsing large amount of data due to lack of CPU power or I/O 

This problem happened to me while ago, when i was using raspberry pi
as my main home server with public services.

Back then I've had to deal with hundred upon thousands login attempts.

Beside resource issues usual iptables blocking scheme wasn't enough because
doping packet was solution only for one particular IP, and was useless for
large boot-nets.

Whoever problem wasn't in amount of hosts but visibility of my services.
By simply dropping connections boot-net still knew that there is some
service to probe and it is just matter of time until one of zombie will
find something.

Solution for that was using simple TCP mechanism of RST reply, which is
standard server reply that mean "There is no service here, go away"

Also we can use UDP mechanism of ICMP reset host unreachable, which is
standard router reply that mean "There is no host here, go away"

And because boot-nets tends to be surprisingly well written and optimal, 
after some RST/ICMP messages are just stopping probing.




INSTALL PREPERATION
+ Large amount of addreses require use of ssh-key for connecting,
  [best way is to follow-up official MikroTik wiki for that](http://wiki.mikrotik.com/wiki/Use_SSH_to_execute_commands_(DSA_key_login))

+ To prevent self-block, exceptions of block rules are needed on top of filtering

```
/ip firewall filter
add action=accept chain=input comment="accept exceptions" src-address-list=_exceptions
add action=accept chain=forward comment="accept exceptions" src-address-list=_exceptions
```

+ Desired IP of excluded hosts should kept as _exceptions list

```
/ip firewall address-list
add address=192.168.88.0/24 comment="exclude from blocking" list=_exceptions
add address=some_ip_address comment="exclude from blocking" list=_exceptions
```

- To block traffic pre-configured rules are required at bottom of MikroTik filtering that are use lists

```
/ip firewall filter
add action=reject chain=input comment="reject tcp reset" protocol=tcp reject-with=tcp-reset src-address-list=_block
add action=reject chain=input comment="icmp reset host unreachable" protocol=udp reject-with=icmp-host-unreachable src-address-list=_block
add action=reject chain=forward comment="icmp reset host unreachable" protocol=udp reject-with=icmp-host-unreachable src-address-list=_block
add action=reject chain=forward comment="reject tcp reset" protocol=tcp reject-with=tcp-reset src-address-list=_block
add action=reject chain=forward comment="reject urg flag" protocol=tcp reject-with=tcp-reset tcp-flags=urg
add action=drop chain=forward comment="reject invalid" connection-state=invalid
```
