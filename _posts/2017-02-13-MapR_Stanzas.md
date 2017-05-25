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
    ssh_password: MaprRocks!
    ssh_method: PASSWORD
    ssh_port: 22

Then validate the config file like this:

  ```/opt/mapr/installer/bin/mapr-installer-cli check -u mapr:q7wp4XaAQpvdn@localhost -n -t /opt/mapr/installer/examples/ian01.yaml```

If the check succeeds, you will see no output and a return status of 0.

For your reference I've included the YAML file that I exported from my test cluster below.

{% highlight yaml %}
environment:
  mapr_core_version: 5.2.0
config:
  admin_id: mapr
  cluster_admin_create: false
  cluster_admin_gid: 1000
  cluster_admin_group: mapr
  cluster_admin_id: mapr
  cluster_admin_uid: 1000
  cluster_id: '3703723865331276875'
  cluster_name: demo.cluster.com
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
  dial_home_id: '720540929'
  disk_format: true
  disk_stripe: 3
  disks:
  - /dev/sdc
  elasticsearch_path: /opt/mapr/es_db
  environment: azure
  hosts:
  - nodea.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodeb.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodec.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
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
  mapr_id: '64468'
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
        password: q7wp4XaAQpvdn
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
        password: q7wp4XaAQpvdn
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
        label: Use existing MySQL server
        name: oozie
        password: q7wp4XaAQpvdn
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
  ssh_password: q7wp4XaAQpvdn
  ssh_method: PASSWORD
  ssh_port: 22
  state: COMPLETED
hosts:
- disks:
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
      size: 7 TB
    /dev/sdd:
      selected: false
      size: 7 TB
    /dev/sr0:
      selected: false
      size: 4 MB
  id: nodec.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
- disks:
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
      size: 7 TB
    /dev/sr0:
      selected: false
      size: 4 MB
  id: nodeb.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
- disks:
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
      size: 7 TB
    /dev/sr0:
      selected: false
      size: 4 MB
  id: nodea.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
groups:
- hosts:
  - nodec.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodea.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodeb.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  label: CONTROL
  services:
  - mapr-zookeeper-5.2.0
- hosts:
  - nodea.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodeb.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  label: MULTI_MASTER
  services:
  - mapr-webserver-5.2.0
  - mapr-resourcemanager-5.2.0
- hosts:
  - nodec.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  label: MASTER
  services:
  - mapr-hiveserver2-1.2
  - mapr-hivewebhcat-1.2
  - mapr-hivemetastore-1.2
  - mapr-hue-3.10.0
  - mapr-historyserver-5.2.0
  - mapr-kibana-4.5.1
  - mapr-grafana-3.1.1
  - mapr-hbasethrift-1.1
  - mapr-mysql
  - mapr-spark-historyserver-2.0.1
  - mapr-oozie-4.2.0
  - mapr-httpfs-1.0
- hosts:
  - nodec.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodea.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodeb.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  label: MONITORING_MASTER
  services:
  - mapr-opentsdb-2.2.1
  - mapr-elasticsearch-2.3.3
- hosts:
  - nodec.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodea.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodeb.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  label: DATA
  services:
  - mapr-hbase-rest-1.1
  - mapr-nodemanager-5.2.0
  - mapr-drill-1.9
- hosts:
  - nodec.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodea.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodeb.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  label: CLIENT
  services:
  - mapr-spark-client-2.0.1
  - mapr-hive-client-1.2
  - mapr-hbase-1.1
  - mapr-asynchbase-1.7.0
  - mapr-kafka-0.9.0
- hosts:
  - nodec.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodea.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  - nodeb.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  label: DEFAULT
  services:
  - mapr-core-5.2.0
  - mapr-collectd-5.5.1
  - mapr-fileserver-5.2.0
  - mapr-fluentd-0.14.00
- hosts:
  - nodea.seqgoshpp54uhcdmhan2f1m5ie.bx.internal.cloudapp.net
  label: COMMUNITY_EDITION
  services:
  - mapr-cldb-5.2.0
  - mapr-nfs-5.2.0
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