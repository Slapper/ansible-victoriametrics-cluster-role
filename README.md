# Ansible Role: Install VictoriaMetrics in cluster mode

Ansible role for installing or upgrading VictoriaMetrics cluster, inspired by the standalone version : https://github.com/dreamteam-gg/ansible-victoriametrics-role . Thanks to [@dreamteam-gg](https://github.com/dreamteam-gg) 

Tested on `Centos 7` & `Centos 8` but it should work on other distributions with minor adjustments :-)

## Requirements

None.

## Ansible inventory

In order to use this role create your inventory file with these 3 groups :

- vmstorage ( for storage nodes )
- vmselect ( for select nodes )
- vminsert ( for insert nodes )

More info here : https://github.com/VictoriaMetrics/VictoriaMetrics/tree/cluster#architecture-overview

```
[vmstorage]
vmstorage01.example.com
vmstorage02.example.com
vmstorage03.example.com
vmstorage04.example.com

[vminsert]
vminsert01.example.com
vminsert02.example.com

[vmselect]
vmselect01.example.com
vmselect02.example.com
```


## Role Variables

The default variables of the role. You can overwrite them on role or playbook level. 

```
---
## defaults settings for all VictoriaMetrics nodes
victoriametrics_repo_url: "https://github.com/VictoriaMetrics/VictoriaMetrics"
victoriametrics_download_url: "{{ victoriametrics_repo_url }}/releases/download/{{ victoriametrics_version }}/victoria-metrics-{{ victoriametrics_version }}-cluster.tar.gz"
victoriametrics_version: "v1.34.6"
victoriametrics_system_user: "victoria"
victoriametrics_system_group: "victoria"

## variables for vmstorage nodes
victoriametrics_vmstorage_data_dir: "/usr/local/bin/victoria-storage"
victoriametrics_vmstorage_retention_period: "24"
victoriametrics_vmstorage_memory_allowed_percent: "60" # 60 is the default value 
victoriametrics_vmstorage_service_args: "" # Add extra variables here .Found more options withvmstorage-prod --help

## variables for vmselect nodes
victoriametrics_vmselect_cache_dir: "/usr/local/bin/victoria-cache"
victoriametrics_vmselect_memory_allowed_percent: "60"
victoriametrics_vmselect_service_args: "" # Add extra variables here . Found more options with vmselect-prod --help

## variables for vminsert nodes
victoriametrics_vminsert_service_args: "" # Add extra variables here . Found more options with vminsert-prod --help
victoriametrics_vminsert_memory_allowed_percent: "60"
```


## Example Playbook for initial installation

```
---
- hosts: vmstorage,vminsert,vmselect
  become: yes

  roles:
   - ansible-vicotriametrics-cluster-role
```

## Example Playbook for update your cluster

- Before upgrade your cluster always check releases : https://github.com/VictoriaMetrics/VictoriaMetrics/releases 

- For updating your cluster without any issues at least a single node of each type should be up and running. Read more :   https://github.com/VictoriaMetrics/VictoriaMetrics/tree/cluster#cluster-availability

- To update the victoriametrics just update the vars `victoriametrics_version` 

- Need to use Ansible `serial` function in order to update one node at a time. More info : https://docs.ansible.com/ansible/latest/user_guide/playbooks_delegation.html#rolling-update-batch-size

- Also using pre_tasks to gather facts for the the nodes in vmstorage,vmselect,vminsert groups to update j2 templates.

```
---
- hosts: vminsert,vmselect,vmstorage
  become: true
  gather_facts: true

  tasks:
    - name: Gather facts for victoria nodes
      setup:

- hosts: vminsert,vmselect,vmstorage
  gather_facts: false
  become: true
  serial: 1

  vars:
    - victoriametrics_version: "v1.40.0"
```

## Extras 

In order to use VictoriaMetrics you need an http load balancer :-)  

In my case i used Haproxy. Here is a simple config for your vminsert and vmselect nodes. In case you add more vmselect or vminsert nodes don't forget to update your Haproxy config accordingly. 

```
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log          127.0.0.1 local2
    log-send-hostname
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     40000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /run/haproxy.sock mode 666 level admin
    # utilize system-wide crypto-policies
    ssl-default-bind-ciphers PROFILE=SYSTEM
    ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 30000
    balance
#---------------------------------------------------------------------
# main frontend vminsert and vmselect
#---------------------------------------------------------------------

frontend vminsert
bind 0.0.0.0:8480
  mode http
  log global
  default_backend vminsert_nodes

frontend vmselect
bind 0.0.0.0:8481
  mode http
  log global
  default_backend vmselect_nodes


frontend stats
bind 0.0.0.0:10010
    mode http
    stats enable
    stats uri /stats
    stats realm HAProxy\ Statistics
    stats auth admin:admin
    stats admin if TRUE
#---------------------------------------------------------------------
# round robin balancing between vminsert and vmselect
#---------------------------------------------------------------------

backend vminsert_nodes
    mode http
    balance roundrobin
    option httpchk GET /health
    http-check expect string OK
    default-server inter 5s fall 3 rise 2
    server vminsert01 10.10.10.100:8480 check
    server vminsert02 10.10.10.101:8480 check

backend vmselect_nodes
    mode http
    balance roundrobin
    option httpchk GET /health
    http-check expect string OK
    default-server inter 5s fall 3 rise 2
    server vmselect01 10.10.10.102:8481 check
    server vmselect02 10.10.10.103:8481 check
```

### To do
Add haproxy install and configuration functionality on the role. 

### License

BSD

### Author Information

@Mtsa miltsatsakis@gmail.com
