## UpUp - CloudUp & NodeUp

CloudUp and NodeUp are two tools that are aiming to replace kube-up:
the easiest way to get a production Kubernetes up and running.

(Currently work in progress, but working.  Some of these statements are forward-looking.)

Some of the more interesting features:

* Written in go, so hopefully easier to maintain and extend, as complexity inevitably increases
* Uses a state-sync model, so we get things like a dry-run mode and idempotency automatically
* Based on a simple meta-model defined in a directory tree
* Can produce configurations in other formats (currently Terraform & Cloud-Init), so that we can have working
  configurations for other tools also.

## Installation

Install glide from [http://glide.sh/](http://glide.sh/)

Build the code (make sure you have set GOPATH):
```
go get -d k8s.io/kube-deploy
cd ${GOPATH}/src/k8s.io/kube-deploy/upup
make
```

## Bringing up a cluster on AWS

* Ensure you have a DNS hosted zone set up in Route 53, e.g. myzone.com

* Pick a subdomain (or sub-subdomain) of this hosted zone, e.g. kubernetes.myzone.com or dev.k8s.myzone.com  We'll call your subdomain `MYZONE`.

* Set `AWS_PROFILE` (if you need to select a profile for the AWS CLI to work)

* Execute:
```
export MYZONE=<kubernetes.myzone.com>
${GOPATH}/bin/cloudup --v=0 --logtostderr -cloud=aws -zone=us-east-1c -name=${MYZONE}
```

If you have problems, please set `--v=8 --logtostderr` and open an issue, and ping justinsb on slack!

## Build a kubectl file

The upup tool is a CLI for doing administrative tasks.  You can use it to generate the kubectl configuration:

```
export MYZONE=<kubernetes.myzone.com>
${GOPATH}/bin/upup kubecfg generate --state=state --master=api.${MYZONE} --name=${MYZONE} --cloud=aws
```

## Delete the cluster

When you're done, you can also have upup delete the cluster.  It will delete all AWS resources tagged
with the cluster name in the specified region.

```
export MYZONE=<kubernetes.myzone.com>
${GOPATH}/bin/upup delete cluster --region=us-east-1 --cluster-id=${MYZONE} # --yes
```

You must pass --yes to actually delete resources (without the `#` comment!)

## Other interesting modes:

* See changes that would be applied: `${GOPATH}/bin/cloudup --dryrun`

* Build a terraform model: `${GOPATH}/bin/cloudup $NORMAL_ARGS --target=terraform`  The terraform model will be built in `state/terraform`

* Specify the k8s build to run: `-kubernetes-version=1.2.2`

* Try HA mode: `-zone=us-east-1b,us-east-1c,us-east-1d`

* Specify the node size: `-node-size=m4.large`

* Specify the number of nodes: `-node-count=4`

# How it works

Everything is driven by a local configuration directory tree, called the "model".  The model represents
the desired state of the world.

Each file in the tree describes a Task.

On the nodeup side, Tasks can manage files, systemd services, packages etc.
On the cloudup side, Tasks manage cloud resources: instances, networks, disks etc.

## Workaround for terraform bug

Terraform currently has a bug where it can't create AWS tags containing a dot.  Until this is fixed,
you can't use terraform to build EC2 resources that are tagged with `k8s.io/...` tags.  Thankfully this is only
the volumes, and it isn't the worst idea to build these separately anyway.

We divide the 'cloudup' model into two parts: 
* models/proto which sets up the volumes and other data which would be hard to recover (e.g. likely keys & secrets in the near future)
* models/cloudup which is the main cloudup model for configuration everything else

So you don't use terraform for the 'proto' phase (you can't anyway, because of the bug!):

```
export MYZONE=<kubernetes.myzone.com>
${GOPATH}/bin/cloudup --v=0 --logtostderr -cloud=aws -zone=us-east-1c -name=${MYZONE} --model=models/proto
```

And then you can use terraform to do the full installation:

```
export MYZONE=<kubernetes.myzone.com>
${GOPATH}/bin/cloudup --v=0 --logtostderr -cloud=aws -zone=us-east-1c -name=${MYZONE} --model=models/cloudup --target=terraform
```

Then, to apply using terraform:

```
cd state/terraform

terraform plan
terraform apply
```
