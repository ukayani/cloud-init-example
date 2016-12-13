# cloud-init-example

# Overview

The aim of this project is to provide a better understanding of the underlying
functionality of cloud-init and the order in which it executes various formats
provided via `user-data`

The following formats are used:

1. x-shellscript
2. cloud-boothook
3. cloud-config
4. upstart-job
5. x-include-url

See: [Cloud-Init Formats](http://cloudinit.readthedocs.io/en/latest/topics/format.html)

The idea behind this CloudFormation template is to test the order in which
each type of cloud-init data is executed.

# Running

To test the example, you'll need access to create EC2 instances in AWS.

1. Upload the `SampleUserData` script to S3 or some file hosting service
2. Make this file publicly accessible (in S3 change permissions for `everyone` to `open/download`)
3. Load the `EC2Test.template` into the Cloud Formation console
4. Specify the URL to the file in step 2 via the `UserDataUrl` param
5. Specify the VPC, Subnet and Keypair with which to create the instance


# Sample User-Data

The user-data in our example is using the multi-part format which allows us
to pass in multiple different formats for cloud-init in one user-data

Think of multipart as a collection of files with the file boundaries being
at the defined `boundary`, ex. ==BOUNDARY==

The following is the userdata for our test:

**UserDataA**

```bash
From nobody Fri Dec  9 00:31:04 2016
Content-Type: multipart/mixed; boundary="==BOUNDARY=="
MIME-Version: 1.0

--==BOUNDARY==
MIME-Version: 1.0
Content-Type: text/x-shellscript; charset="us-ascii"
#!/bin/bash
echo "Part1" >> /var/log/order.log
echo "Executing: Part1"
--==BOUNDARY==
MIME-Version: 1.0
Content-Type: text/x-shellscript; charset="us-ascii"
#!/bin/bash
echo "Part2" >> /var/log/order.log
echo "Executing: Part2"
--==BOUNDARY==
MIME-Version: 1.0
Content-Type: text/x-include-url; charset="us-ascii"
#!/bin/bash

[http://link-to-sample-user-data]
--==BOUNDARY==--
```

The following is the content of the `SampleUserData` which should be uploaded to a public location

```bash
From nobody Fri Dec  9 00:31:04 2016
Content-Type: multipart/mixed; boundary="===============2678395050260980330=="
MIME-Version: 1.0

--===============2678395050260980330==
MIME-Version: 1.0
Content-Type: text/x-shellscript; charset="us-ascii"
#!/bin/bash

echo "Part 3" >> /var/log/order.log

--===============2678395050260980330==
MIME-Version: 1.0
Content-Type: text/cloud-boothook; charset="us-ascii"

#cloud-boothook
echo "Part 4" >> /var/log/order.log

--===============2678395050260980330==
MIME-Version: 1.0
Content-Type: text/cloud-config; charset="us-ascii"

#cloud-config
runcmd:
  - echo "Part 5 runcmd" >> /var/log/order.log
bootcmd:
  - echo "Part 5 bootcmd" >> /var/log/order.log

--===============2678395050260980330==
MIME-Version: 1.0
Content-Type: text/upstart-job; charset="us-ascii"

#upstart-job
description "Sample upstart job"
author "Test"

start on stopped rc RUNLEVEL=[345]

script
	echo "Part 6" >> /var/log/order.log
end script
--===============2678395050260980330==
MIME-Version: 1.0
Content-Type: text/upstart-job; charset="us-ascii"

#upstart-job
description "Sample upstart job which triggers at same time as rc scripts (This won't execute on first boot)"
author "Test"

start on starting rc RUNLEVEL=[345]

script
	echo "Part 7" >> /var/log/order.log
end script
--===============2678395050260980330==--

```

Lets break the multipart format down into its cloud-init types and describe what they do

### Part 1
- Format: x-shellscript
- Action: append partname to `/var/log/order.log`

### Part 2
- Format: x-shellscript
- Action: append partname to `/var/log/order.log`

### Part X
- Format: x-include-url
- Action: Download a userdata compatible file (SampleUserData) and parse it like userdata. 
- Description: Place links per line of the file and cloud-init will download these files and subject 
them to the standard user data parsing logic

##Inside SampleUserData

Part X links to our SampleUserData file which contains the following parts:

### Part 3
- Format: x-shellscript
- Action: append partname to `/var/log/order.log`

### Part 4
- Format: cloud-boothook
- Action: append partname to /var/log/order.log 
- Description: This type will place the content in a script and execute it immediately, so it is one of the the first parts to run

### Part 5
- Format: cloud-config
- Action: Executes cloud-config yaml formatted instructions (`runcmd` and `bootcmd`), these commands append to `/var/log/order.log`

### Part 6
- Format: upstart-job
- Action: Installs an upstart job config to `/etc/init`, this job appends to `/var/log/order.log`
- Description: The line `start on stopped rc RUNLEVEL=[345]` tells upstart to run this job when the `rc` job finishes (explained below)

### Part 7
- Format: upstart-job
- Action: Installs an upstart job config to `/etc/init`, this job appends to `/var/log/order.log`
- Description: The line `start on starting rc RUNLEVEL=[345]` tells upstart to run this job when the `rc` job is starting up

# Results

As we can see the cloud-init types in our example append their respective part name to a log in `/var/log/order.log`

Based on the order in which the part names appear, we can see the order in which cloud-init 
executes the parts.

The resulting `/var/log/order.log` is as follows after first boot:

```
Part 4
Part 5 bootcmd
Part1
Part2
Part 3
Part 5 runcmd
Part 6
```

## Order of Execution

From this we can derive the following order of execution:

1. **cloud-boothook** 
    - This config type is the earliest to run
2. **cloud-config bootcmd**
    - This bootcmd instruction is treated similarly to cloud-boothook so it runs early
3. **x-shellscript**
    - Shellscript types are executed after boothooks and bootcmd
4. **cloud-config runcmd**
    - This runcmd instruction is treated similarly to x-shellscript in terms of order of execution
5. **upstart-job**
    - This upstart job is last to execute since it hooks on the completion of the rc task (more on rc below)
    
Moreover, the reason `cloud-boothook` ran before cloud-config `bootcmd` is because of their ordering within the userdata file.
Within the user-data file, the `boothook` is defined as part 4 and `bootcmd` is defined part 5.
    
Similarly, `x-shellscript` ran before cloud config `runcmd` because of the ordering within the userdata file.
    
So, a general order of execution for cloud-init types is:

1. `cloud-boothook` or `bootcmd` (cloud-config)
2. `x-shellscript`  or `runcmd` (cloud-config)
3. `upstart-job`

Notice, we had a part 7 upstart-job but it didn't show up in our order.log file (more on this below). 
  
## After rebooting

After rebooting the machine, lets see which cloud-init types are executed:

`/var/log/order.log`

```
Part 4
Part 5 bootcmd
Part1
Part2
Part 3
Part 5 runcmd
Part 6
Part 7
Part 4
Part 5 bootcmd
Part 6
```
  
We can see that after reboot the following parts ran:
  
1. Part 7 - upstart-job on `starting rc`
2. Part 4 - cloud-boothook  
3. Part 5 - cloud-config bootcmd
4. Part 6 - upstart-job on `stopped rc`

We can conclude that the only cloud-init types which execute during
every standard boot are `upstart-job`, `cloud-boothook`, and `cloud-config bootcmd`. 
Everything else from our list ran `only once`

# upstart, cloud-init and rc

## What is upstart?

Linux/unix platforms typically have a process that runs once the system boots up. This process is responsible
for running all other processes on the system. Because it is the parent process of all other processes,
it is designated PID 1. This process is always running so it is a daemon process (running in the background). 

Traditionally, this process was called `init` but it has since been replaced by better alternatives such as `upstart` and `systemd`.

The reason init was replaced is because it runs child processes in sequential order thereby increasing the time it takes
for the system to boot. New alternative such as `upstart` is an event-based init system that can asynchronously run processes.

With `upstart` you can have processes started when certain events are fired, such as another process starting or stopping

## `/etc/init`

With `upstart` as we saw with the user-data `upstart-job` config, the resulting file is stored by cloud-init in the `/etc/init` folder.
This folder contains all upstart jobs, which can be long lived (daemon) or short lived (task). 

Lets examine one of the upstart-jobs we defined in our user-data:

```bash
#upstart-job
description "Sample upstart job"
author "Test"

start on stopped rc RUNLEVEL=[345]

script
	echo "Part 6" >> /var/log/order.log
end script
```

All upstart jobs have a line similar to the following:
`start on stopped rc RUNLEVEL=[345]`

Ignoring the `RUNLEVEL=[345]`, the line is fairly self-explanatory.

It typically takes the form `[start/stop] on [event] [jobname]`
where: 
- `start/stop` is action to take on the current job. 
- `event` is an event emitted by (possibly) another job or the system
- `jobname` is the name of another job (job names are defined by the filename of the job in `/etc/init`)

In our case, we are telling upstart to run the `script` portion of our job
when the job named `rc` has *stopped*.

**What exactly does the rc job do?**

The rc job triggers a process which runs services defined in `/etc/rc.d/rc{X}.d`
where **X** is the current system [run level](https://en.wikipedia.org/wiki/Runlevel). In traditional System V style linux systems, runlevels were
categorizations of the state of a machine ie. shutting down, rebooting, running with disk/network initialized etc.

The default run-level in Amazon Linux is **3**, which means the system is running in a normal state. 
So when the rc job runs (in run-level 3), it runs scripts/services in `/etc/rc.d/rc3.d`

As it turns out, one of the scripts/services defined in `/etc/rc.d/rc3.d` is **`cloud-init`** !

## Full circle: When does cloud-init run?

So as we saw in the previous section, `upstart` contains a job called `rc` which eventually executes
`cloud-init` as part of the many scripts it executes.

Once cloud init runs, it downloads our user-data (which we pass into our EC2 instance definition). 
It then parses this user data and stores the parts in various locations.

1. Stores upstart-jobs in `/etc/init`
2. Stores shellscripts and runcmd in `/var/lib/cloud/instance/scripts`
3. Stores boothooks in `/var/lib/cloud/instance/boothooks`

For boothooks/bootcmd, cloud-init runs these immediately hence the reason they show up early in our `order.log`

It then executes all shellscripts/runcmd.

Once `cloud-init` finishes executing and all other scripts for the given run level are done executing, the upstart `rc`
job emits the `stopped` event. 

If we remember **Part 6**, the upstart-job had the following line: `start on stopped rc RUNLEVEL=[345]`
When the `rc` job completes its execution and emits the `stopped` event, upstart will start our **part 6** upstart-job.

This explains why we have *Part 6* printed last in our `order.log` after first boot.
 
### So why doesn't *Part 7* execute on first boot?
 
When `upstart` runs the `rc` job it emits the `starting` event for `rc` and then `rc` proceeds to
execute the scripts/services for the given run level. As we mentioned above, `cloud-init` is one of the
services run by `rc`. Because cloud-init starts after the `starting` event for `rc` is emitted, it would be
impossible for the **Part 7** upstart-job to be executed on first boot; the event which it depends on occurs before the job is created.

Upon rebooting the system, we see that the first part to run is `Part 7` (as seen in order.log). This makes sense, since upstart
would start this job before `cloud-init` is run.

# TLDR;

So, a general order of execution for cloud-init types is:

1. `cloud-boothook` or `bootcmd` (cloud-config)
    - Run every time the system boots
2. `x-shellscript`  or `runcmd` (cloud-config)
    - Runs only one time (first boot)
3. `upstart-job`
    - Runs every time system boots

**cloud-init** is bootstrapped in the following manner:

**Upstart** --[runs]--> **rc** ---[runs]--> **cloud-init** 

Because the upstart `rc` job runs `cloud-init`, any upstart jobs created by `cloud-init` which are intended to execute on first boot must trigger on events which occur on or after
`rc` stopped.

# Resources

- [Upstart Primer](http://upstart.ubuntu.com/getting-started.html)
- [Cloud Init Formats](http://cloudinit.readthedocs.io/en/latest/topics/format.html)
- [History of Init](https://en.wikipedia.org/wiki/Init)



