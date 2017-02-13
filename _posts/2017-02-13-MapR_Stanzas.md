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

    ```/opt/mapr/installer/bin/mapr-installer-cli install -u mapr:MaprRocks\!@localhost -nv -t /opt/mapr/installer/examples/ian01.yaml```

I've attached my YAML to the end of this post for your reference.

What if you don't want to write the stanza YAML file by hand?  Then just install via the webui (e.g. `https://<hostname>:9443`), then export a stanza file with the export command (shown below). Then next time you install you should be able to use that YAML file. For example, here's how I captured the configurations I specified for our Tech Marketing MapR cluster:

    /opt/mapr/installer/bin/mapr-installer-cli export -u mapr:MaprRocks\!@localhost -n --file /opt/mapr/installer/examples/ian01.yaml

That file will not contain your ssh password, so you'll want to add "ssh_password" under the config section, like this:

    ssh_password: MaprRocks!

Then validate the config file like this:

    /opt/mapr/installer/bin/mapr-installer-cli check -u mapr:MaprRocks\!@localhost -n -t /opt/mapr/installer/examples/ian01.yaml

If the check succeeds, you will see no output and a return status of 0.

For your reference I've included the YAML file that I exported from my test cluster below.

{% highlight yaml %}
environment:
  mapr_core_version: 5.2.0
config:
  admin_id: mapr
  cluster_admin_create: false
  cluster_admin_gid: 5000
  cluster_admin_group: mapr
  cluster_admin_id: mapr
  cluster_admin_uid: 5000
  cluster_id: '0000078629867426246'
  cluster_name: my.cluster.com
  data:
    diskStripeDefault: 3
    licenseType: M3
    licenseValidation: INSTALL
    'null': true
    showAdvancedServiceConfig: false
    showAdvancedServiceSelection: false
    showLicenseValidation: true
    showLogs: false
    template: template-05-converged
  dial_home_id: '130502442'
  disk_format: true
  disk_stripe: 3
  disks:
  - /dev/sdb
  - /dev/sdc
  - /dev/sdd
  - /dev/sde
  - /dev/sdf
  elasticsearch_path: /opt/mapr/es_db
  environment: ''
  hosts:
  - tm5
  - tm4
  - tm2
  install_dir: /opt/mapr/installer
  installer_version: 1.4.201612081140
  license_install: false
  license_modules:
  - DATABASE
  - HADOOP
  - STREAMS
  license_type: M3
  links:
    self: https://localhost:9443/api/config
    services: https://localhost:9443/api/config/services
  mapr_id: '00000'
  mapr_name: jklfdsalkj@yahoo.com
  mep_version: '2.0'
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
      version: 5.2.0
    mapr-collectd:
      enabled: true
      version: 5.5.1
    mapr-core:
      enabled: true
      version: 5.2.0
    mapr-drill:
      enabled: true
      version: '1.9'
    mapr-elasticsearch:
      enabled: true
      version: 2.3.3
    mapr-fileserver:
      enabled: true
      version: 5.2.0
    mapr-fluentd:
      enabled: true
      version: 0.14.00
    mapr-grafana:
      enabled: true
      version: 3.1.1
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
      version: 5.2.0
    mapr-hive:
      enabled: true
      version: '1.2'
    mapr-hive-client:
      enabled: true
      version: '1.2'
    mapr-hivemetastore:
      database:
        create: true
        label: Use existing MySQL server
        name: hive
        password: MaprRocks!
        type: MYSQL
        user: hive
      enabled: true
      version: '1.2'
    mapr-hiveserver2:
      enabled: true
      version: '1.2'
    mapr-hivewebhcat:
      enabled: true
      version: '1.2'
    mapr-httpfs:
      enabled: true
      version: '1.0'
    mapr-hue:
      database:
        create: true
        label: Use existing MySQL server
        name: hue
        password: MaprRocks!
        type: MYSQL
        user: hue
      enabled: true
      version: 3.10.0
    mapr-kafka:
      enabled: true
      version: 0.9.0
    mapr-kibana:
      enabled: true
      version: 4.5.1
    mapr-mysql:
      enabled: true
    mapr-nfs:
      enabled: true
      version: 5.2.0
    mapr-nodemanager:
      enabled: true
      version: 5.2.0
    mapr-oozie:
      database:
        create: true
        host: MaprRocks!
        label: Use existing MySQL server
        name: oozie
        password: MaprRocks!
        type: MYSQL
        user: oozie
      enabled: true
      version: 4.2.0
    mapr-opentsdb:
      enabled: true
      version: 2.2.1
    mapr-resourcemanager:
      enabled: true
      version: 5.2.0
    mapr-spark:
      enabled: true
      version: 2.0.1
    mapr-spark-client:
      enabled: true
      version: 2.0.1
    mapr-spark-historyserver:
      enabled: true
      version: 2.0.1
    mapr-webserver:
      enabled: true
      version: 5.2.0
    mapr-yarn:
      enabled: true
      version: 5.2.0
    mapr-zookeeper:
      enabled: true
      version: 5.2.0
  services_version: 1.4.201612081140
  ssh_id: mapr
  ssh_password: MaprRocks!
  ssh_method: PASSWORD
  ssh_port: 22
  state: INSTALLED
