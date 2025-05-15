# Ansible Role - Patroni
[![Maintainer](https://img.shields.io/badge/maintained%20by-claranet-e00000?style=flat-square)](https://www.claranet.fr/)
[![License](https://img.shields.io/github/license/claranet/ansible-role-patroni?style=flat-square)](LICENSE)
[![Release](https://img.shields.io/github/v/release/claranet/ansible-role-patroni?style=flat-square)](https://github.com/claranet/ansible-role-patroni/releases)
[![Status](https://img.shields.io/github/actions/workflow/status/claranet/ansible-role-patroni/molecule.yml?branch=main&style=flat-square&label=tests)](https://github.com/claranet/ansible-role-patroni/actions?query=workflow%3A%22Ansible+Molecule%22)
[![Ansible version](https://img.shields.io/badge/ansible-%3E%3D2.10-black.svg?style=flat-square&logo=ansible)](https://github.com/ansible/ansible)
[![Ansible Galaxy](https://img.shields.io/badge/ansible-galaxy-black.svg?style=flat-square&logo=ansible)](https://galaxy.ansible.com/claranet/patroni)


> :star: Star us on GitHub — it motivates us a lot!

Installs and configures a [Patroni (HA solution for PostgreSQL)](https://github.com/zalando/patroni/) cluster on Debian and RedHat systems using this Ansible role.  

Patroni is a **lightweight high-availability (HA) template** for managing PostgreSQL clusters, written in Python. It supports multiple **Distributed Configuration Stores (DCS)** such as Consul, etcd, Kubernetes, and ZooKeeper, allowing flexible and resilient deployments.  

This role provides an automated and adaptable way to bootstrap a new Patroni cluster, manage configuration changes, and seamlessly integrate with [claranet.postgresql](https://github.com/claranet/ansible-role-postgresql) for PostgreSQL object management.  



## Table of Contents

1. [Role Requirements](#warning-requirements)
2. [Role Dependencies](#arrows_counterclockwise-dependencies)
3. [Role Installation](#zap-role-installation)
4. [Features and control tags/variables](#features-and-control-tagsvariables)
6. [Tested Linux/Patroni/PostgreSQL versions](#tested-linuxpatronipostgresql-versions)
7. [Getting started and prerequisites setup](#getting-started-and-prerequisites-setup)
    - [DCS setup](#dcs-setup)
    - [Example: Setting up consul using `ansible-consul`](#example-setting-up-consul-using-ansible-consul)
    - [Setting up other DCS options](#setting-up-other-dcs-options)
    - [PostgreSQL installation](#postgresql-installation)
8. [Role features in use](#role-features-in-use)
    - [Proxy usage](#proxy-usage)
    - [Patroni Installation](#patroni-installation)
    - [Configuration](#configuration)
      - [DCS specific parameters](#dcs-specific-parameters)
      - [Bootstrap configuration](#bootstrap-configuration)
      - [PostgreSQL configuration](#postgresql-configuration)
      - [Patroni regular configuration](#patroni-regular-configuration)
      - [DCS connection configuration](#dcs-connection-configuration)
      - [RestAPI configuration](#restapi-configuration)
      - [patronictl configuration](#patronictl-configuration)
      - [Watchdog configuration](#watchdog-configuration)
      - [Patroni tags configuration](#patronig-tags-configuration)
    - [DNSMASQ setup](#dnsmasq-setup)
    - [Manage PostgreSQL objects with claranet.postgresql](#manage-postgresql-objects-with-claranetpostgresql)
    - [Advanced customization](#advanced-customization)
9. [Full setup example](#pencil2-full-setup-example)
10. [Hardening](HARDENING.md)
11. [Contributing](CONTRIBUTING.md)
12. [License](LICENSE)
13. [Author information](#author-information)


## :warning: Requirements

- ansible-core >= 2.16, Ansible >= 9
- This role requires root privileges, so tell ansible to use `become: true` in any convenient way.


## :arrows_counterclockwise: Dependencies

[requirements.yml](molecule/shared/tools/requirements.yml)
```yaml
collections:
  - name: community.general
  - name: community.postgresql
    version: 3.2.0

roles:
  - src: https://github.com/claranet/ansible-role-postgresql.git
    name: claranet.postgresql
    version: feat/feature_control_with_variables
    scm: git
```


## :zap: Role Installation

```bash
ansible-galaxy install claranet.patroni
```


## Features and control tags/variables

This role support the following features and tags in the following order during execution:
Feature         | variable                 | tags
----------------|--------------------------|--------------------
Installation    | patroni_install          | install, installation
Configuration   | patroni_configure        | config, configure, configuration
DNSMASQ setup   | patroni_setup_dnsmasq    | dnsmasq


## Tested Linux/Patroni/PostgreSQL versions

`PostgreSQL 16`

Linux/PostgreSQL  | 4.0.5 |  4.0.5 
------------------|:----:|:----:
Debian 11         | Yes  | Yes  
Debian 12         | Yes  | Yes  
Ubuntu 22.04      | Yes  | Yes  
Ubuntu 24.04      | Yes  | Yes  
RockyLinux 8.9    | Yes  | Yes  
RockyLinux 9.3    | Yes  | Yes  


## Getting started and prerequisites setup

As mentioned earlier, Patroni is a **template** for managing PostgreSQL replication using a **Distributed Configuration Store (DCS)** such as Consul, Etcd, Exhibitor, or ZooKeeper.

Therefore, Patroni (and, by extension, this role) relies on a properly configured DCS and existing postgresql packages, so you must **set up the DCS and install the necessary PostgreSQL packages** before running this role.


### DCS setup
---

Since Patroni supports multiple DCS options, this role does **not** include their setup. However, it has been primarily tested with **Consul**, which is set as the **default DCS** in this role.  

#### Example: Setting up consul using `ansible-consul`
---

To set up a **Consul** cluster, you first need to install the `ansible-consul` role. Add the following content to your `requirements.yml` file and then run: `ansible-galaxy install -r requirements.yml`

_requirements.yml_
```yaml
roles:
  # ...
  - src: https://github.com/ansible-collections/ansible-consul.git
    name: ansible-consul
    scm: git
    version: master
```

Since the `ansible-consul` role requires all cluster nodes to be part of the same host group, we'll structure the inventory accordingly.

When invoking the role, we'll pass the variable `consul_group_name` with the group name containing the nodes.

Below is an example inventory and playbook for bootstrapping a 3-node Consul cluster on the Patroni servers. All hosts are placed within the `patroni_servers` hostgroup.

_Sample inventory (inventory/hosts.yml):_
```yaml 
---
patroni_servers:
  hosts:
    node1:
    node2:
    node3:
  vars:
    consul_advertise_address: "{{ ansible_default_ipv4.address }}"
    consul_addresses_dns: "127.0.0.1 {{ ansible_default_ipv4.address }}"
    consul_node_role: server
    consul_bootstrap_expect: true
    consul_recursors:
      - <local dns server>
      - 8.8.8.8
```

_Sample playbook:_
```yaml
- name: Install Consul
  hosts: patroni_servers
  become: true

  roles:
    - name: ansible-consul
      vars:
        consul_group_name: patroni_servers
```

Verifying the Consul Cluster

Once the playbook completes successfully, run the following command on any of the three nodes to check if the cluster is properly bootstrapped:
```bash
node1 $ consul members
```

For more customization options, refer to the full [ansible-consul documentation](https://github.com/ansible-collections/ansible-consul) 


### Setting up other DCS options  
---  

You can use the following Ansible roles to install alternative **Distributed Configuration Stores (DCS)** and update the `patroni_dcs` variable accordingly:  

- [**andrewrothstein.etcd-cluster**](https://github.com/andrewrothstein/ansible-etcd-cluster) – for setting up an **Etcd** cluster.  
- [**AnsibleShipyard.ansible-zookeeper**](https://github.com/AnsibleShipyard/ansible-zookeeper) – for deploying **ZooKeeper**.  

Make sure to configure Patroni to use the appropriate DCS by updating the `patroni_dcs` variable in your inventory.  


### PostgreSQL installation
---

After setting up the DCS, the next step is to install the required PostgreSQL packages.  

To do this, install the `claranet.postgresql` role by adding the following content to your `requirements.yml` file and then running `ansible-galaxy install -r requirements.yml`.


_requirements.yml_
```yaml
roles:
  # ...
  - src: https://github.com/claranet/ansible-role-postgresql.git
    name: claranet.postgresql
    version: feat/feature_control_with_variables
    scm: git
```

Below is a sample playbook that installs the PostgreSQL version `16` without additional configuration, as Patroni will handle it.
> :rotating_light: Make sure to set the variables `postgresql_is_patroni, postgresql_install, postgresql_only_install` as shown in the following example so that only the packages installation in performed by `claranet.postgresql`.

_Sample playbook:_
```yaml
- name: Install PostgreSQL
  hosts: patroni_servers
  become: true
  vars:
    patroni_postgresql_version: 16

  roles:
    - name: claranet.postgresql
      vars:
        postgresql_is_patroni: true
        postgresql_version: "{{ patroni_postgresql_version }}"
        postgresql_install: true
        postgresql_only_install: true
        # patroni_http_pkg_proxy: <proxy_address>
```

For more customization options, refer to the full [claranet.postgresql documentation](https://github.com/ansible-collections/ansible-consul) for further customization.


## Role features in use

> :rotating_light: When using this role in dry-run mode (`--check`), it should function almost correctly, reporting configuration changes and Patroni restarts (albeit with less accurate details on the reasons causing the restart). If you encounter any issues related to dry-run, please feel free to open an issue.


### Proxy usage
---
This role supports use of proxies.

The variables `patroni_http_general_proxy` and `patroni_https_general_proxy` can be used to specify a proxy for general internet access (such as downloading files).

The variables `patroni_http_pkg_proxy` and `patroni_https_pkg_proxy` can be used to specify a proxy for package manager interaction (such as downloading packages or updating cache).

_Notes:_ 

These variables are translated to environnement variables `http_proxy` and `https_proxy` which are passed to corresponding tasks.


### Patroni installation
---
_default Patroni version is current latest version_

> :rotating_light: This role requires all the patroni nodes be put inside an ansible host group and that hostgroup name specified in the variable `patroni_group_name`.

```yaml
# Patroni version to install. "" => install the latest version
# This version is passed directory to pip. So if you want a specific version but want a package based 
# installation, you might need to specific an obscure version number like this: 4.0.5-1.pgdg110+1
patroni_version: ""
# Configured DCS, check https://patroni.readthedocs.io/en/latest/installation.html#general-installation-for-pip for other DCS that can be specified
patroni_dcs: consul
# Installed PostgreSQL version
patroni_postgresql_version: 16
# Default replication user created during bootstrap
patroni_replication_username: replicator
patroni_replication_password: repuserpasswd
# Default superuser configuration
patroni_superuser_username: postgres
patroni_superuser_password: supersecretpostgrespasswd
```


Advanced installation configuration:
```yaml
# Whether or not to run patroni installation tasks
patroni_install: true
# Patroni being originally a python package, this role defaults to using pip to install patroni.
# Set this variable to false to install patroni using system packages
# With pip based installation it's much easier to support all the available DCS
patroni_install_from_pip: true
# When using package based installation, add an entry for the configured dcs containing
# the packages that are required for Patroni to access the DCS
# Default value for patroni_dcs_packages is distribution dependent: (Check vars/<os distro>.yml)
# patroni_dcs_packages:
#   <dcs name>: []
# Pip packages for Patroni installation
patroni_pip_packages:
  - setuptools
  - psycopg2-binary
  - patroni[{{ patroni_dcs }}]{{ '==' ~ patroni_version if patroni_version | d('') | length > 0 else '' }}

# Installation directory when installing from pip
patroni_venv_install_dir: /opt/patroni
# Patroni bin directory: (allows for unifying pip and system package installation)
patroni_bin_dir: "{{ patroni_install_from_pip | ternary(patroni_venv_install_dir ~ '/bin', '/usr/bin') }}"
# Creates a /etc/profile.d/patroni.sh to include patroni binaries in the PATH
patroni_configure_profile: true
# Configures a patronictl helper alias in /etc/profile.d/patroni.sh alias patronictl='patronictl -c <config.yml>'
# This eliminates the need to always specify the configuration file while invoking patronictl which is very tedious
patroni_configure_helper_alias: true
# PostgreSQL port
patroni_postgresql_port: 5432
# Patroni configuration directory
patroni_config_dir: /etc/patroni
# Patroni regular configuration file within "patroni_config_dir"
patroni_config_file: config.yml
# Patroni configuration file within "patroni_config_dir" which contains only bootstrap data. (information under the bootstrap key)
# After Patroni cluster is bootstrapped this file is never changed as Patroni would not take any of those changes into account anyways
patroni_bootstrap_file: bootstrap.yml
# PostgreSQL system user/group
patroni_system_user: postgres
patroni_system_group: postgres
```


### Configuration
---
When using **Patroni**, some parameters are stored in the **Distributed Configuration Store (DCS)**, while others are kept in the local configuration file at `/etc/patroni/config.yml`. If there is a conflict between these two sources, the local `config.yml` typically takes precedence.  

By default, this role **only reloads the Patroni service** when configuration changes are made. This approach helps prevent potential service outages.  

However, some configuration changes require **restarting the underlying PostgreSQL engine** to take effect. Since automatic restarts are disabled by default, you can temporarily set the following variable to allow the role to restart the PostgreSQL engine when needed.
> :rotating_light: DO NOT LEAVE `patroni_config_change_allow_restart` PERMANENTLY at `true` to avoid potential service outages

```yaml
# Controls whether or not a restart can be performed when a configuration change
# requires a patronictl -c <config.yml> restart to be taken into account
patroni_config_change_allow_restart: false
```


#### DCS specific parameters
---

DCS specific parameters are documented [here](https://patroni.readthedocs.io/en/latest/dynamic_configuration.html#dynamic-configuration) with their default values specified below.


```yaml
patroni_dcs_loop_wait: 10
patroni_dcs_ttl: 30
patroni_dcs_retry_timeout: 10
patroni_dcs_maximum_lag_on_failover: 1048576
patroni_dcs_maximum_lag_on_syncnode: -1
patroni_dcs_max_timelines_history: 0
patroni_dcs_check_timeline: false
patroni_dcs_primary_start_timeout: 300
patroni_dcs_primary_stop_timeout: 0 
patroni_dcs_synchronous_mode: false # should be on/off/quorum
patroni_dcs_synchronous_mode_strict: false
patroni_dcs_synchronous_node_count: 1
patroni_dcs_failsafe_mode: false
patroni_dcs_postgresql_use_pg_rewind: false
patroni_dcs_postgresql_use_slots: true
patroni_dcs_postgresql_recovery_conf: []
#  - { option: "standby_mode",    value: "on" }
#  - { option: "restore_command", value: "cp ../wal_archive/%f %p" }
# PostgreSQL parameters that need be stored inside the DCS. It is advised to store this variable
# at the hostgroup level to make sure these parameters have the same value accross all the cluster nodes
patroni_dcs_postgresql_parameters:
  - { option: "max_connections", value: "100" }
  - { option: "max_locks_per_transaction", value: "64" }
  - { option: "max_worker_processes", value: "8" }
  - { option: "max_prepared_transactions", value: "0" }
  - { option: "wal_level", value: "replica" }
  - { option: "wal_log_hints", value: "on" }
  - { option: "track_commit_timestamp", value: "off" }
  - { option: "max_wal_senders", value: "10" }
  - { option: "max_replication_slots", value: "10" }
  - { option: "wal_keep_segments", value: "8" }
  - { option: "wal_keep_size", value: "128MB" }
patroni_dcs_standby_cluster: []
  # - { option: "host", value: "" }
  # - { option: "port", value: "" }
  # - { option: "primary_slot_name", value: "" }
  # - { option: "create_replica_methods", value: "" }
  # - { option: "restore_command", value: "" }
  # - { option: "archive_cleanup_command", value: "" }
  # - { option: "recovery_min_apply_delay", value: "" }
patroni_dcs_member_slots_ttl: 30min
# Permanent slots
patroni_dcs_slots: []
#  - { name: "permanent_physical_1", type: "physical" }
#  - { name: "permanent_logical_1",  type: "logical", database: "foo", plugin: "pgoutput" }
patroni_dcs_ignore_slots: []
#  - { name: "ignore_1", type: "physical" }
#  - { name: "permanent_logical_1",  type: "logical", database: "foo", plugin: "pgoutput" }
```


_Notes :_

For a properly functionning cluster, Patroni requires that [certain PostgreSQL parameters](https://patroni.readthedocs.io/en/latest/patroni_configuration.html#important-rules) to be identical accross all nodes. These parameters are stored in the DCS under the `postgresql.parameters` key.


By default the variable `patroni_dcs_postgresql_parameters` only includes these required parameters. The variable `patroni_refine_dcs_postgresql_params` defaults to `true` and ensures that:
* `patroni_dcs_postgresql_parameters` contains only DCS-specific parameters. Any additional parameters added here will be ignored.
* `patroni_postgresql_parameters` does not include DCS-specific parameters, meaning the role actively removes non-DCS specifics parameters from this variable.

If you want to have full control over which PostgreSQL parameters are stored inside the DCS, set the variable `patroni_refine_dcs_postgresql_params` to `false`.


#### Bootstrap configuration
---
_The bootstrap configuration is stored in a separate file (`patroni_bootstrap_file`)_

The boostrap configuration parameters are documenterd [here](https://patroni.readthedocs.io/en/latest/yaml_configuration.html#bootstrap-configuration) with their default values specified below.

> :rotating_light: **Once the Patroni cluster is bootstrapped, any modifications to these variables will be ignored by Patroni (and this role), as their modification will have no impact.**

```yaml
# https://patroni.readthedocs.io/en/latest/replica_bootstrap.html#bootstrap
patroni_bootstrap_method_name: ""
patroni_bootstrap_method_command: ""
patroni_bootstrap_method_keep_existing_recovery_conf: false
patroni_bootstrap_method_no_params: false
patroni_bootstrap_method_recovery_conf: []
#  - { option: "standby_mode",    value: "on" }
#  - { option: "restore_command", value: "cp ../wal_archive/%f %p" }
patroni_bootstrap_initdb:
  - { option: "encoding", value: "UTF8" }
  - { option: "data-checksums" }
  # - { option: "locale", value: "C.UTF8" }
patroni_bootstrap_users:
  - { name: "{{ patroni_superuser_username }}", password: "{{ patroni_superuser_password }}", options: [] }
  - { name: "{{ patroni_replication_username }}", password: "{{ patroni_replication_password }}", options: ['replication'] }
patroni_bootstrap_post_bootstrap: ""
patroni_bootstrap_post_init: ""
```


#### PostgreSQL configuration
---

PostgreSQL configuration parameters are documented [here](https://patroni.readthedocs.io/en/latest/yaml_configuration.html#postgresql) with their default values specified below.


```yaml
patroni_postgresql_authentication:
  - { type: "superuser", username: "{{ patroni_superuser_username }}", password: "{{ patroni_superuser_password }}" }
  - { type: "replication", username: "{{ patroni_replication_username }}", password: "{{ patroni_replication_password }}" }
  # - { type: "rewind", username: "{{ patroni_replication_username }}", password: "{{ patroni_replication_password }}" }
# The directory on the controller where the callback scripts are stored
patroni_postgresql_callbacks_dir: callbacks
patroni_postgresql_callbacks:
  - { event: "on_reload", script: "" }
  - { event: "on_restart", script: "" }
  - { event: "on_role_change", script: "" }
  - { event: "on_start", script: "" }
  - { event: "on_stop", script: "" }
patroni_postgresql_connect_address: "{{ ansible_host }}:{{ patroni_postgresql_port }}"
patroni_postgresql_proxy_address: null
patroni_postgresql_create_replica_methods:
#  - pgbackrest
#  - wal_e
  - basebackup
patroni_postgresql_pgbackrest: []
#  - { option: "command",   value: "/usr/bin/pgbackrest --stanza=main --delta restore" }
#  - { option: "keep_data", value: "true" }
#  - { option: "no_params", value: "true" }
patroni_postgresql_wal_e: []
#  - { option: "command",   value: "patroni_wale_restore" }
#  - { option: "no_master", value: "1" }
#  - { option: "envdir",    value: "/etc/wal_e/envdir" }
#  - { option: "use_iam",   value: "1" }
patroni_postgresql_basebackup: []
#  - { option: "checkpoint", value: "fast" }
#  - { option: "max-rate",   value: "100M" }
#  - { option: "verbose" }

patroni_postgresql_data_dir: os dependent, checkout vars/<os-distro>.yml
patroni_postgresql_config_dir: os dependent, checkout vars/<os-distro>.yml
patroni_postgresql_bin_dir: os dependent, checkout vars/<os-distro>.yml
patroni_postgresql_bin_name: {}
  # pg_ctl:
  # initdb:
  # pgcontroldata:
  # pg_basebackup:
  # postgres:
  # pg_isready:
  # pg_rewind:
patroni_postgresql_listen: 0.0.0.0:{{ patroni_postgresql_port }}
patroni_postgresql_use_unix_socket: true
patroni_postgresql_use_unix_socket_repl: false
patroni_postgresql_pgpass: os dependent, checkout vars/<os-distro>.yml
patroni_postgresql_recovery_conf: []
#  - { option: "standby_mode",    value: "on" }
#  - { option: "restore_command", value: "cp ../wal_archive/%f %p" }
patroni_postgresql_custom_conf: ""
patroni_postgresql_parameters:
  - { option: "unix_socket_directories", value: "/var/run/postgresql" }
patroni_postgresql_pg_hba: []
#  - { type: "host", database: "all", user: "all", address: "0.0.0.0/0", method: "ident", options: "map=omicron" }
#  - { type: "host", database: "replication", user: "{{ patroni_replication_username }}", address: "0.0.0.0/0", method: "md5" }
patroni_postgresql_pg_ident: []
#  - { mapname: "omicron", sysuser: "robert", pguser: "bob" }
patroni_postgresql_pg_ctl_timeout: 60
patroni_postgresql_use_pg_rewind: null
patroni_postgresql_remove_data_directory_on_rewind_failure: false
patroni_postgresql_remove_data_directory_on_diverged_timelines: false
patroni_postgresql_replica_method: ""
patroni_postgresql_pre_promote: ""
patroni_postgresql_before_stop: ""
```


#### Patroni regular configuration
---

General Patroni configuration parameters stored inside the `config.yml` are specified below with a link pointing to their documentation.

```yaml
# https://patroni.readthedocs.io/en/latest/yaml_configuration.html#global-universal
patroni_scope: main
patroni_namespace: /service/
patroni_name: "{{ inventory_hostname }}"

# https://patroni.readthedocs.io/en/latest/yaml_configuration.html#log
patroni_log_destination: stderr
patroni_log_level: INFO
patroni_log_format: "%(asctime)s %(levelname)s: %(message)s"
patroni_log_dateformat: ""
patroni_log_max_queue_size: 1000
patroni_log_dir: /var/log/patroni
patroni_log_file_num: 4
patroni_log_file_size: 25000000
patroni_log_loggers:
  - { module: "patroni.postmaster", level: "WARNING" }
  - { module: "urllib3", level: "DEBUG" }
```


#### DCS connection configuration
---

The DCS (Distributed Configuration Store) connection settings are stored inside `config.yml`. These settings determine how Patroni communicates with the DCS backend. Below are the available configurations for different DCS types.

[CONSUL](https://patroni.readthedocs.io/en/latest/yaml_configuration.html#consul)
```yaml
patroni_consul_port: 8500
patroni_consul_host: "127.0.0.1:{{ patroni_consul_port }}"
patroni_consul_url: ""
patroni_consul_scheme: http
patroni_consul_token: ""
patroni_consul_verify: ""
patroni_consul_cacert: ""
patroni_consul_cert: ""
patroni_consul_key: ""
patroni_consul_dc: ""
patroni_consul_checks: ""
patroni_consul_register_service: true
patroni_consul_service_check_interval: 5s
patroni_consul_consistency: default
```

[ETCD](https://patroni.readthedocs.io/en/latest/yaml_configuration.html#etcd)
```yaml
patroni_etcd_host: ""
patroni_etcd_hosts: 127.0.0.1:2379
patroni_etcd_use_proxies: false
patroni_etcd_url: ""
patroni_etcd_proxy: ""
patroni_etcd_srv: ""
patroni_etcd_protocol: http
patroni_etcd_username: ""
patroni_etcd_password: ""
patroni_etcd_cacert: ""
patroni_etcd_cert: ""
patroni_etcd_key: ""
```

[ZOOKEEPER](https://patroni.readthedocs.io/en/latest/yaml_configuration.html#zookeeper)
```yaml
patroni_zookeeper_hosts: 127.0.0.1:2181
patroni_zookeeper_use_ssl: false
patroni_zookeeper_cacert: null
patroni_zookeeper_cert: null
patroni_zookeeper_key: null
patroni_zookeeper_key_password: null
patroni_zookeeper_verify: true
patroni_zookeeper_set_acls: null
patroni_zookeeper_auth_data: {}
```

[EXHIBITOR](https://patroni.readthedocs.io/en/latest/yaml_configuration.html#exhibitor)
```yaml
patroni_exhibitor_hosts: ""
patroni_exhibitor_port: ""
patroni_exhibitor_poll_interval: null
```


#### RestAPI configuration
---

RestAPI configuration parameters are documented [here](https://patroni.readthedocs.io/en/latest/yaml_configuration.html#rest-api) with their default values specified below.

```yaml
patroni_restapi_port: 8008
patroni_restapi_connect_address: "{{ ansible_host }}:{{ patroni_restapi_port }}"
patroni_restapi_listen: "0.0.0.0:{{ patroni_restapi_port }}"
patroni_restapi_username: ""
patroni_restapi_password: ""
patroni_restapi_certfile: ""
patroni_restapi_keyfile: ""
patroni_restapi_keyfile_password: ""
patroni_restapi_cafile: ""
patroni_restapi_ciphers: ""
patroni_restapi_verify_client: ""
patroni_restapi_allowlist: []
patroni_restapi_allowlist_include_members: []
patroni_restapi_http_extra_headers: ""
patroni_restapi_https_extra_headers: ""
patroni_restapi_request_queue_size: 5
```


#### PatroniCTL configuration
---

PatroniCTL configuration parameters are described [here](https://patroni.readthedocs.io/en/latest/yaml_configuration.html#ctl) with their default values specified below.

```yaml
# Basic-auth username for accessing protected REST API endpoints. If not provided patronictl will use the value provided for REST API “username” parameter.
patroni_ctl_authentication_username: ""
# Basic-auth password for accessing protected REST API endpoints. If not provided patronictl will use the value provided for REST API “password” parameter.
patroni_ctl_authentication_password: ""
# Allow connections to REST API without verifying SSL certs.
patroni_ctl_insecure: ""
# Specifies the file with the CA_BUNDLE file or directory with certificates of
# trusted CAs to use while verifying REST API SSL certs. If not provided patronictl will use the value provided for REST API “cafile” parameter.
patroni_ctl_cacert: ""
# Specifies the file with the client certificate in the PEM format.
patroni_ctl_certfile: ""
# Specifies the file with the client secret key in the PEM format.
patroni_ctl_keyfile: ""
# Specifies a password for decrypting the client keyfile.
patroni_ctl_keyfile_password: ""
```


#### Watchdog configuration
---
Watchdog configuration parameters are described [here](https://patroni.readthedocs.io/en/latest/yaml_configuration.html#watchdog) with their default values specified below.

```yaml
patroni_watchdog_mode: 'off' # off/automatic/required
patroni_watchdog_device: /dev/watchdog
patroni_watchdog_safety_margin: 5
```


#### Patronig tags configuration
---
Tags configuration parameters are documented [here](https://patroni.readthedocs.io/en/latest/yaml_configuration.html#tags) with their default values specified below.

```yaml
patroni_tags:
  - { name: "nofailover", value: "false" }
  - { name: "noloadbalance", value: "false" }
  - { name: "clonefrom", value: "false" }
  - { name: "nosync", value: "false" }
  - { name: "replicatefrom", value: "" }
```


### DNSMASQ setup
---
_dnsmasq setup is disabled (`patroni_setup_dnsmasq: false`) by default_

> :rotating_light: This is an opinionated configuration that assumes systemd-resolved is managing DNS on the host. If another service is handling DNS resolution, behavior may be unpredictable or break entirely.

When using Consul as the Distributed Configuration Store (DCS), it provides a built-in DNS server that dynamically resolves `master/primary/replica` nodes in a Patroni cluster.
To integrate this with system DNS resolution, dnsmasq is used as a lightweight local DNS forwarder.

Ensure that `consul_addresses_dns` variable contains `127.0.0.1` so the Consul DNS server listens on `localhost`.

You can test the DNS resolution using the dig command:
```bash
# Example: Querying the Patroni primary node through Consul's DNS service
node1 $ dig @127.0.0.1 -p 8600 master.main.service.consul
```

This setup ensures that each Patroni node can locally resolve DNS names in the format: `<master/primary/replica>.<patroni_scope>.service.consul` where `patroni_scope` defaults to the Patroni cluster name.


```yaml
# Install & configure dnsmasq
patroni_setup_dnsmasq: false
# Listening address for dnsmasq
patroni_dnsmasq_listen_address: 127.0.1.53
# Listening port for dnsmasq
patroni_dnsmasq_port: 53
# Listening interface
patroni_dnsmasq_interface: lo
# List of the upstream dns server
patroni_dnsmasq_upstream_servers: []
# The addresses of the built-in consul DNS servers. 
# This can be updated to specify any consul dns server
patroni_dnsmasq_consul_dns_servers:
  - ip: 127.0.0.1
    port: 8600
    domain: consul
# Extra dnsmasq configuration
patroni_dnsmasq_extra_config: ""
# IP address used in the 'DNS=' setting of systemd-resolved to route DNS queries through dnsmasq
patroni_dnsmasq_resolve_dns: "{{ patroni_dnsmasq_listen_address }}"
```


### Manage PostgreSQL objects with claranet.postgresql
---

After your Patroni cluster is bootstrapped, you can use the claranet.postgresql role to manage PostgreSQL objects like databases, users, and extensions.

However, since Patroni dynamically manages replication and failover, any node can become the primary(master) at any time. This creates a challenge for `claranet.postgresql`, which relies on the `postgresql_replication_role` variable to determine the primary server, which can cause inconsistencies if not handled correctly.

To ensure `claranet.postgresql` always interacts with the correct primary instance, the following variables are available.


```yaml
# Set the variable postgresql_conn_vars with the options required to connect to the local PostgreSQL instance
patroni_set_pg_conn_vars: true
# Set the variable postgresql_replication_role to allow claranet.postgresql to correctly identify the primary server
patroni_set_pg_replication_role: true
```

You can then invoke the `claranet.postgresql` role like this after the claranet.patroni is executed.

Once `claranet.patroni` has set up the cluster, and updated the variables `postgresql_conn_vars`, `postgresql_replication_role`, you can safely apply `claranet.postgresql` to manage PostgreSQL objects.

```yaml
- role: claranet.postgresql
  vars:
    postgresql_is_patroni: true
    postgresql_version: "{{ patroni_postgresql_version }}"
    postgresql_replication: true
    postgresql_install: false
    postgresql_users:
      # Create two groups 'group1' and 'group2' by making use of thr role_attr_flags attribute
      - name: group1
        role_attr_flags: NOLOGIN
```


### Advanced customization
---

It is highly recommended you modify these variables only if you know what you're doing.

```yaml
# Enable patroni configuration
patroni_configure: true
# Prefer using patroni_postgresql_pg_hba as it has precedence over this variable
patroni_dcs_postgresql_pg_hba: []
#  - { type: "host", database: "all", user: "all", address: "0.0.0.0/0", method: "ident", options: "map=omicron" }
#  - { type: "host", database: "replication", user: "{{ patroni_replication_username }}", address: "0.0.0.0/0", method: "md5" }
# Prefer using patroni_postgresql_pg_ident as it has precedence over this variable
patroni_dcs_postgresql_pg_ident: []
#  - { mapname: "omicron", sysuser: "robert", pguser: "bob" }

# Prefer using patroni_postgresql_hba as this is deprecated
patroni_bootstrap_pg_hba: []
#  - { type: "host", database: "all", user: "all", address: "0.0.0.0/0", method: "ident", options: "map=omicron" }
#  - { type: "host", database: "replication", user: "{{ patroni_replication_username }}", address: "0.0.0.0/0", method: "md5" }

# Disable logging on some tasks that might output sensitive information
patroni_no_log: false

# Required packages to install Patroni using pip
patroni_pip_install_pkg_prerequisites: os dependent, checkout vars/<os-distro>.yml 
# Packages required to enable Patroni to connect to the DCS when installing the dcs
patroni_dcs_packages: os dependent, checkout vars/<os-distro>.yml 
# Packages used to install Patroni when patroni_install_from_pip: false
patroni_system_packages: os dependent, checkout vars/<os-distro>.yml 
```

## :pencil2: Full setup example

_hosts.yml_
```yaml
---
patroni_servers:
  hosts:
    node1:
    node2:
    node3:
  vars:
    consul_advertise_address: "{{ ansible_default_ipv4.address }}"
    consul_addresses_dns: "127.0.0.1 {{ ansible_default_ipv4.address }}"
    consul_node_role: server
    consul_bootstrap_expect: true
    # upstream DNS servers;
    consul_recursors:
      - 127.0.0.11 # this address provided to containers by bridge based docker networks
      - 8.8.8.8

    patroni_version: ""
    patroni_postgresql_version: 16
    patroni_replication_username: replicator
    patroni_replication_password: s3cr3tpasswd
    patroni_superuser_username: postgres
    patroni_superuser_password: s3cr3tpasswd

    patroni_dcs_postgresql_parameters:
      - {option: "max_connections", value: 400}
      - {option: "wal_keep_segments", value: 10}
      - {option: "max_wal_senders", value: 11}

    patroni_postgresql_parameters:
      - {option: "unix_socket_directories", value: "/var/run/postgresql"}
      - {option: "log_rotation_size", value: "23MB"}  # doesnt require restart

    patroni_postgresql_pg_hba:
      - {type: "local", database: "all", user: "postgres", method: "peer"}
      - {type: "host", database: "replication", user: "{{ patroni_replication_username }}", address: "0.0.0.0/0", method: "md5"}

    patroni_dnsmasq: true
    patroni_dnsmasq_upstream_servers: [127.0.0.11]
    patroni_dnsmasq_consul_dns_servers:
      - ip: 127.0.0.1 # Address on which the local consul dns server is listening to
        port: 8600

    postgresql_users:
      # Create two groups 'group1' and 'group2' by making use of thr role_attr_flags attribute
      - name: group1
        role_attr_flags: NOLOGIN
      - name: user1
    postgresql_databases:
      - name: db1
        owner: user1
```


_palybook.yml_
```yaml
---
- name: Setup a Patroni cluster
  hosts: patroni_servers
  become: true
  gather_facts: true

  roles:
    # Install consul dcs
    - role: ansible-consul
      vars:
        consul_group_name: patroni_servers

    # Install PostgreSQL packages
    # Can be safely commented after the packages installation is done for the first time
    - role: claranet.postgresql
      vars:
        postgresql_is_patroni: true
        postgresql_install: true
        postgresql_only_install: true
        postgresql_version: "{{ patroni_postgresql_version }}"

    # Setup Patroni cluster
    - role: claranet.patroni
      vars:
        patroni_group_name: patroni_servers

    # Manage postgresql objects (users, databases, privileges, etc...)
    - role: claranet.postgresql
      vars:
        postgresql_is_patroni: true
        postgresql_install: false
        postgresql_version: "{{ patroni_postgresql_version }}"
```


## :closed_lock_with_key: [Hardening](HARDENING.md)

## :heart_eyes_cat: [Contributing](CONTRIBUTING.md)
Checkout the [Contributing](CONTRIBUTING.md) if you are looking for a guide on how to setup an environnement so you can test this role as a developper.

## :copyright: [License](LICENSE)

[Mozilla Public License Version 2.0](https://www.mozilla.org/en-US/MPL/2.0/)

## Author information

Proudly made by the Claranet team and inspired by:
- [Kostiantyn Nemchenko](https://github.com/kostiantyn-nemchenko/ansible-role-patroni/tree/master)
