# How to test the Kube Storage Version Migrator

## Background

This repository outlines the steps I took to better understand how the [kube-storage-version-migrator](https://github.com/kubernetes-sigs/kube-storage-version-migrator) migrates CRs to newer versions.

I will use three versions of the CronTab CRD to familiarize myself with the `kube-storage-version-migrator`. Let's look at the spec of all three CRD versions:


| CRD Spec Field | v1 | v2 | v3 |
|----------------|----|----|----|
| cronSpec       | x  | x  | x  |
| replicas       | x  | x  | x  |
| image          | x  |    |    | <-- Notice the image field was removed in v2.
| newRequirement |    |    | x  | <-- Notice the newRequirement field was added in v3.

Notes:
- The `spec.image` field was removed in v2.
- The `spec.newRequirement` field was added in v3 and is a required field.

## Major Takeaways

When creating a CronTab CR at v1 with the image and cronSpec fields set:

- The spec.Image value is lost when migrating the CR from v1 to v2.
- The CR will fail to migrate from v2 to v3 because the spec.newRequirement field is not set.

## Prereqs

- etcdctl: https://github.com/etcd-io/etcd/releases
- docker: https://docs.docker.com/get-docker/
- kind: https://kind.sigs.k8s.io/docs/user/quick-start/#installation
- jq: https://stedolan.github.io/jq/

## Set Up your Kind Cluster

```bash
$ kind create cluster --config kind-config-etcd.yaml 
```

## Copy ETCD Certificates

```bash
docker cp kind-control-plane:/etc/kubernetes/pki/etcd/ca.crt etcd-certificates/ca.crt
docker cp kind-control-plane:/etc/kubernetes/pki/etcd/peer.crt etcd-certificates/peer.crt
docker cp kind-control-plane:/etc/kubernetes/pki/etcd/peer.key etcd-certificates/peer.key
export ETCDCTL_CACERT=./etcd-certificates/ca.crt 
export ETCDCTL_CERT=./etcd-certificates/peer.crt 
export ETCDCTL_KEY=./etcd-certificates/peer.key

# Check that you can run the etcdctl get command successfully
$ etcdctl get /registry  --prefix=true | echo $!
0
```

## Introduce the crontab CRD and create a v1 CR

```bash
k apply -f samples/v1.crd.yaml
k apply -f samples/cr.yaml
$ etcdctl get /registry/stable.example.com/crontabs/default/my-new-cron-object | sed 1d | jq .apiVersion
"stable.example.com/v1"

# Check that the spec.Image field exists:
$ etcdctl get /registry/stable.example.com/crontabs/default/my-new-cron-object | sed 1d | jq .spec
{
  "cronSpec": "* * * * */5",
  "image": "my-awesome-cron-image"
}
```
## Update the CronTab CRD to v2 and make sure that the CR is still at v1

```bash
k apply -f samples/v2.crd.yaml
$ etcdctl get /registry/stable.example.com/crontabs/default/my-new-cron-object | sed 1d | jq .apiVersion
"stable.example.com/v1"
```

## Start the kube-storage-version-migrator

```bash
# I stored the manifests that I built to make this easy to use, but feel free to build/deploy the kube-storage-version-migrator on your own
k apply -f manifests

# Wait for grep to succeed...
$ k get storageversionmigrations | grep crontab
crontabs.stable.example.com-vxsvk

# Check that the CR is at the new version
$ etcdctl get /registry/stable.example.com/crontabs/default/my-new-cron-object | sed 1d | jq .apiVersion
"stable.example.com/v2"

# Note that the image was dropped from the CR:
$ etcdctl get /registry/stable.example.com/crontabs/default/my-new-cron-object | sed 1d | jq .spec
{
  "cronSpec": "* * * * */5"
}

# Upgrade the CRD to v3
k apply -f samples/v3.crd.yaml

# Wait for a new crontab storageversionmigration resource to appear, can be spec up by deleting the trigger pod in the kube-system namespace
k get storageversionmigrations | grep crontab

# Check the status of the job
k get storageversionmigrations crontabs.stable.example.com-rrfhm -o json | jq .status
{
  "conditions": [
    {
      "lastUpdateTime": "2021-06-11T16:56:50Z",
      "status": "True",
      "type": "Running"
    }
  ]
}

# Check that the CR does not go to v3
$ etcdctl get /registry/stable.example.com/crontabs/default/my-new-cron-object | sed 1d | jq .apiVersion
"stable.example.com/v2"

# Check Migration pod logs
0611 16:59:06.352996       1 core.go:221] migration of my-new-cron-object, in the default namespace, will be retried: CronTab.stable.example.com "my-new-cron-object" is invalid: spec.newRequirement: Required value
W0611 16:59:06.370074       1 core.go:221] migration of my-new-cron-object, in the default namespace, will be retried: CronTab.stable.example.com "my-new-cron-object" is invalid: spec.newRequirement: Required value
```

