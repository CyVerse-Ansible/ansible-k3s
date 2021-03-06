CyVerse Ansible k3s
===================

This role will create a k3s standalone or cluster. 

Requirements
------------

If using Docker with k3s, then this role will depend on Docker already installed or a role that provides it.

This role will setup a firewall (ufw) and by default allow all nodes within the k3s cluster to communicate with each other.

This role makes assumptions in your host files. The following is an example inventory file with k3s-masters, k3s-agents, and k3s-cluster defined, which is the minimum declaration.

Note: At this time, only 1 master is supported

In .ini format
````
[k3s-masters]
  w.x.y.z
    
[k3s-agents]
  a.b.c.d
  e.f.g.h

[k3s-cluster:children]
  k3s-masters
  k3s-agents
````
In yaml format
````
all:
  hosts:
    k1:
      ansible_host: w.x.y.z
      ansible_user: root
    k2:
      ansible_host: a.b.c.d
      ansible_user: root
    k3:
      ansible_host: e.f.g.h
      ansible_user: root
  children:
    k3s-masters:
      hosts:
        k1:
    k3s-agents:
      hosts:
        k2:
        k3:
    k3s-cluster:
      children:
        k3s-masters:
        k3s-agents:
````

Role Variables
--------------

The following table lists optional ansible variables along with the default values if not defined.

Variable Name | Default value if not defined | Description
------------- | ---------------------- | -----------
K3S_DOCKER_ENABLE | true | enables the docker engine
K3S_TRAEFIK_ENABLE | false | disable traefik ingress
K3S_MASTER_INSTALL | true | reinstall master node(s)
K3S_MASTER_PORT | 6443 | master node port
K3S_POSTGRESQL_ENABLE | false | enables the use of postgresql
K3S_POSTGRESQL_INSTALL | false | enables installation of postgresql on the first k3s master; K3S_POSTGRESQL_ENABLE must be true
K3S_POSTGRESQL_HOST | 127.0.0.1 | host name or ip setting for postgresql db, from the k3s master configuration
K3S_POSTGRESQL_PORT | 5432 | port for postgresql db
K3S_POSTGRESQL_DB   | kubernetes | postgres database name
K3S_POSTGRESQL_USER | k3suser | db username to K3S_POSTGRESQL_DB
K3S_POSTGRESQL_PASS | randomly generated | password to use for K3S_POSTGRESQL_USER to access K3S_POSTGRESQL_DB; stored in /opt/k3s after being generated
K3S_FIREWALL_ADD_PORTS | none | This is an array of dictionaries (see example playbook for examples); each element should have port, rule, proto, and src

Example Playbook
----------------

This is a sample playbook:
````
- hosts: k3s-cluster
  become: true
  roles:
    - k3s
  vars:
    K3S_FORCE_UNINSTALL: true
    K3S_POSTGRESQL_ENABLE: true
    K3S_POSTGRESQL_INSTALL: true
    K3S_FIREWALL_ADD_PORTS:
      - port: "8888"
        rule: "allow"
        proto: "tcp"
        src:   "1.2.3.0/24"
      - port: "443"
        rule: "deny"
        proto: "tcp"
        src:   "any"
````

Author Information
------------------
Edwin Skidmore (edwin@cyverse.org) 
