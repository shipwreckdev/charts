# EKS Node Drainer

This chart will create a `Deployment` and a corresponding `DaemonSet` that will monitor EKS nodes, determine when a scaling event is happening due to an autoscaling group change, intervene in the scaling process to allow `Pods` to drain, and then let the process proceed.

## How It Works

* `kube-node-drainer-asg-status-updater` (`Deployment`) constantly polls `Nodes` and looks for instances with the proper cluster tag `kubernetes.io/cluster/(your EKS cluster name)` and with a lifecycle status of `Terminating:Wait` and writes them to a `ConfigMap`:

```bash
`jq -r [.AutoScalingGroups[] | select((.Tags[].Key | contains("kubernetes.io/cluster/(your EKS cluster name)"))) | .Instances[] | select(.LifecycleState == "Terminating:Wait") | .InstanceId] | sort | join(",")`
```

* `kube-node-drainer-ds` (`DaemonSet`) then checks the `ConfigMap` created in the previous step and uses `kubectl drain` on the `Nodes` that are marked.

## Configurable Values

|Parameter|Description|Default|
|-|-|-|
|`clusterName`| The EKS cluster name.|`nil`|
|`drainTimeout`| The timeout value for the `kubectl drain` command that gets run as a part of the `DaemonSet` on marked `Nodes`.|`600s`|
|`imagePullSecret`| Image pull secret to attach  to `ServiceAccount` if pulling from a private repo.|`nil`|
|`awsCliImage`| The image for `aws-cli`.|`nil`|
|`hyperkubeImage`| The image for `hyperkube`.|`nil`|
|`rbac.create`| If set to `true` (default), create `ServiceAccount`, `ClusterRole`, and `ClusterRoleBinding`.|`true`|
|`rbac.serviceAccountAnnotations`| If set, values should be in `foo: bar` annotation form.|`nil`|

## Requirements

Your **autoscaling group** will require a **lifecyle hook** like the following, implemented as an **initial lifecycle hook** in this example:

```hcl
initial_lifecycle_hook {
    name                 = "nodedrainer"
    default_result       = "ABANDON"
    heartbeat_timeout    = 1200
    lifecycle_transition = "autoscaling:EC2_INSTANCE_TERMINATING"
  }
```

The `heartbeat_timeout` in this case should be higher than the `drainTimeout` value set in the chart. This timer will need to be adjusted depending on your use case and how you're managing your infrastructure. This was originally written with Terraform and its timeouts in mind.

## Monitoring and Configuring

It is recommended to watch the `kube-node-drainer-asg-status-updater` `Pod` during testing to verify that the `ConfigMap` is properly created on initial release deployment and updated when there is a scaling event.

You can monitor the `kube-node-drainer-ds` `Pods` (per `Node`) during an autoscaling event to get real time output as to the status of that particular node's draining process.

Keep in mind that only one `status-updater` `Pod` is created per release. This `Pod` will likely be terminated and brought up on another node during an autoscaling event, but this is normal and expected. The `Pod` will pick up where it left off on monitoring and reconfiguring the `ConfigMap` as `Nodes` are drained.

The command run by the `DaemonSet` `Pods` looks like this:

```bash
kubectl drain --ignore-daemonsets=true --delete-local-data=true --force=true --timeout={{ .Values.drainTimeout }} "${NODE_NAME}"
```

Due to this, `DaemonSets` are ignored and the pods related to this release are not impacted in a way that will interrupt the autoscaling process.

## Troubleshooting

* _The autoscaling event times out before my nodes are cleanly drained._ 
  * Raise the `drainTimeout` value and the `heartbeat_timeout` value until you find a combination that works properly. This may need adjusting depending on your cluster size, etc.
* _I'm getting all sorts of permissions errors._ 
  * The `rbac.yaml` file was tweaked to allow the minimum amount of rights required for this to work. You may need to evaluate this for your use case.

## References

The logic for properly draining `Nodes` was inspired by https://github.com/kubernetes-incubator/kube-aws.
