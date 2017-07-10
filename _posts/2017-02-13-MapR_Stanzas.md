---
layout: post
title: Automating MapR with MapR Stanzas
tags: [mapr, automation]
bigimg: /img/keyboard.jpg
---

In my life as a technical marketeer for MapR I have configured more clusters than you can shake a stick at. So, imagine my excitement when I heard that MapR installations can be automated with a new capability called, "MapR Stanzas".

MapR Stanzas allow you to automate the MapR installation process by specifying in a YAML config file all the options that you would have otherwise specified in the MapR installation web UI.  You can read more about MapR stanzas at http://maprdocs.mapr.com/home/AdvancedInstallation/Stanzas/RunningSilentInstaller.html. 

Here's how I installed MapR in a test environment using MapR stanzas:

1. Download the MapR installer

    ```wget http://package.mapr.com/releases/installer/mapr-setup.sh -P /tmp```

2. Run the MapR installer

    ```sudo bash /tmp/mapr-setup.sh```

3. When it completes, it will advise you to complete the installation through a webui (e.g. `https://<hostname>:9443`), but don't do that. Instead, complete the installation with stanzas, like this:

    ```/opt/mapr/installer/bin/mapr-installer-cli install -u mapr:q7wp4XaAQpvdn@localhost -nv -t /opt/mapr/installer/examples/ian01.yaml```

I've attached my YAML to the end of this post for your reference.

What if you don't want to write the stanza YAML file by hand?  Then just install via the webui (e.g. `https://<hostname>:9443`), then export a stanza file with the export command (shown below). Then next time you install you should be able to use that YAML file. For example, here's how I captured the configurations I specified for our Tech Marketing MapR cluster:

  ```/opt/mapr/installer/bin/mapr-installer-cli export -u mapr:q7wp4XaAQpvdn@localhost -n --file /opt/mapr/installer/examples/ian01.yaml```

That file will not contain your ssh password, so you'll want to add "ssh_password" under the config section, like this:

    ssh_id: mapr
    ssh_password: q7wp4XaAQpvdn
    ssh_method: PASSWORD
    ssh_port: 22

Then validate the config file like this:

  ```/opt/mapr/installer/bin/mapr-installer-cli check -u mapr:q7wp4XaAQpvdn@localhost -n -t /opt/mapr/installer/examples/ian01.yaml```

If the check succeeds, you will see no output and a return status of 0.

For your reference I've included the YAML file that I exported from my test cluster below.

{% highlight yaml %}
environment:
  mapr_core_version: 5.2.1
config:
  admin_id: mapr
  cluster_admin_create: false
  cluster_admin_gid: 1000
  cluster_admin_group: mapr
  cluster_admin_id: mapr
  cluster_admin_uid: 1000
  cluster_name: my.cluster.com
  data:
    diskStripeDefault: 3
    licenseType: M3
    licenseValidation: INSTALL
    process_type: install
    showAdvancedServiceConfig: false
    showAdvancedServiceSelection: false
    showLicenseValidation: true
    showLogs: false
    template: template-05-converged
  dial_home_id: '843699986'
  disk_format: true
  disk_stripe: 3
  disks:
  - /dev/sdc
  elasticsearch_path: /opt/mapr/es_db
  environment: azure
  hosts:
  - 10.1.1.4
  - 10.1.1.5
  - 10.1.1.6
  install_dir: /opt/mapr/installer
  installer_version: 1.5.201705021610
  license_accepted: true
  license_modules:
  - DATABASE
  - HADOOP
  - STREAMS
  license_type: M3
  links:
    self: https://localhost:9443/api/config
    services: https://localhost:9443/api/config/services
  mapr_id: '64468'
  mapr_name: jklfdsalkj@yahoo.com
  mep_version: '3.0'
  no_internet: false
  os_version: ubuntu_14.04
  repo_core_url: http://package.mapr.com/releases
  repo_eco_url: http://package.mapr.com/releases
  services:
    mapr-asynchbase:
      enabled: true
      version: 1.7.0
    mapr-cldb:
      enabled: true
      version: 5.2.1
    mapr-collectd:
      enabled: true
      version: 5.7.1
    mapr-core:
      enabled: true
      version: 5.2.1
    mapr-drill:
      enabled: true
      version: '1.10'
    mapr-elasticsearch:
      enabled: true
      version: 2.3.3
    mapr-fileserver:
      enabled: true
      version: 5.2.1
    mapr-fluentd:
      enabled: true
      version: 0.14.00
    mapr-grafana:
      enabled: true
      version: 4.1.2
    mapr-hbase:
      enabled: true
      version: '1.1'
    mapr-hbase-common:
      enabled: true
      version: '1.1'
    mapr-hbase-rest:
      enabled: true
      version: '1.1'
    mapr-hbasethrift:
      enabled: true
      version: '1.1'
    mapr-historyserver:
      enabled: true
      version: 5.2.1
    mapr-hive:
      enabled: true
      version: '2.1'
    mapr-hive-client:
      enabled: true
      version: '2.1'
    mapr-hivemetastore:
      database:
        create: false
        type: DEFAULT
      enabled: true
      version: '2.1'
    mapr-hiveserver2:
      enabled: true
      version: '2.1'
    mapr-hivewebhcat:
      enabled: true
      version: '2.1'
    mapr-httpfs:
      enabled: true
      version: '1.0'
    mapr-hue:
      database:
        create: false
        type: DEFAULT
      enabled: true
      version: 3.12.0
    mapr-kafka:
      enabled: true
      version: 0.9.0
    mapr-kibana:
      enabled: true
      version: 4.5.4
    mapr-librdkafka:
      enabled: true
      version: 0.9.0
    mapr-nfs:
      enabled: true
      version: 5.2.1
    mapr-nodemanager:
      enabled: true
      version: 5.2.1
    mapr-oozie:
      database:
        create: false
        type: DEFAULT
      enabled: true
      version: 4.3.0
    mapr-opentsdb:
      enabled: true
      version: 2.3.0
    mapr-resourcemanager:
      enabled: true
      version: 5.2.1
    mapr-spark:
      enabled: true
      version: 2.1.0
    mapr-spark-client:
      enabled: true
      version: 2.1.0
    mapr-spark-historyserver:
      enabled: true
      version: 2.1.0
    mapr-streams-clients:
      enabled: true
      version: 0.9.0
    mapr-webserver:
      enabled: true
      version: 5.2.1
    mapr-yarn:
      enabled: true
      version: 5.2.1
    mapr-zookeeper:
      enabled: true
      version: 5.2.1
  services_version: 1.5.201705021610
  ssh_id: mapr
  ssh_method: PASSWORD
  ssh_port: 22
  state: INIT
