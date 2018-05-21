# pcf-capi-troubleshoot-playbook

## Introduction
Cloud controller is a Ruby Rails application and is one of busiest component within a PCF deployment given it is responsible for all API calls.
In current supported PCF versions (1.12, 2.0, 2.1), cloud_controller_ng is the actual process running in cloud controller VM.

```
ps aux | grep cloud_controller_ng

vcap     28800  0.1  4.7 1276348 193192 ?      S<l  Apr03   2:34 ruby /var/vcap/packages/cloud_controller_ng/cloud_controller_ng/bin/cloud_controller -c /var/vcap/jobs/cloud_controller_ng/config/cloud_controller_ng.yml
```

This process is monitored by monit and has hardcoded CPU / Memory usage limit.

```
cat /var/vcap/monit/job/0006_cloud_controller_ng.monitrc


check process cloud_controller_ng
  with pidfile /var/vcap/sys/run/cloud_controller_ng/cloud_controller_ng.pid
  start program "/var/vcap/jobs/cloud_controller_ng/bin/cloud_controller_ng_ctl start"
        as uid vcap and gid vcap
  stop program "/var/vcap/jobs/cloud_controller_ng/bin/cloud_controller_ng_ctl stop"
        as uid vcap and gid vcap
  group vcap
  if totalmem > 3500 Mb for 3 cycles then alert
  if totalmem > 3500 Mb for 15 cycles then exec "/var/vcap/jobs/cloud_controller_ng/bin/restart_drain"
  if totalmem > 3750 Mb for 3 cycles then exec "/var/vcap/jobs/cloud_controller_ng/bin/restart_drain"

  if failed host 172.16.0.20 port 9022 protocol http

      and request '/v2/info'
      with timeout 60 seconds for 5 cycles
  then restart
```

This threshold is hardcoded in purpose as engineer team believe in normal situation cloud_controller_ng should never use more than 3.5GB memory, otherwise it may indicate a possible memory leak / bloat issue.

However in case cloud_controller_ng’s memory usage exceeds 3.5GB threshold, it will be killed and restarted by monit.

This could cause issues in customer production environment as all ongoing tasks on this cloud controller VM (other cloud controller may still work) will fail. And this playbook is aimed to demonstrate a general troubleshooting process for such issue.

## Troubleshooting memory dump

