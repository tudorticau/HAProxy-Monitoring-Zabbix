HAProxy Zabbix Discovery and Template
=====================================

[Zabbix](http://zabbix.com) is a powerful open-source monitoring platform, capable of monitoring anything and everything, with the right configuration.
Zabbix's powerful Discovery capability is awesome, making it possible to automatically register hosts as they come online or monitor database servers without having to add individual databases and tables one by one.
This project contains everything you need to discover and monitor HAProxy frontends, backends and backend servers.

[HAProxy](http://www.haproxy.org/) is an awesome multi-purpose load-balancer.

> HAProxy is an open source solution for load balancing and reverse proxying both TCP and HTTP requests—and, in keeping with the abbreviation in its name, it is high availability.
> Like other load balancers or proxies, HAProxy is very flexible and largely protocol-agnostic—it can handle anything sent over TCP

### Latest / Changelog

* [04/26/2020]: created template for Zabbix v4

### Prerequisites

* Zabbix Server >= 4.x (tested on 4)
* HAProxy >= 2.0 (tested on 2.1.3 and 2.2)
* Socat (when using sockets) or nc (when accessing haproxy status via tcp connection)

# Integration logic
* This solution will provide you with a comprehensive template, a lot of controls without loading the HAProxy server with requests. Monitoring process will be done in next way:
* a.	Script haproxy_discovery.sh  will be executed by zabbix and will discovery all the Frontends, Backends and servers under those backends. All data will be returned in JSON format to Zabbix. For reading data zabbix will use data from socket file /var/lib/haproxy/stats
* b.	The script haproxy_stats.sh will extract ALL data from the socket /var/lib/haproxy/stats and will write them in file /var/tmp/haproxy_stats.cache. Script will take care that data from this file to not be older than 59 sec, if more – the file will be rewrited with the new info from socket.
* c.	The script haproxy_stats will be executed with parameters $1, $2, $3, $4. 
* a.	$1 – will provide path to the socket
* b.	$2 – name for the backend
* c.	$3 – name for backend server according to haproxy.cfg
* d.	$4 – the statistic key that we need (specified in item key)
* An example:
* The string: haproxy_stats.sh $1 $2 $3 $4, where $4 is key status will be executed as:
* haproxy_stats.sh /var/lib/haproxy/stats www-backend www01 status – will get status info for the backend www-backend unsing socket file /var/lib/haproxy/stats


### Instructions

* Place `userparameter_haproxy.conf` into `/etc/zabbix/zabbix_agentd.d/` directory, assuming you have Include set in `zabbix_agend.conf`, like so:
```
### Option: Include
# You may include individual files or all files in a directory in the configuration file.
# Installing Zabbix will create include directory in /usr/local/etc, unless modified during the compile time.
#
# Mandatory: no
# Default:
Include=/etc/zabbix/zabbix_agentd.d/
```
* Place `haproxy_discovery.sh` into `/etc/zabbix/scripts/` directory and make sure it's executable (`sudo chmod +x /etc/zabbix/scripts/haproxy_discovery.sh`)
* Place `haproxy_stats.sh` into `/etc/zabbix/scripts/` directory and make sure it's executable (`sudo chmod +x /etc/zabbix/scripts/haproxy_stats.sh`)
* Import `haproxy_v4_template.xml`  template via Zabbix Web UI interface (provided by `zabbix-frontend-php` package)
* Restart zabbix agent to load new config: systemctl restart zabbix-agent

```

```
* Configure HAProxy control socket
* Configure HAProxy to listen on `/var/lib/haproxy/stats` by adding/configuring in HAProxy configuration file (/etc/haproxy/haproxy.cfg):
* global
*    default usage, through socket
*   stats socket /var/lib/haproxy/stats  mode 666 level admin
* Reload HAProxy conf restarting the HAProxy or reload: service haproxy restart
* Test if HAProxy user can read from socket
* 1.	Install socat and nc utilities: yum install nc socat –yum
* 2.	Test: sudo -uhaproxy echo "show info;show stat" | socat stdio unix-connect:/var/lib/haproxy/stats
* We must get data
```

```

>**MAKE SURE TO HAVE APPROPRIATE PERMISSIONS ON HAPROXY SOCKET**
>You can specify what permissions a stats socket file will be created with in `haproxy.cfg`. When using non-admin socket for stats, it's _mostly_ safe to allow very loose permissions (0666).
>You can even use something more restrictive like 0660, as long as you add Zabbix Agent's running user (usually "zabbix") to the HAProxy group (usually "haproxy").
>This way you don't have to prepend `socat` with `sudo` in `userparameter_haproxy.conf` to make sure Zabbix Agent can access the socket. And you don't have to create `/etc/sudoers` entry for Zabbix. And don't need to remember to make it restrictive, avoiding all implication of misconfiguring use of SUDO.
>The symptom of permissions problem on the socket is the following error from Zabbix Agent:
>`Received value [] is not suitable for value type [Numeric (unsigned)] and data type [Decimal]`

* Verify on server with HAProxy installed:
```
root@haproxy1:~$ sudo zabbix_agentd -t haproxy.list.discovery[FRONTEND]
  haproxy.list.discovery[FRONTEND]              [t|{"data":[{"{#FRONTEND_NAME}":"http-frontend"},{"{#FRONTEND_NAME}":"https-frontend"}]}]

root@haproxy1:~$ sudo zabbix_agentd -t haproxy.list.discovery[BACKEND]
  haproxy.list.discovery[BACKEND]               [t|{"data":[{"{#BACKEND_NAME}":"www-backend"},{"{#BACKEND_NAME}":"api-backend"}]}]

root@haproxy1:~$ sudo zabbix_agentd -t haproxy.list.discovery[SERVERS]
  haproxy.list.discovery[SERVERS]               [t|{"data":[{"{#BACKEND_NAME}":"www-backend","{#SERVER_NAME}":"www01"},{"{#BACKEND_NAME}":"www-backend","{#SERVER_NAME}":"www02"},{"{#BACKEND_NAME}":"www-backend","{#SERVER_NAME}":"www03"},{"{#BACKEND_NAME}":"api-backend","{#SERVER_NAME}":"api01"},{"{#BACKEND_NAME}":"api-backend","{#SERVER_NAME}":"api02"},{"{#BACKEND_NAME}":"api-backend","{#SERVER_NAME}":"api03"}]}]
```

* Add hosts with HAProxy installed to just imported Zabbix HAProxy template.
* Wait for discovery.. Frontend(s), Backend(s) and Server(s) should show up under Host Items.



### Troubleshooting

#### Security
SELinux might be preventing the check from using the socket.
```
# check your audit logs
$ tail -f /var/log/audit/audit.log

# get more details
$ tail -n100 -f /var/log/audit/audit.log | audit2why

# look for haproxy related messages
$ tail -n100 -f /var/log/audit/audit.log | grep haproxy | audit2why
```

You can try temporarily disabling SELinux, while testing. It's up to you to re-enable it.
```
# disable SELinux, make sure to re-enable it, if you are relying on SELinux
$ setenforce 0
```

#### Discover
```
/etc/zabbix/scripts/haproxy_discovery.sh $1 $2
$1 is a path to haproxy socket
$2 is FRONTEND or BACKEND or SERVERS

# /etc/zabbix/scripts/haproxy_discovery.sh /var/lib/haproxy/stats FRONTEND    # second argument is optional
# /etc/zabbix/scripts/haproxy_discovery.sh /var/lib/haproxy/stats BACKEND     # second argument is optional
# /etc/zabbix/scripts/haproxy_discovery.sh /var/lib/haproxy/stats SERVERS     # second argument is optional
```

#### haproxy_stats.sh script
```
## Usage: haproxy_stats.sh $1 $2 $3 $4
### $1 is a path to haproxy socket - optional, defaults to /var/lib/haproxy/stats
### $2 is a name of the backend, as set in haproxy.cfg
### $3 is a name of the server, as set in haproxy.cfg
### $4 is a stat as references by HAProxy terminology
# haproxy_stats.sh /var/lib/haproxy/stats www-backend www01 status
# haproxy_stats.sh www-backend BACKEND status
# haproxy_stats.sh https-frontend FRONTEND status
```


#### Stats
```
## Bytes In:      echo "show stat" | socat $1 stdio | grep "^$2,$3" | cut -d, -f9
## Bytes Out:     echo "show stat" | socat $1 stdio | grep "^$2,$3" | cut -d, -f10
## Session Rate:  echo "show stat" | socat $1 stdio | grep "^$2,$3" | cut -d, -f5
### $1 is a path to haproxy socket
### $2 is a name of the backend, as set in haproxy.cfg
### $3 is a name of the server, as set in haproxy.cfg
# echo "show stat" | socat /var/lib/haproxy/stats stdio | grep "^www-backend,www01" | cut -d, -f9
# echo "show stat" | socat /var/lib/haproxy/stats stdio | grep "^www-backend,BACKEND" | cut -d, -f10
# echo "show stat" | socat /var/lib/haproxy/stats stdio | grep "^https-frontend,FRONTEND" | cut -d, -f5
# echo "show stat" | socat /var/lib/haproxy/stats stdio | grep "^api-backend,api02" | cut -d, -f18 | cut -d\  -f1
```

#### More
Take a look at the out put of the following to learn more about what is available though HAProxy socket
```
echo "show stat" | socat /var/lib/haproxy/stats stdio
```

