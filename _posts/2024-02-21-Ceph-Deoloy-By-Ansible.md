---
layout: post
title: Easy Ceph Deployment by ceph-ansible
toc: true
categories: [Ceph]
---

I used `ceph-ansible` to deploy several Ceph 15 (octopus) and Ceph 17 (quincy) environments on Ubuntu 22.04 and Rocky 8 in the past, which worked well after some code changes. Recently I successfully deployed a Ceph 18 (reef) environment on Ubuntu 22.04 using `ceph-ansible` (again!). Overall it worked pretty well. I am sharing my steps in this article.

<!--more-->

Let's assume the Ubuntu 22.04 hosts have been installed. The following steps can be performed on any server which can reach ALL the Ubuntu hosts.

## Step 1: Upgrade ansible

The ansible needs to be installed or upgraded to more recent version.
```
$ pip3 install --upgrade ansible ansible-core

$ pip3 list |grep ansible
ansible                 9.2.0
ansible-base            2.10.17
ansible-core            2.16.3
```

If Ansible is not upgraded, an error might be displayed: `Ansible version must be either 2.15 or 2.16!`

## Step 2: Clone ceph-ansible repository

```
$ git clone https://github.com/ceph/ceph-ansible.git
$ cd ceph-ansible
$ git checkout stable-8.0   # This branch is used to deploy ceph 18 (reef)
```

## Step 3: Create configuration files

I created two files for ceph installation

- `ceph-hosts` defines the inventory for ansible

```
[mons]
ceph-1  ansible_host=10.0.33.11
ceph-2  ansible_host=10.0.33.12
ceph-3  ansible_host=10.0.33.13

[mgrs]
ceph-1
ceph-2
ceph-3

[osds]
ceph-1
ceph-2
ceph-3

[monitoring]
ceph-1
ceph-2
ceph-3

[clients]
ceph-1
```

- `group_vars/all.yml` defines some environment variables

You need to change some values for your use, for example `ansible_user`, `ansible_ssh_pass`, `public_network`, `cluster_network`.

{% highlight diff %}
{% raw %}
ansible_user: root
ansible_ssh_pass: password
no_log: false

cluster: ceph
configure_firewall: False
ceph_origin: repository
ceph_repository: community
ceph_stable_release: reef
ceph_mirror: http://eu.ceph.com
ceph_stable_key: http://eu.ceph.com/keys/release.asc
ceph_stable_repo: "{{ ceph_mirror }}/debian-{{ ceph_stable_release }}"
ceph_stable_distro_source: jammy
public_network: "10.0.32.0/20"
cluster_network: "10.0.32.0/20"
monitor_interface: eth0
ip_version: ipv4

ceph_conf_key_directory: /etc/ceph
# Permissions for keyring files in /etc/ceph
ceph_keyring_permissions: '0600'
cephx: true

devices:
  - '/dev/sdb'
ceph_conf_overrides:
   mon:
     mon_allow_pool_delete: true

dashboard_enabled: True
dashboard_protocol: http
dashboard_port: 8443
dashboard_admin_user: admin
dashboard_admin_password: password
grafana_admin_user: admin
grafana_admin_password: admin
grafana_uid: 472
grafana_datasource: Dashboard
grafana_dashboard_version: reef
grafana_port: 3000
grafana_allow_embedding: True
{% endraw %}
{% endhighlight %}

## Step 4: (Optional) Update hosts

I have faced some issues in the past, therefore I wrote small playbooks to fix them
- `update_authorized_keys.yaml`: copy local public key to Ubuntu hosts
- `update_hostname.yaml`: update hostname of Ubuntu hosts to the inventory hostname
- `update_resolv_conf.yaml`: update name servers in /etc/resolv.conf in Ubuntu hosts
- `remove-unattended-upgrades.yaml`: kill apt processes and remove unattended-upgrades

The content of files are appended to the end of this article.

```
$ cat > update_node.yaml <<EOF
---
- import_playbook: update_authorized_keys.yaml
- import_playbook: update_hostname.yaml 
- import_playbook: update_resolv_conf.yaml
- import_playbook: remove-unattended-upgrades.yaml
EOF

$ ansible-playbook update_node.yaml -i ceph-hosts

```

## Step 5: Install Ceph

```
$ ansible-galaxy install -r requirements.yml    # Install requirements

$ cp ./site.yml.sample site.yml

$ ansible-playbook site.yml -i ceph-hosts --extra-vars "yes_i_know=true"

```

`yes_i_know=true` is used to ignore the warning below
```
    TASK [Warn about ceph-ansible current status] ****************************************
    fatal: [localhost]: FAILED! => changed=false 
    msg: cephadm is the new official installer. Please, consider migrating. See https://docs.ceph.com/en/latest/cephadm/install for new deployments or https://docs.ceph.com/en/latest/cephadm/adoption for migrating existing deployments.
```

