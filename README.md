# fail2mtblock

### FUNCTION

This script will parse secure/auth logs from local (or remote) system and based on fail login attempts will:
- create unwanted IP list,
- download already blocked list from MikroTik, 
- compare both lists,
- add new unwanted IP to Mikrotik block list

Naturally initial run of this script will require some time and resources to parse (which can be avoided, se below), each next run require minimum amount of effort to finish. To best proformance using logrotate is advised.

Additionally because blocking is maintained by MikroTik router all servers are protected.


### BACKGROUND

In many cases simple fail2ban-like application will do just fine, whoever some small or even home made environments based on ARM boards can easly choke  on parsing large amount of data due to lack of CPU power or I/O. 

This problem happened to me while ago, when i was using raspberry pi as my main home server with public services.

Back then I've had to deal with hundred upon thousands login attempts.

Beside resource issues usual iptables blocking scheme wasn't enough because doping packet was solution only for one particular IP, and was useless for large boot-nets.

Whoever problem wasn't in amount of hosts but visibility of my services.

By simply dropping connections boot-net still knew that there is some service to probe and it is just matter of time until one of zombie will find something.

Solution for that was using simple TCP mechanism of RST reply, which is standard server reply that mean

`There is no service here, go away`

Also same trick can be use in UDP mechanism of ICMP reset host unreachable, which is standard router reply that mean

`There is no host here, go away`

And because boot-nets tends to be surprisingly well written and optimal,  after some RST/ICMP messages are just stopping probing.


### INSTALL REQUIRMENTS

+ Large amount of addreses require use of ssh-key for connecting, [best way is to follow-up official MikroTik wiki for that](http://wiki.mikrotik.com/wiki/Use_SSH_to_execute_commands_(DSA_key_login))

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

+ To block traffic pre-configured rules are required at bottom of MikroTik filtering that are use lists

```
/ip firewall filter
add action=reject chain=input comment="reject tcp reset" protocol=tcp reject-with=tcp-reset src-address-list=_block
add action=reject chain=input comment="icmp reset host unreachable" protocol=udp reject-with=icmp-host-unreachable src-address-list=_block
add action=reject chain=forward comment="icmp reset host unreachable" protocol=udp reject-with=icmp-host-unreachable src-address-list=_block
add action=reject chain=forward comment="reject tcp reset" protocol=tcp reject-with=tcp-reset src-address-list=_block
add action=reject chain=forward comment="reject urg flag" protocol=tcp reject-with=tcp-reset tcp-flags=urg
add action=drop chain=forward comment="reject invalid" connection-state=invalid
```

+ My initial block list with over 3k hosts can be easily added by downloading it directly on MikroTik
```
/tool fetch https://raw.githubusercontent.com/urbinek/fail2mtblock/master/initial_block_list.rsc
/import file-name="initial_block_list.rsc"
```

### INSTALL

If above requirements are met, installation can be simply done by 
```
cd /opt/
git clone https://github.com/urbinek/fail2mtblock.git
cd fail2mtblock/
chmod +x fail2mtblock 
```

### CONFIGURATION

For now script needs to be configured manually by defining header section according to comments
```bash
# define MikroTik and ssh-key
mt_firewall="192.168.88.1"		# domain name works as well
mt_login="admin-ssh"			# delegated login name for MikroTik
mt_ssh_port="22"                       # ssh port
mt_ssh_key="/root/.ssh/id_dsa"		# absolute path to ssh-key for MikroTik
mt_block_list="_block"			# name of /ip firewall address-list list

# define files managed by script
#local_auth="/var/log/auth.log"	# auth.log for debian
local_auth="/var/log/secure" 		# secure for cenos/rhel 
local_authfail="/tmp/local_authfail.list"	# filtered $local_auth by fail/invalid 
local_sorted="/tmp/local_sorted.list"	# sorted and simplified $local_authfail
mt_blocked="/tmp/mt_blocked.list"		# currently blocked IP from MikroTik
mt_todo="/tmp/mt_todo.list"		# list of new IP addreses to block
```
### USAGE
 
Script itself can be run manually,
```
/opt/fail2mtblock/fail2mtblock
```

But using cron is more efficient, e.g:
``` 
# Linux 0 tolerance blocking script for MikroTik   
0 */2 * * *       /opt/fail2mtblock/fail2mtblock >> /var/log/fail2mtblock.log 2>&1
```

### OTHER

Example log from working script
```
2016-08-15 11:53:19 : Get failed/invalid attempts from logs...
2016-08-15 11:53:19 : Filter IP addreses from /tmp/local_authfail.list
2016-08-15 11:53:19 : Get current MikroTik block list...
2016-08-15 11:53:28 : Comparing both lists...
2016-08-15 11:53:29 : Adding new IP addreses to block list...
2016-08-15 11:53:30 : added 119.205.99.167...
2016-08-15 11:53:32 : added 162.203.149.43...
2016-08-15 11:53:32 : Done!
```