We are going to use 3rd party tool [heapy](https://github.com/schneems/heapy) to dump and analysis the heap memory.

Edit the cloud controller runner (/var/vcap/jobs/cloud_controller_ng/packages/cloud_controller_ng/cloud_controller_ng/lib/cloud_controller/runner.rb) and apply the following changes.

```
diff --git a/lib/cloud_controller/runner.rb b/lib/cloud_controller/runner.rb
index 03895faf0..c05047ef3 100644
--- a/lib/cloud_controller/runner.rb
+++ b/lib/cloud_controller/runner.rb
@@ -14,6 +14,8 @@ require 'cloud_controller/metrics/request_metrics'
 
require_relative 'message_bus_configurer'
 
+require 'objspace'; ObjectSpace.trace_object_allocations_start
+
module VCAP::CloudController
class Runner
attr_reader :config_file, :insert_seed_data
@@ -122,6 +124,24 @@ module VCAP::CloudController
EM.add_timer(0) do
logger.warn('Collecting diagnostics')
collect_diagnostics
+
+ File.open("/tmp/heap_dump_before_gc", "w") do |file|
+ ObjectSpace.dump_all(output: file)
+ end
+
+ File.open("/tmp/gc_stat_before_gc", "w") do |file|
+ file.write(GC.stat)
+ end
+
+ GC.start
+
+ File.open("/tmp/heap_dump_after_gc", "w") do |file|
+ ObjectSpace.dump_all(output: file)
+ end
+
+ File.open("/tmp/gc_stat_after_gc", "w") do |file|
+ file.write(GC.stat)
+ end
end
end
end
```

Note that we are taking dump with `USR1` signal, we also take dump before and after GC (garbage collection). You can make changes to such optionals and take dump at your convenience.

Now restart cloud_controller and reproduce the issue, then kill the cloud controller process with -USR1 signal.

```
Kill -USR1 <pid_of_cc>
```

Check the dump files.

```
cloud_controller/bbe28341-228c-4367-97f3-4fe9f212f8ea:/tmp$ ls -ltr
total 2135804
-rw-r--r-- 1 vcap vcap 2081478156 Apr 19 08:49 heap_dump_before_gc
-rw-r--r-- 1 vcap vcap        780 Apr 19 08:49 gc_stat_before_gc
-rw-r--r-- 1 vcap vcap  105550568 Apr 19 08:50 heap_dump_after_gc
-rw-r--r-- 1 vcap vcap        769 Apr 19 08:50 gc_stat_after_gc
```

Dump files are basically json output of all objects with their memory usage information. 

```
cloud_controller/bbe28341-228c-4367-97f3-4fe9f212f8ea:/tmp$ more heap_dump_before_gc
{"type":"ROOT", "root":"vm", "references":["0x5621c2caa230", "0x5621c6d335e0", "0x5621c6d334a0", "0x5621c6d33360", "0x5621c6d33270", "0x5621c6d33180", "0x5621c6d33090", "0x5621c6d32fa0", "0x5621c6d32e88", "0x5621c6d32d98", "0x5621c6d32ca8
", "0x5621c6d32b68", "0x5621c77f3ed0", "0x5621c77f3d90", "0x5621c77f3ca0", "0x5621c77f3b60", "0x5621c77f3a70", "0x5621c77f3930", "0x5621c77f37f0", "0x5621c77f3638", "0x5621c77f34a8", "0x5621c2ca8250", "0x5621c2cdbf88", "0x5621c2cb1968", "
0x5621c2cb1918", "0x5621c440f510", "0x5621c2cb1850", "0x5621c2cb1828", "0x5621c2cd6650", "0x5621c2cdbfb0", "0x5621c58b34d0", "0x5621c58b34f8", "0x5621c6d9d5f8", "0x5621c6d9d580", "0x5621c6d9d508", "0x5621c6d9d710", "0x5621c2caa258"]}
{"type":"ROOT", "root":"machine_context", "references":["0x5621c2caa258", "0x5621c58b3db8", "0x5621c2caa258", "0x5621c58b3db8", "0x5621c2cae150", "0x5621c58b3db8", "0x5621c9b64040", "0x5621c9b64090", "0x5621c9b64090", "0x5621c58b3db8", "0
x5621c2cae0b0", "0x5621c58b3db8", "0x5621c2cae0b0", "0x5621c9b64950", "0x5621c2cae150", "0x5621c737bba0", "0x5621c58b33b8", "0x5621c764a9f8", "0x5621c2cc3988", "0x5621c764a9f8", "0x5621c2cae150", "0x5621c4044700", "0x5621c9b649a0", "0x562
1c9b61750", "0x5621c764a9f8", "0x5621c3ff7d38", "0x5621c2cd9648", "0x5621c2cd9648", "0x5621c2cd40d0", "0x5621c2cba450", "0x5621c2cd9a30", "0x5621c2cb8a88", "0x5621c9b640b8", "0x5621c2cd40d0", "0x5621c7379828", "0x5621c74938a8", "0x5621c9b
64090", "0x5621c9b64090", "0x5621c73796e8", "0x5621c9b640b8", "0x5621c73796e8", "0x5621c9b640e0", "0x5621c9b640e0", "0x5621c9b64090", "0x5621c7379828", "0x5621c74938a8", "0x5621c9b64090", "0x5621c73796e8", "0x5621c2cbc2f0", "0x5621c2cbbc8
8", "0x5621c2cb8a88", "0x5621c9b64090", "0x5621c58b34f8", "0x5621c9b64090", "0x5621c2cbc2f0", "0x5621c9b64090", "0x5621c2cb8a88", "0x5621c2cb5018", "0x5621c2cb8a88", "0x5621c9b64090", "0x5621c2cb8a88", "0x5621c2cbbc88", "0x5621c2cb8a88",
"0x5621c58b3480", "0x5621c9b640b8", "0x5621c7393598", "0x5621c2cbc2f0", "0x5621c2cbbc88", "0x5621c2cb8a60", "0x5621c74938a8", "0x5621c737bba0", "0x5621c58b33b8", "0x7f6e270f7318", "0x5621c2cb8a88", "0x5621c44a5448", "0x5621c2cb13f0", "0x5
621c9b62ce0", "0x5621c75a0480", "0x5621c9b62560", "0x5621c3888098", "0x5621c41781f8", "0x5621c6d9cef0", "0x5621c690d470", "0x5621c2cb5018", "0x5621c2cb8a88", "0x5621c2cb13f0", "0x5621c4543ad0", "0x5621c2cc2308", "0x5621c6d9d508", "0x5621c
2cb13f0", "0x5621c6d9d508", "0x5621c6d33680", "0x5621c9b59708", "0x5621c3eb7cc0", "0x5621c6d9d508", "0x7f6f141e2398", "0x5621c9b63258", "0x5621c9b622e0", "0x5621c58b34f8", "0x5621c2cc4090", "0x5621c3eb7cc0", "0x7f6f141e2398", "0x5621c7379
968", "0x5621c74938a8", "0x7f6f141e2398", "0x5621c2cb13f0", "0x5621c74938a8", "0x5621c764ad68", "0x5621c2cc1930", "0x7f6f141e22f8", "0x5621c2cb13f0", "0x5621c764ad68", "0x7f6f141e22f8", "0x5621c2cb13f0", "0x5621c764ad68", "0x7f6f141e22f8"
, "0x5621c2cb1468", "0x5621c2cb13f0", "0x5621c3eb7838", "0x5621c3eb7cc0", "0x5621c3eb7838", "0x5621c3eb7838", "0x5621c3eb7900", "0x5621c764acf0", "0x5621c3eb7838", "0x5621c3eb7900", "0x5621c3eb7838", "0x5621c3eb7900", "0x5621c2c9f948", "0
x5621c2cb2660", "0x5621c764acf0", "0x5621c2cb2660", "0x5621c3888098", "0x5621c41781f8", "0x5621c3eb7cc0", "0x5621c3eb7cc0", "0x5621c2cb2610", "0x5621c2cb25e8", "0x5621c2cb25e8", "0x5621c2cd7f78", "0x5621c2cd7f50", "0x5621c2cd7f50", "0x562
1c2cd7f28", "0x5621c2cc1160", "0x5621c2e7f3a8", "0x5621c2e7f0b0", "0x5621c2e7f3a8", "0x5621c2e7f3a8", "0x5621c2e7f3a8"]}
```

Obviously this is not ideal so we are going to install 3rd party tool heapy to have a better view.

The installation is simple and easy.

```
gem install heapy
```

Now if we have a look at the dump file, we can see the memory usage of each generation (GC).

```
heapy read heap_dump

Analyzing Heap
==============
Generation: nil object count: 275784, mem: 0.0 kb
Generation:  35 object count: 4512, mem: 468.5 kb
Generation:  36 object count: 16504, mem: 1645.1 kb
Generation:  37 object count: 18783, mem: 2002.5 kb
Generation:  38 object count: 13673, mem: 1500.3 kb
Generation:  39 object count: 15236, mem: 1779.3 kb
Generation:  40 object count: 12458, mem: 1357.9 kb
Generation:  41 object count: 1681, mem: 292.9 kb
Generation:  42 object count: 474, mem: 58.9 kb
```

We can also look at the one particular generation to see what’s inside.
For example the heap dump below shows that mysql2.rb is using most of the memory after generation 100.
And if this is a dump file from a problematic cloud controller then we should suspect something went wrong with the sql library (or sql query).

```
heapy read heap_dump 100

Analyzing Heap (Generation: 100)
-------------------------------

allocated by memory (22437115) (in bytes)
==============================
  21011776  /var/vcap/packages/cloud_controller_ng/cloud_controller_ng/vendor/bundle/ruby/2.3.0/gems/sequel-4.49.0/lib/sequel/adapters/mysql2.rb:256
   1020560  /var/vcap/packages/cloud_controller_ng/cloud_controller_ng/vendor/bundle/ruby/2.3.0/gems/sequel-4.49.0/lib/sequel/model/base.rb:264
    270320  /var/vcap/packages/cloud_controller_ng/cloud_controller_ng/vendor/bundle/ruby/2.3.0/gems/sequel-4.49.0/lib/sequel/dataset/actions.rb:1051
     58216  /var/vcap/packages/cloud_controller_ng/cloud_controller_ng/vendor/bundle/ruby/2.3.0/gems/sequel-4.49.0/lib/sequel/model/associations.rb:2248
      7700  /var/vcap/packages/cloud_controller_ng/cloud_controller_ng/vendor/bundle/ruby/2.3.0/gems/sequel-4.49.0/lib/sequel/model/associations.rb:3060
      6672  /var/vcap/packages/cloud_controller_ng/cloud_controller_ng/vendor/bundle/ruby/2.3.0/gems/sequel-4.49.0/lib/sequel/dataset/query.rb:104
      4320  /var/vcap/packages/cloud_controller_ng/cloud_controller_ng/vendor/bundle/ruby/2.3.0/gems/rack-protection-1.5.3/lib/rack/protection/base.rb:33
      4186  /var/vcap/packages/cloud_controller_ng/cloud_controller_ng/vendor/bundle/ruby/2.3.0/gems/multi_json-1.12.1/lib/multi_json/adapters/yajl.rb:11
```

At this stage, troubleshooting (from support’s point of view) is pretty much done, we know mysql library is using most of the memory and further investigation is required to be done by R&D.

## But There is Always Something More

Sometimes showing the biggest memory consumer is still not enough. For example, in this particular case where mysql library consumed most memory, some may say it is expected to see high memory mysql connector usage in a very busy environment.

How can tell whether this is normal or not?

One more step we can do is to enable the sql debug mode for cloud controller so that we can see the exact query command submitted by cloud controller.

NOTE: Enabling sql debug mode may cause extreme workload to cloud controller and lead to downtime on entire PCF deployment. Please make sure you understand the risk and proceed.


To do so, modify the manifest file of PAS as following.

```
  instance_groups:
  - name: cloud_controller
    jobs:
    - name: cloud_controller_ng
      properties:
        cc:
+         db_logging_level: info
+         log_db_queries: true
```

Then bosh deploy using this new manifest file.

```
bosh2 -e myenv -d cf-4a2f60f74aba4736ff3a deploy /tmp/cf.yml

Using environment '172.16.0.11' as user 'director' (bosh.*.read, openid, bosh.*.admin, bosh.read, bosh.admin)

Using deployment 'cf-4a2f60f74aba4736ff3a'

  instance_groups:
  - name: cloud_controller
    jobs:
    - name: cloud_controller_ng
      properties:
        cc:
+         db_logging_level: "<redacted>"
+         log_db_queries: "<redacted>"

Continue? [yN]: yes

Task 951

Task 951 | 06:32:47 | Preparing deployment: Preparing deployment (00:00:12)
Task 951 | 06:33:11 | Preparing package compilation: Finding packages to compile (00:00:01)
Task 951 | 06:33:12 | Updating instance cloud_controller: cloud_controller/bbe28341-228c-4367-97f3-4fe9f212f8ea (0) (canary) (00:01:27)

Task 951 Started  Mon Apr 23 06:32:47 UTC 2018
Task 951 Finished Mon Apr 23 06:34:39 UTC 2018
Task 951 Duration 00:01:52
Task 951 done
```

Once the deploy completes, we should see the exact mysql query from cloud_controller_ng.log.

Here is one example:

```
{"timestamp":1524464815.9774377,"message":"(0.001510s) SELECT `spaces`.* FROM `spaces` INNER JOIN `spaces_developers` ON (`spaces_developers`.`space_id` = `spaces`.`id`) WHERE (`spaces_developers`.`user_id` = 10008)","log_level":"info","source":"cc.db","data":{"request_guid":"56141a86-7db5-4d88-4852-0510b60d682e::999f129e-ab16-4d31-8275-de416c04ee30"},"thread_id":47145902737240,"fiber_id":47145910709120,"process_id":28736,"file":"/var/vcap/packages/cloud_controller_ng/cloud_controller_ng/vendor/bundle/ruby/2.3.0/gems/sequel-4.49.0/lib/sequel/database/logging.rb","lineno":88,"method":"block in log_each"}
{"timestamp":1524464816.0698578,"message":"(0.002264s) SELECT count(*) AS `count` FROM `spaces` WHERE (`spaces`.`id` IN (SELECT `id` FROM (SELECT * FROM (SELECT * FROM (SELECT `spaces`.`id` FROM `spaces` INNER JOIN `spaces_developers` ON ((`spaces_developers`.`space_id` = `spaces`.`id`) AND (`spaces_developers`.`user_id` = 10008)) UNION SELECT `spaces`.`id` FROM `spaces` INNER JOIN `spaces_managers` ON ((`spaces_managers`.`space_id` = `spaces`.`id`) AND (`spaces_managers`.`user_id` = 10008))) AS `t1` UNION SELECT `spaces`.`id` FROM `spaces` INNER JOIN `spaces_auditors` ON ((`spaces_auditors`.`space_id` = `spaces`.`id`) AND (`spaces_auditors`.`user_id` = 10008))) AS `t1` UNION SELECT `spaces`.`id` FROM `spaces` INNER JOIN `organizations_managers` ON ((`organizations_managers`.`organization_id` = `spaces`.`organization_id`) AND (`organizations_managers`.`user_id` = 10008))) AS `t1`)) LIMIT 1","log_level":"info","source":"cc.db","data":{"request_guid":"9307f4e2-4dd1-4f6b-59c1-a98c5b9cdc10::e30d23df-8d43-4c1c-b30a-470c13d47738"},"thread_id":47145902737360,"fiber_id":47145910157620,"process_id":28736,"file":"/var/vcap/packages/cloud_controller_ng/cloud_controller_ng/vendor/bundle/ruby/2.3.0/gems/sequel-4.49.0/lib/sequel/database/logging.rb","lineno":88,"method":"block in log_each"}
```

Look at the log above and you can see this is a five layer select query where the inner-most one uses inner-join between `spaces` and `spaces_developers`.

In our case, we have 10K users * 250 apps in one space, so that is roughly 25M records.

Last but not least, make sure to revert the change (remove db_logging_level and log_db_queries and re-deploy) once the troubleshooting / investigation is over.


## Appendix - Prepare a test environment

We are using a known memory bloat issue in ERT prior to 1.12.17 to demonstrate how should we troubleshoot in such case.

In short the issue is due to we are checking user permission in a very in-sufficient way. For example when a non-admin user login into apps manager, apps manager need to know which apps does this user have access to.
And query happens in the following sequence.

1) Find all spaces that this user has access to.
2) Find all apps under that space.
3) For each app, find all users that has access to it and see if the current user is there.