If everything goes well, the end of output will be similar as below
```
PLAY RECAP ******************************************************************************************************************************************************************
localhost                  : ok=0    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
ceph-1                     : ok=434  changed=74   unreachable=0    failed=0    skipped=526  rescued=0    ignored=0
ceph-2                     : ok=303  changed=60   unreachable=0    failed=0    skipped=396  rescued=0    ignored=0
ceph-3                     : ok=312  changed=63   unreachable=0    failed=0    skipped=393  rescued=0    ignored=0


INSTALLER STATUS ************************************************************************************************************************************************************
Install Ceph Monitor           : Complete (0:01:17)
Install Ceph Manager           : Complete (0:00:57)
Install Ceph OSD               : Complete (0:01:58)
Install Ceph Client            : Complete (0:00:22)
Install Ceph Dashboard         : Complete (0:00:33)
Install Ceph Grafana           : Complete (0:01:22)
Install Ceph Node Exporter     : Complete (0:02:15)
Install Ceph Crash             : Complete (0:00:14)
```

## Step 6: (Optional) Add a new OSD

To add a new OSD, for example `ceph-4`, update the `ceph-hosts` as below
```
[osds]
ceph-1
ceph-2
ceph-3
ceph-4  ansible_host=10.0.33.14
```

A small modification is required
{% highlight diff %}
{% raw %}
diff --git a/site.yml.sample b/site.yml.sample
index e5bd9de61..9350cc449 100644
--- a/site.yml.sample
+++ b/site.yml.sample
@@ -57,7 +57,7 @@
           - '!ohai'
       delegate_to: "{{ item }}"
       delegate_facts: True
-      with_items: "{{ groups['all'] | difference(groups.get('clients', [])) }}"
+      with_items: "{{ groups['all'] }}"
       run_once: true
       when: delegate_facts_host | bool
{% endraw %}
{% endhighlight %}

Then run the commands below
```
$ cp ./site.yml.sample site.yml

$ ansible-playbook site.yml -i ceph-hosts --limit ceph-4
```

Please refer to [Adding osd(s)](https://docs.ceph.com/projects/ceph-ansible/en/latest/day-2/osds.html)

## Step 7: Ceph Dashboard

Open the URL `http://ceph-1:8443` in browser, login with the username/password in `group_vars/all.yml`, the ceph dashboard is shown.

![Ceph Dashboard]({{ "/images/2024-02-21_18-56-Ceph-Dashboard.png" | absolute_url }}){:width="100%"}{:.glightbox}

## Ansible playbooks

### 1. update_authorized_keys.yaml

{% highlight diff %}
{% raw %}
---

- hosts: all
  tasks:
  - name: copy public key to remote server
    copy:
      src: ~/.ssh/id_rsa.pub
      dest: /root/.ssh/authorized_keys
{% endraw %}
{% endhighlight %}

### 2. update_hostname.yaml
{% highlight diff %}
{% raw %}
---
- hosts: all
  gather_facts: no
  tasks:

  - setup:

  - name: Set a hostname specifying strategy
    hostname:
      name: "{{ inventory_hostname }}"
      use: "systemd"
    when: ansible_facts.hostname != inventory_hostname

  - setup:

  - name: verify hostname
    ansible.builtin.assert:
      that:
        - ansible_facts.hostname == inventory_hostname
      fail_msg: "ansible_facts.hostname is not same as inventory_hostname"
      success_msg: "ansible_facts.hostname is same as inventory_hostname"
{% endraw %}
{% endhighlight %}

### 3. update_resolv_conf.yaml 
{% highlight diff %}
{% raw %}
---

- hosts: all
  tasks:
    - name: Overwrite /etc/resolv.conf
      # /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
      copy: 
        dest: /etc/resolv.conf
        force: yes
        content: |
            nameserver 8.8.8.8
            nameserver 1.1.1.1
{% endraw %}
{% endhighlight %}

### 4. remove-unattended-upgrades.yaml
{% highlight diff %}
{% raw %}
---

- hosts: all
  tasks:

  - name: Make sure /var/lib/dpkg/lock-frontend is not in use
    shell: "lsof -t /var/lib/dpkg/lock-frontend | kill -9 || true"
    ignore_errors: yes

  - name: Remove package unattended-upgrades
    apt:
      pkg: "{{ item }}"
      state: absent
      force: true
    with_items:
      - unattended-upgrades
    ignore_errors: yes

  - name: Make sure unattended-upgrades is completely purged
    shell: "dpkg --purge unattended-upgrades || true"
    ignore_errors: yes

{% endraw %}
{% endhighlight %}
