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

As we can see in the files user data part in our example appends the part name to a log in `/var/log/order.log`

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

1. cloud-boothook 
    - This config type is the earliest to run
2. cloud-config bootcmd
    - This bootcmd instruction is treated similarly to cloud-boothook so it runs early
3. x-shellscript
    - Shellscript types are executed after boothooks and bootcmd
4. cloud-config runcmd
    - This runcmd instruction is treated similarly to x-shellscript in terms of order of execution
5. upstart-job
    - This upstart job is last to execute since it hooks on the completion of the rc task (more on rc below)