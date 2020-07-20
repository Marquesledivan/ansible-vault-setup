# Hashicorp Vault and Consul setup using Ansible Role

We are going to create an Ansile Role for Vault setup so we can reuse it. We will begin by creating a new user account named "vault" and "consul" which will help with a secure setup. We will use this account to isolate the ownership of vault. We don't create any home directory or shell for this user so that user can't log in to a server.

Next, we need to download vault archive from here on our remote vault instance. This will give a zip archive file. To unzip vault archive, we need to install unzip so we can unzip vault archive and takeout needed binary. Once this is done, we need to unzip vault archive, move our vault binary to "/usr/local/bin" and make vault user as the owner of this binary with reading and execute permissions.

We need to set binary capabilities on Linux, to give the Vault executable the ability to use the mlock syscall without running the process as root.

We need to setup systemd init file to manage the persistent vault daemon. We need to set below content into systemd service file. Finally, start the vault server.

Attention to define the variables of Ansible, in these variables are configuring the notes for each environment of the Consul and the safe and it is recommended that the people who are going to change the playbooy have knowledge in the Vault and Consul.

This environment is functioning in a vault cluster and in the Consul.

### Setup

We set up haproxy for Consul and Vault.

Config Haproxy:

```bash

listen vault
    bind {{ ansible_enp0s3.ipv4.address }}:80
    mode http
    balance roundrobin
    option httpchk GET /v1/sys/health
    server vault1 {{ vault1 }}:8200 check fall 3 rise 2
    server vault2 {{ vault2 }}:8200 check fall 3 rise 2
    server vault3 {{ vault3 }}:8200 check fall 3 rise 2

listen consul
    bind {{ ansible_enp0s3.ipv4.address }}:8500
    mode http
    balance roundrobin
    option httpchk GET /ui
    server vault1 {{ consul1 }}:8500 check fall 3 rise 2
    server vault2 {{ consul2 }}:8500 check fall 3 rise 2
    server vault3 {{ consul3 }}:8500 check fall 3 rise 2

```

Config Consul:

```bash
 {
   "server": true,
   "node_name": "{{ ansible_hostname }}",
   "datacenter": "dc1",
   "data_dir": "/var/lib/consul/data",
   "bind_addr": "0.0.0.0",
   "client_addr": "0.0.0.0",
   "advertise_addr": "{{ ansible_enp0s3.ipv4.address }}",
   "bootstrap_expect": 3,
   "ui": true,
   "retry_join": [{{ consul_retry_join }}],
   "log_level": "DEBUG",
   "enable_syslog": true,
   "acl_enforce_version_8": false
}
```
Config agent Consul:

```bash

{
    "server": false,
    "node_name": "{{ ansible_hostname }}",
    "datacenter": "dc1",
    "data_dir": "/var/lib/consul/data",
    "bind_addr":  "{{ ansible_enp0s3.ipv4.address }}",
    "client_addr": "127.0.0.1",
    "retry_join": [{{ consul_haproxy }}],
    "log_level": "DEBUG",
    "enable_syslog": true,
    "acl_enforce_version_8": false
 }
```
Config Vault:

```bash
storage "consul" {
  address = "127.0.0.1:8500"
  path = "vault/"
}
listener "tcp" {
  address       = "{{ address }}"
  cluster_address =  "{{ ansible_enp0s3.ipv4.address }}:8201"
  tls_disable     = 1
}

disable_mlock = true
disable_cache = true
api_addr =  {{ api_addr }}
cluster_addr = "http://{{ ansible_enp0s3.ipv4.address }}:8201" 
cluster_name = "{{ cluster_name }}"
ui = true
default_lease_ttl = "768h"
max_lease_ttl = "768h"
pid_file = "/var/lib/vault/vault.pid"

```

Provide the server's IP address in the inventory file on which we want to run this manual. How we are using Consul as the Vault backend.

Once done run command:

ansible-playbook playbook.yml

## Reference documentation
[haproxyproject](http://www.haproxy.org)
[vaultproject](https://www.vaultproject.io/docs/install)
[consulproject](https://www.consul.io/docs)
