## **Diktyo Guide: How to install the Network-Aware Scheduling Plugins in K8s**

Kube-Scheduler/Scheduler Plugins, [https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)

Paper: Currently under revision, can be accepted at the time of the assignment.

KEP approved: [https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/kep/260-network-aware-scheduling](https://github.com/kubernetes-sigs/scheduler-plugins/tree/master/kep/260-network-aware-scheduling)

Diktyo-io: [https://github.com/diktyo-io](https://github.com/diktyo-io)

Benchmark: Locust and Teastore application (see diktyo/teastore in diktyo.zip)

Testbed: kubeadm cluster v1.22.4 at KUL. At least 5 nodes required per student group.

Guide folder: _diktyo.zip_

**Installation Guide:**

- 1) Install both CRDs. Go to crd-manifests/

Install AppGroup CRD:

_kubectl apply -f appgroup/crd.yaml_

Install Network Topology CRD:

_kubectl apply -f networktopology/crd.yaml_

Deploy CRD examples to see that everything works:

_kubectl apply -f appgroup/example.yaml_

_kubectl apply -f networktopology/networktopology.yaml (also used in the assignment)_

- 2) Label nodes based on the assignment topology. An example is available at _crd-manifests\networktopology/labels.txt_ based on the networktopology CR.

- 3) Add random delays between 20 and 60 ms on each ens\* network interface of worker nodes (not the master node!) based on Traffic Control. Please check the file tc.txt to see what is the appropriate tc command. You may not use ssh to login to each node but you need to deploy a DaemonSet for this. Also use an imagePullPolicy so that the container image of the Daemonset is pulled exactly one time from the container registry. You can check that the Traffic Control commands have correctly been executed as follows:

_kubectl exec -it \<random pod of daemonset\>_

Then the "_ip link show | grep ens.\*"_ command must show for all _ens_ interfaces the following output:

_2: ens3: \<BROADCAST,MULTICAST,UP,LOWER\_UP\> mtu 1500 qdisc_ _ **netem** _ _state UFAULT group default qlen 1000_

Delete the Daemonset after on all worker workers nodes these output shows

- 4) Deploy controller:

_kubectl apply -f controller/all-in-one.yaml_

Check logs of the deployed container to see that the following output appears. Please ignore any other error messages that appear in the logs.

_kubectl logs scheduler-plugins-controller-566c9c679f-xzvzd -n scheduler-plugins_

_W0221 09:57:33.693841 1 client\_config.go:615] Neither --kubeconfig nor --master was specified. Using the inClusterConfig. This might not work._

_I0221 09:57:33.695620 1 appgcroup.go:106] "Starting App Group controller"_

_I0221 09:57:33.695656 1 elasticquota.go:115] "Starting Elastic Quota control loop"_

_I0221 09:57:33.696677 1 elasticquota.go:117] "Waiting for informer caches to sync"_

_I0221 09:57:33.696015 1 podgroup.go:96] "Starting Pod Group controller"_

_I0221 09:57:34.096810 1 elasticquota.go:122] "Elastic Quota sync finished"_

_I0221 09:57:34.096888 1 podgroup.go:103] "Pod Group sync finished"_

_I0221 09:57:34.096908 1 appgroup.go:113] "App Group sync finished"_

- 5) Deploy network-aware-scheduler:

Install it via the deploy-sched-plugins.yaml file. The sched-cc.yaml must be in the master node in the corresponding folder: /etc/kubernetes/

Image: j_pedro1992/kube-scheduler:kubecon_

- 6) Run the Netperf Component:

Go to pushing-netperf-metrics-to-prometheus/

First, please install the required dependencies shown in _guide.txt_

Deploy netperf pods in the cluster:

_kubectl apply -f k8s-netperf.yaml_

Run the scrip _runTestConfigmap.sh_

Check that the costs are added to the netperf-metrics configmap:

_kubectl get configmaps netperf-metrics_

Since delays are added via TC, running the netperf component only once is enough. Please stop the script and delete all netperf pods.

_kubectl delete -f k8s-netperf.yaml_

Note: The netperf component is open-sourced here: [https://github.com/jpedro1992/pushing-netperf-metrics-to-prometheus/tree/configmap](https://github.com/jpedro1992/pushing-netperf-metrics-to-prometheus/tree/configmap)

- 7) Run the teastore application:

Read first _teastore/readme.md_

Deploy the helm chart with normal Kubernetes Scheduler:

_helm install ts teastore/teastore-helm_

Deploy it with Diktyo:

First edit the _teastore/teastore-helm-diktyo/teastore-appgroup-crd.yaml_ to add dependencies as shown in the _teastore/architecture.pdf_ document with appropriate _maxNetworkCost_ and _minBandwidth_ fields. Please note that in the architecture document the database component and a number of dependencies has been omitted. These smaller dependencies typically require less bandwidth and can deal with a higher network cost (i.e. traffic control delay).

Then apply the crd

_kubectl apply -f teastore/teastore-helm-diktyo/teastore-appgroup-crd.yaml_

Then, inspect the teastore manifest in _teastore/teastore-helm-diktyo/templates/teastore-clusterip._ What needs to be changed so that the diktyo scheduler is used?

Then delete the helm chart with the normal Kubernetes Scheduler if not already done

_helm delete ts_

Then create the helm chart with the diktyo scheduler

_helm install ts teastore/teastore-helm-diktyo_

To check the performance of Teastire, please run the locust generator as explained in the teastore/readme.md file. This runs a performance experiment during 3 minutes.

**What differences/conclusions can you already draw from these? Is the performance of Diktyo as expected? What can you change to improve it?**
