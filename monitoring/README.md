# OpenShift Monitoring Scripts

This folder contains a set of scripts related to monitoring / checking of OpenShift or OpenShift-deployed resources.

## check_urls

This script dynamically pulls in the set of routes across all projects in OpenShift and checks them.

Requirements:

* python 3+
* requests lib (eg. ```pip install requests```)

To run:

```
./check_urls.sh
```

There will be "false negatives" due to orphaned routes or broken apps, but the purpose of this script is mainly to check whether the router is experiencing problems, not to test specific apps.

Sample output (names, etc. changed to protect the guilty...):

```
=================
Starting a run.
=================
time='2017-01-07 10:08:45.218158',error=Failed with Exception, project=xyz, route_name=xyz, route=https://xyz.gov.bc.ca, exception='('Connection aborted.', RemoteDisconnected('Remote end closed connection without response',))''
time='2017-01-07 10:07:12.953645', error=Failed with bad status, project=pqr, route_name=pqr, route=http://pqr.gov.bc.ca, response_code=503
=================
Run complete... Grabbing a coffee before next run...
=================
```

## prune.sh

This script will prune the builds, deployments, and images from the registry. Set up as a cron on one of the master servers.

To create the pruner user for the script

```
oc project default
echo '{"kind":"ServiceAccount","apiVersion":"v1","metadata":{"name":"pruner"}}' | oc create -f -
oadm policy add-cluster-role-to-user system:image-pruner system:serviceaccount:default:pruner
oadm policy add-cluster-role-to-user cluster-admin system:serviceaccount:default:pruner
secret=$(oc get sa/pruner --template '{{index .secrets 0 "name"}}')
token=$(oc get secret $secret --template '{{.data.token}}' | base64 -d)
oc login --token $token
oc config view --minify --flatten > pruner.kubeconfig
```

Script will have no output, but will email on error.

## List of things to be monitored

### critical (alert on issues)

- CPU - reservations per node (alert at 80%?) `oc describe nodes -l region=app | grep -A4 'Allocated resources:' | grep '%' | awk '{print $2, $6}'`
- Gluster thin pool usage
- Gluster volumes for registry, logging, and metrics
- Docker Pool usage - Currently monitored on all hosts with `lvs --noheadings -o data_percent /dev/docker-vg/docker-pool | tr -d [:space:]` and critical alerts at 90% (where docker will break)

### warning (needs a warning to the groups)

- Build times (what should be the threshold?)
- Number of items needing pruning. This can also let us know if the prune cron is failing

### notice (more things to track)

- Disk - How many of each size of persistant volumes are available (Daily/Weekly report?) `oc get pv | grep Available | awk '{print $2}' | sort | uniq -c`

### trending (long term perfomance data)

CPU/MEM/IOPS per project

CPU/MEM/IOPS per pod