With this logic, say if we have 10K users, 250 apps in one space, then we will get 10K user info for 250 times. This will exhaust the CC memory with a single login event.
 
So to reproduce this issue, first we need to create a test environment with 10K users. You can either do it via Ruby console or Mysql (suppose you are using Mysql as CCDB).

I am only listing the steps to create 10K users from Ruby console here.

1) SSH into cloud_controller.
2) Open Ruby Console --  ‘/var/vcap/jobs/cloud_controller_ng/bin/console’
3) Create 10K users and add them as space developer (non-admin). This could take a while so just be patient.
Note: the space and org guide here can be retrieved from `cf space --guid` and `cf org --guid`

```
org = Organization.find(guid: <org-guid-from-earlier>)
space = Space.find(guid: <space-guid-from-earlier>)

10_000.times do
  user = User.create(guid: SecureRandom.uuid)
  org.add_user(user)
  space.reload
  space.add_developer(user)
end
```

You can then ssh into the mysql server node and check it from mysql console as well.

```
MariaDB [ccdb]> select count(*) from users;
+----------+
| count(*) |
+----------+
|    10005 |
+----------+
```

Next we need to create some test apps.

```
space = Space.find(guid: <space-guid-from-earlier>)
250.times do |i|
  app = AppModel.create(name: "app-#{i}", space: space)
  process = app.add_process(type: 'web')
end
```

Now we are ready to go.

Access the apps manager via browser and login, it should take a few seconds or minutes to login depends on the spec of your deployment.

And after you login successfully, the memory usage of cloud controller will sky rocket.