hosts:
- completion: 100
  disks:
    /dev/fd0:
      selected: false
      size: 0 Bytes
    /dev/sda1:
      selected: false
      size: 29 GB
      unavailable: Disk mounted at /
    /dev/sdb1:
      selected: false
      size: 28 GB
      unavailable: Disk mounted at /mnt
    /dev/sdc:
      selected: true
      size: 1000 GB
    /dev/sr0:
      selected: false
      size: 4 MB
  id: 10.1.1.5
  prereqs:
    CPU:
      required: x86_64
      state: VALID
      value: x86_64
    Cluster Admin:
      required: present
      state: VALID
      value: mapr
    Disks:
      required: /dev/fd0, /dev/sr0, /dev/sdc
      state: VALID
      value: /dev/sdc
    Distribution:
      required: SLES,Suse,CentOS,RedHat,Ubuntu
      state: VALID
      value: Ubuntu 14.04
    Free /:
      required: 10 GB
      state: VALID
      value: 27.5 GB
    Free /opt:
      required: 128 GB
      state: WARN
      value: 27.5 GB
    Free /tmp:
      required: 10 GB
      state: VALID
      value: 27.5 GB
    GID:
      required: '1000'
      state: VALID
      value: '1000'
    Hadoop:
      required: absent
      state: VALID
      value: absent
    Home Dir:
      required: present
      state: VALID
      value: /home/mapr
    Hostname:
      required: nodeb
      state: VALID
      value: nodeb
    Internet:
      required: present
      state: VALID
      value: present
    Owner /dev/shm:
      required: uid 0
      state: VALID
      value: uid 0
    Owner /tmp:
      required: uid 0
      state: VALID
      value: uid 0
    Perm /dev/shm:
      required: '1023'
      state: VALID
      value: '01777'
    Perm /tmp:
      required: '1023'
      state: VALID
      value: '01777'
    RAM:
      required: 8 GB
      state: VALID
      value: 14.0 GB
    SWAP:
      required: 1.4 GB
      state: VALID
      value: 2.0 GB
    UID:
      required: '1000'
      state: VALID
      value: '1000'
    Yarn:
      required: absent
      state: VALID
      value: absent
  services: []
  state: CHECK_WARN
  status: Verified with warnings
  valid: false
