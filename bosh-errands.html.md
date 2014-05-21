---
title: BOSH Errands
---

### What is an Errand?

A BOSH Errand is a short lived BOSH job that operator can run multiple times after a deploy has finished. Common use cases include:

* Smoke tests
* Cloud Foundry Service Broker binding and unbinding
* Pushing an app to Cloud Foundry

### How to Deploy an Errand

A BOSH errand has a structure similar to that of regular BOSH job in a BOSH release. An errand job in BOSH release should provide a `run` script that will be executed as part of running the errand.

In the job description section of a deployment manifest, a BOSH errand must have its `lifecycle` property set to `errand`. (See the example below). The resource pool must also provide enough capacity to accommodate an ephemeral VM for the errand.

### Example of an Errand Script

The [cf-mysql release](https://github.com/cloudfoundry/cf-mysql-release) uses a BOSH errand to deregister a service broker by creating the following errand `run` script. Notice that, as in this case, a `run` script may also be templated.

```
#!/bin/bash
set -e
export PATH=$PATH:/var/vcap/packages/ruby/bin
cd /var/vcap/packages/broker-registrar

CF_API_URL='<%= p("cf.api_url") %>'
CF_ADMIN_USERNAME='<%= p("cf.admin_username") %>'
CF_ADMIN_PASSWORD='<%= p("cf.admin_password") %>'
BROKER_NAME='<%= p("broker.name") %>'
BROKER_URL='http://<%= p("broker.host") %>:<%= p("broker.port") %>'
BROKER_USERNAME='<%= p("broker.username") %>'
BROKER_PASSWORD='<%= p("broker.password") %>'

# partially-redacted command for debugging
echo "
bundle exec ./bin/broker-registrar delete
    --cf-address $CF_API_URL
    --cf-username $CF_ADMIN_USERNAME
    --cf-password ********
    --broker-name $BROKER_NAME
    --broker-url $BROKER_URL
    --broker-username $BROKER_USERNAME
    --broker-password ********
"

bundle exec ./bin/broker-registrar delete \
    --cf-address $CF_API_URL                \
    --cf-username $CF_ADMIN_USERNAME        \
    --cf-password $CF_ADMIN_PASSWORD        \
    --broker-name $BROKER_NAME              \
    --broker-url $BROKER_URL                \
    --broker-username $BROKER_USERNAME      \
    --broker-password $BROKER_PASSWORD
```

### Example of an Manifest

```
jobs:
- name: errand
  template: errand
  instances: 1
  lifecycle: errand
  resource_pool: default
  networks:
  - name: default

- name: regular_job
  template: regular_job
  instances: 1
  resource_pool: default
  networks:
  - name: default
```

### How to Run an Errand

Once a release with an errand has been deployed, the errand can be run using the following directive:

`bosh run errand <errand_name>`

In this command, `<errand_name>` is a job name provided in deployment manifest. This will compile all the packages required by the errand job and execute the `run` script provided by the release for the errand.

The output of an errand will be printed on the screen once the errand is finished.

Once the errand is finished executing, the resource pool will be refilled with the extra VM that was used by an errand.

### Canceling an Errand

While errand is running it can be canceled by the following directive:

`bosh cancel task <task_name>`

This will terminate the errand process, release the deployment lock, and refill the resource pool.

### Handling Errand Output

In addition to stdout and stderror, the need exists to handle errand output.  When output is less than 1MB, it  is streamed to the screen. When output is larger than 1MB, it needs to be handle more specifically. By default, output is sent to a log file. These options exist:


`run errand <errand_name> [--download-logs] [--logs-dir  destination_directory]`

```
    Run specified errand
    --download-logs                                    download logs
    --logs-dir destination_directory                   logs download directory
```

Soon, the user will have the option to not save output to a log file and instead stream output > 1MB to the screen. (https://www.pivotaltracker.com/story/show/70384252)

###Hey wait a second, my BOSH Errand logs are gone…###

When an errand completes, the VM that it ran on goes away.  This is the nature of errands.  When the VM goes away, naturally all the logs that the errand created on that VM go away too.

Here is how you handle this situation and get access to the logs.  
* In the errand script, redirect stdout to a logfile so that the logfile can be downloaded.  
* On the BOSH cli, download logs with the option --download-logs.  The logs will be downloaded to your present working directory by default.  This default can  be overridden with the optional  --logs-dir <directory_you_specify> 


### Listing Errands

Coming soon: the directive that will allow to list all errands in a deployment.

### Warnings and Important Tips

* Errands do not have a notion of persistence. The persistent disk migration for errands is not supported.

* As part of `run errand` the BOSH deployment executes the drain script. If the drain script is exercising dynamic drain it might result in unpredicted behavior.

* While an errand is running the director will hold a deployment lock which will prevent all other deploys. Additionally, `cloud check` CLI command will be unavailable and the health monitor will not scan and fix current deployments during the duration of the errand.
