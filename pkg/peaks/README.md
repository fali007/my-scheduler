# PEAKS 

## POC capabilities.
1. Get CPU and memory metrics from loadwatcher.
2. Assume we have power model of node avalilabe (locally). We will use kepler model server later.
3. When ever kubernetes scheduler asks for peaks score for each node to schedule a pod
    1. Calculate power for each node wrt current utilisation using power model. (Metrics available from loadwatcher)
    2. Get pod cpu utilisation (request/limit == null ? default : request/limit)
    3. Predicted node utilisation = node utilisation + pod utilisation.
    4. Calculate power for each node wrt predicted utilisation of each node.
    5. Peaks score for node with least jump in power should be highest.

# Custom scheduler plugin for Kubernetes for energy efficient scheduling.

## Concept

Power characteristics of a computer changes a lot, even machines with identical configurations might have different power to utilisation relationship or the same machine can have different power usage for a variety of workloads. We are trying to introduce these changing power characteristics of a physical machine in Kubernetes scheduling.

By end of this project we are hoping to enhance Kubernetes scheduling to schedule pods on least energy consuming nodes once the pod is scheduled.

### Assumptions and generalisation

1. Pod utilisation - We take pod cpu request/default as cpu usage of the pod.
2. Nature of workload - We do not consider nature of workload of a container while building power model.
3. Power metrics - We are using Kepler power metrics, which we consider as ground truth (Still not convinced). The model gets more accurate if we could get measured power.

### Power model

We collect node cpu and power usage from Kepler. We use an exponential model to fit a power model for these data points.

### Scheduler plugin configuration

We start by configuring load-watcher to get node cpu metrics. We pass load-watcher endpoint as argument in out `KubeSchedulerConfiguration`.

```jsx
apiVersion: kubescheduler.config.k8s.io/v1beta2
kind: KubeSchedulerConfiguration
leaderElection:
  leaderElect: false
profiles:
- schedulerName: Peaks
  plugins:
    preScore:
      disabled:
      - name: '*'
    score:
      disabled:
      - name: '*'
      enabled:
      - name: Peaks
  pluginConfig:
    - name: Peaks
      args:
        watcherAddress: http://172.21.152.81:2020
```

We also implement metrics collector, which gets the node cpu metrics from load-watcher. 

### Scheduler plugin implementation

We want our scheduler to receive score request when ever Kubernetes have a new pod to schedule on cluster. We get pod cpu (requests/limits) requirements from this input request. We are expected to send score - integer value 0 to 100 back as the response - for each node.

```jsx
maximum_power = 1000000
score = {}
for each nodes in cluster {
    current_power_usage = get_power_from_power_model(node_cpu_usage)
    predicted_power_usage = get_power_from_power_model(node_cpu_usage + pod_cpu_usage)
    jump_in_power = predicted_power_usage - current_power_usage
    score[node] = (maximum_power - jump_in_power) * 100 / maximum_power
}
```

** Here `jump_in_power` will be always positive as me use strictly increasing function.


# Build commands of PEAKS

We need to build helper functions using codegen command.

```jsx
./hack/update-codegen.sh
```

This will create all the necessary utils for object creation and conversion.

To build image we use the make commad

```jsx
make release-image.amd64
```

To run our version of plugin we deploy the new image on a Kubernetes cluster after pushing image to docker repository.

```jsx
docker tag gcr.io/k8s-staging-scheduler-plugins/kube-scheduler:v20230922-v0.26.7-35-g7a5b8557-amd64 felixgeorge/test:trimaran
docker push felixgeorge/test:trimaran
```