- completion: 100
  disks:
    /dev/fd0:
      selected: false
      size: 0 Bytes
    /dev/sda1:
      selected: false
      size: 29 GB
      unavailable: Disk mounted at /
    /dev/sdb1:
      selected: false
      size: 28 GB
      unavailable: Disk mounted at /mnt
    /dev/sdc:
      selected: true
      size: 1000 GB
    /dev/sr0:
      selected: false
      size: 4 MB
  id: 10.1.1.4
  prereqs:
    CPU:
      required: x86_64
      state: VALID
      value: x86_64
    Cluster Admin:
      required: present
      state: VALID
      value: mapr
    Disks:
      required: /dev/fd0, /dev/sr0, /dev/sdc
      state: VALID
      value: /dev/sdc
    Distribution:
      required: SLES,Suse,CentOS,RedHat,Ubuntu
      state: VALID
      value: Ubuntu 14.04
    Free /:
      required: 10 GB
      state: VALID
      value: 26.8 GB
    Free /opt/mapr:
      required: 128 GB
      state: WARN
      value: 26.8 GB
    Free /tmp:
      required: 10 GB
      state: VALID
      value: 26.8 GB
    GID:
      required: '1000'
      state: VALID
      value: '1000'
    Hadoop:
      required: absent
      state: VALID
      value: absent
    Home Dir:
      required: present
      state: VALID
      value: /home/mapr
    Hostname:
      required: nodea
      state: VALID
      value: nodea
    Internet:
      required: present
      state: VALID
      value: present
    Owner /dev/shm:
      required: uid 0
      state: VALID
      value: uid 0
    Owner /tmp:
      required: uid 0
      state: VALID
      value: uid 0
    Perm /dev/shm:
      required: '1023'
      state: VALID
      value: '01777'
    Perm /tmp:
      required: '1023'
      state: VALID
      value: '01777'
    RAM:
      required: 8 GB
      state: VALID
      value: 14.0 GB
    SWAP:
      required: 1.4 GB
      state: VALID
      value: 2.0 GB
    UID:
      required: '1000'
      state: VALID
      value: '1000'
    Yarn:
      required: absent
      state: VALID
      value: absent
  services: []
  state: CHECK_WARN
  status: Verified with warnings
  valid: false
- completion: 100
  disks:
    /dev/fd0:
      selected: false
      size: 0 Bytes
    /dev/sda1:
      selected: false
      size: 29 GB
      unavailable: Disk mounted at /
    /dev/sdb1:
      selected: false
      size: 28 GB
      unavailable: Disk mounted at /mnt
    /dev/sdc:
      selected: true
      size: 1000 GB
    /dev/sr0:
      selected: false
      size: 4 MB
  id: 10.1.1.6
  prereqs:
    CPU:
      required: x86_64
      state: VALID
      value: x86_64
    Cluster Admin:
      required: present
      state: VALID
      value: mapr
    Disks:
      required: /dev/fd0, /dev/sr0, /dev/sdc
      state: VALID
      value: /dev/sdc
    Distribution:
      required: SLES,Suse,CentOS,RedHat,Ubuntu
      state: VALID
      value: Ubuntu 14.04
    Free /:
      required: 10 GB
      state: VALID
      value: 27.4 GB
    Free /opt:
      required: 128 GB
      state: WARN
      value: 27.4 GB
    Free /tmp:
      required: 10 GB
      state: VALID
      value: 27.4 GB
    GID:
      required: '1000'
      state: VALID
      value: '1000'
    Hadoop:
      required: absent
      state: VALID
      value: absent
    Home Dir:
      required: present
      state: VALID
      value: /home/mapr
    Hostname:
      required: nodec
      state: VALID
      value: nodec
    Internet:
      required: present
      state: VALID
      value: present
    Owner /dev/shm:
      required: uid 0
      state: VALID
      value: uid 0
    Owner /tmp:
      required: uid 0
      state: VALID
      value: uid 0
    Perm /dev/shm:
      required: '1023'
      state: VALID
      value: '01777'
    Perm /tmp:
      required: '1023'
      state: VALID
      value: '01777'
    RAM:
      required: 8 GB
      state: VALID
      value: 14.0 GB
    SWAP:
      required: 1.4 GB
      state: VALID
      value: 2.0 GB
    UID:
      required: '1000'
      state: VALID
      value: '1000'
    Yarn:
      required: absent
      state: VALID
      value: absent
  services: []
  state: CHECK_WARN
  status: Verified with warnings
  valid: false
groups: []
{% endhighlight %}

<br><br>
<div class="main-explain-area padding-override jumbotron">
  <img src="http://iandow.github.io/img/paypal.png" width="120" style="margin-left: 15px" align="right">
  <p class="margin-override font-override">
    Did you enjoy the blog? Did you learn something useful? If you would like to support this blog please consider making a small donation. Thanks!</p>
  <br>
  <div id="paypalbtn">
    <a class="btn btn-primary btn" href="https://www.paypal.me/iandownard/3.5">Donate via PayPal</a>
  </div>
</div>