hosts:
- disks:
    /dev/sda1:
      selected: false
      size: 803 GB
      unavailable: Disk mounted at /
    /dev/sda2:
      selected: false
      size: 1 KB
    /dev/sda5:
      selected: false
      size: 127 GB
    /dev/sdb:
      selected: true
      size: 931 GB
    /dev/sdc:
      selected: true
      size: 931 GB
    /dev/sdd:
      selected: true
      size: 931 GB
    /dev/sde:
      selected: true
      size: 931 GB
    /dev/sdf:
      selected: true
      size: 931 GB
    /dev/sr0:
      selected: false
      size: 1024 MB
  id: tm2
- disks:
    /dev/sda1:
      selected: false
      size: 803 GB
      unavailable: Disk mounted at /
    /dev/sda2:
      selected: false
      size: 1 KB
    /dev/sda5:
      selected: false
      size: 127 GB
    /dev/sdb:
      selected: true
      size: 931 GB
    /dev/sdc:
      selected: true
      size: 931 GB
    /dev/sdd:
      selected: true
      size: 931 GB
    /dev/sde:
      selected: true
      size: 931 GB
    /dev/sdf:
      selected: true
      size: 931 GB
    /dev/sr0:
      selected: false
      size: 1024 MB
  id: tm5
- disks:
    /dev/sda1:
      selected: false
      size: 803 GB
      unavailable: Disk mounted at /
    /dev/sda2:
      selected: false
      size: 1 KB
    /dev/sda5:
      selected: false
      size: 127 GB
    /dev/sdb:
      selected: true
      size: 931 GB
    /dev/sdc:
      selected: true
      size: 931 GB
    /dev/sdd:
      selected: true
      size: 931 GB
    /dev/sde:
      selected: true
      size: 931 GB
    /dev/sdf:
      selected: true
      size: 931 GB
    /dev/sr0:
      selected: false
      size: 1024 MB
  id: tm4
groups:
- hosts:
  - tm5
  - tm2
  - tm4
  label: CONTROL
  services:
  - mapr-zookeeper-5.2.0
- hosts:
  - tm2
  - tm4
  label: MULTI_MASTER
  services:
  - mapr-webserver-5.2.0
  - mapr-resourcemanager-5.2.0
- hosts:
  - tm5
  label: MASTER
  services:
  - mapr-hivewebhcat-1.2
  - mapr-hiveserver2-1.2
  - mapr-hue-3.10.0
  - mapr-hivemetastore-1.2
  - mapr-historyserver-5.2.0
  - mapr-kibana-4.5.1
  - mapr-grafana-3.1.1
  - mapr-hbasethrift-1.1
  - mapr-mysql
  - mapr-spark-historyserver-2.0.1
  - mapr-oozie-4.2.0
  - mapr-httpfs-1.0
- hosts:
  - tm5
  - tm2
  - tm4
  label: MONITORING_MASTER
  services:
  - mapr-opentsdb-2.2.1
  - mapr-elasticsearch-2.3.3
- hosts:
  - tm5
  - tm2
  - tm4
  label: DATA
  services:
  - mapr-hbase-rest-1.1
  - mapr-nodemanager-5.2.0
  - mapr-drill-1.9
- hosts:
  - tm5
  - tm2
  - tm4
  label: CLIENT
  services:
  - mapr-spark-client-2.0.1
  - mapr-hive-client-1.2
  - mapr-hbase-1.1
  - mapr-asynchbase-1.7.0
  - mapr-kafka-0.9.0
- hosts:
  - tm5
  - tm2
  - tm4
  label: DEFAULT
  services:
  - mapr-core-5.2.0
  - mapr-collectd-5.5.1
  - mapr-fileserver-5.2.0
  - mapr-fluentd-0.14.00
- hosts:
  - tm2
  label: COMMUNITY_EDITION
  services:
  - mapr-cldb-5.2.0
  - mapr-nfs-5.2.0
{% endhighlight %}

