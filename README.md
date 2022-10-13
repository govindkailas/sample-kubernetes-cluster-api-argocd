
Create and Manage Kubernetes Clusters with Cluster API and ArgoCD
-----------------------------------------------------------------


![](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-16.07.49.png?fit=2096%2C1180&ssl=1)

In this article, you will learn how to create and manage multiple Kubernetes clusters using Kubernetes Cluster API and ArgoCD. We will create a single, local cluster with Kind. On that cluster, we will provision the process of other Kubernetes clusters creation. In order to perform that process automatically, we will use ArgoCD. Thanks to it, we can handle the whole process from a single Git repository. Before we start, let’s do a theoretical brief.

If you are interested in topics related to the Kubernetes multi-clustering you may also read some other articles about it:

1.  [Kubernetes Multicluster with Kind and Cilium](https://piotrminkowski.com/2021/10/25/kubernetes-multicluster-with-kind-and-cilium/)
2.  [Multicluster Traffic Mirroring with Istio and Kind](https://piotrminkowski.com/2021/07/12/multicluster-traffic-mirroring-with-istio-and-kind/)
3.  [Kubernetes Multicluster with Kind and Submariner](https://piotrminkowski.com/2021/07/08/kubernetes-multicluster-with-kind-and-submariner/)

Introduction
------------

Did you hear about a project called [Kubernetes Cluster API](https://cluster-api.sigs.k8s.io/)? It provides declarative APIs and tools to simplify provisioning, upgrading, and managing multiple Kubernetes clusters. In fact, it is a very interesting concept. We are creating a single Kubernetes cluster that manages the lifecycle of other clusters. On this cluster, we are installing Cluster API. And then we are just defining new workload clusters, by creating Cluster API objects. Looks simple? And that’s what is.

Cluster API provides a set of CRDs extending the Kubernetes API. Each of them represents a customization of a Kubernetes cluster installation. I will not get into the details. But, if you are interested you may read more about it [here](https://cluster-api.sigs.k8s.io/user/concepts.html). What’s is important for us, it provides a CLI that handles the lifecycle of a Cluster API management cluster. Also, it allows creating clusters on multiple infrastructures including AWS, GCP, or Azure. However, today we are going to run the whole infrastructure locally on Docker and Kind. It is also possible with Kubernetes Cluster API since it supports Docker.

We will use Cluster API CLI just to initialize the management cluster and generate YAML templates. The whole process will be managed by the ArgoCD installed on the management cluster. Argo CD perfectly fits our scenario, since it supports multi-clusters. The instance installed on a single cluster can manage many other clusters that is able to connect with.

Finally, the last tool used today – [Kind](https://kind.sigs.k8s.io/). Thanks to it, we can run multiple Kubernetes clusters on the same machine using Docker container nodes. Let’s take a look at the architecture of our solution described in this article.

Architecture with Kubernetes Cluster API and ArgoCD
---------------------------------------------------

Here’s the picture with our architecture. The whole infrastructure is running locally on Docker. We install Kubernetes Cluster API and ArgoCD on the management cluster. Then, using both those tools we are creating new clusters with Kind. After that, we are going to apply some Kubernetes objects into the workload clusters (`c1`, `c2`) like `Namespace`, `ResourceQuota` or `RoleBinding`. Of course, the whole process is managed by the Argo CD instance and configuration is stored in the Git repository.

![kubernetes-cluster-api-argocd-arch](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-10.24.47.png?resize=687%2C457&ssl=1)

Source Code
-----------

If you would like to try it by yourself, you may always take a look at my source code. In order to do that you need to clone my GitHub [repository](https://github.com/piomin/sample-kubernetes-cluster-api-argocd.git). After that, you should just follow my instructions. Let’s begin.

Create Management Cluster with Kind and Cluster API
---------------------------------------------------

In the first step, we are going to create a management cluster on Kind. You need to have Docker installed on your machine, `kubectl` and `kind` to do that exercise by yourself. Because we use Docker infrastructure to run Kubernetes workloads clusters, Kind must have an access to the Docker host. Here’s the definition of the Kind cluster. Let’s say the name of the file is `mgmt-cluster-config.yaml`:

    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    nodes:
    - role: control-plane
      extraMounts:
        - hostPath: /var/run/docker.sock
          containerPath: /var/run/docker.sock

Now, let’s just apply the configuration visible above when creating a new cluster with Kind:

    $ kind create cluster --config mgmt-cluster-config.yaml --name mgmt

If everything goes fine you should see a similar output. After that, your Kubernetes context is automatically set to `kind-mgmt`.

![](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-10.40.38.png?resize=696%2C191&ssl=1)

Then, we need to initialize a management cluster. In other words, we have to install Cluster API on our Kind cluster. In order to do that, we first need to install Cluster API CLI on the local machine. On macOS, I can use the `brew install clusterctl` command. Once the `clusterctl` has been successfully installed I can run the following command:

    $ clusterctl init --infrastructure docker

The result should be similar to the following. Maybe, without this timeout 🙂 I’m not sure why it happens, but it doesn’t have any negative impact on the next steps.

![kubernetes-cluster-api-argocd-init-mgmt](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-10.47.55.png?resize=696%2C180&ssl=1)

Once we successfully initialized a management cluster we may verify it. Let’s display e.g. a list of namespaces. There are five new namespaces created by the Cluster API.

    kubectl get ns
    NAME                                STATUS   AGE
    capd-system                         Active   3m37s
    capi-kubeadm-bootstrap-system       Active   3m42s
    capi-kubeadm-control-plane-system   Active   3m40s
    capi-system                         Active   3m44s
    cert-manager                        Active   4m8s
    default                             Active   12m
    kube-node-lease                     Active   12m
    kube-public                         Active   12m
    kube-system                         Active   12m
    local-path-storage                  Active   12m

Also, let’s display a list of all pods. All the pods created inside new namespaces should be in a running state.

    $ kubectl get pod -A

We can also display a list of installed CRDs. Anyway, the Kubernetes Cluster API is running on the management cluster and we can proceed to the further steps.

![](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-10.56.04.png?resize=588%2C361&ssl=1)

Install Argo CD on the management Kubernetes cluster
----------------------------------------------------

I will install Argo CD in the `default` namespace. But you can as well create a namespace `argocd` and install it there (following Argo CD documentation).

    $ kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

Then, let’s just verify the installation:

    $ kubectl get pod
    NAME                                  READY   STATUS    RESTARTS   AGE
    argocd-application-controller-0       1/1     Running   0          63s
    argocd-dex-server-6dcf645b6b-6dlk9    1/1     Running   0          63s
    argocd-redis-5b6967fdfc-vg5k6         1/1     Running   0          63s
    argocd-repo-server-7598bf5999-96mh5   1/1     Running   0          63s
    argocd-server-79f9bc9b44-d6c8q        1/1     Running   0          63s

As you probably know, Argo CD provides a web UI for management. To access it on the local port (8080), I will run the `kubectl port-forward` command:

    $ kubectl port-forward svc/argocd-server 8080:80

Now, the UI is available under `http://localhost:8080`. To login there you need to find a Kubernetes Secret `argocd-initial-admin-secret` and decode the password. The username is `admin`. You can easily decode secrets using for example [Lens](https://k8slens.dev/) – advanced Kubernetes IDE. For now, only just log in there. We will back to Argo CD UI later.

Create Kubernetes cluster with Cluster API and ArgoCD
-----------------------------------------------------

We will use the `clusterctl` CLI to generate YAML manifests with a declaration of a new Kubernetes cluster. To do that we need to run the following command. It will generate and save the manifest into the `c1-clusterapi.yaml` file.

    $ clusterctl generate cluster c1 --flavor development \
      --infrastructure docker \
      --kubernetes-version v1.21.1 \
      --control-plane-machine-count=3 \
      --worker-machine-count=3 \
      > c1-clusterapi.yaml

Our `c1` cluster consists of three master and three worker nodes. Following Cluster API documentation we would have to apply the generated manifests into the management cluster. However, we are going to use ArgoCD to automatically apply the Cluster API manifest stored in the Git repository to Kubernetes. So, let’s create a manifest with Cluster API objects in the Git repository. To simplify the process I will use Helm templates. Because there are two clusters to create we have to Argo CD applications that use the same template with parametrization. Ok, so here’s the Helm template based on the manifest generated in the previous step. You can find it in our sample Git repository under the path `/mgmt/templates/cluster-api-template.yaml`.

    apiVersion: cluster.x-k8s.io/v1beta1
    kind: Cluster
    metadata:
      name: {{ .Values.cluster.name }}
      namespace: default
    spec:
      clusterNetwork:
        pods:
          cidrBlocks:
            - 192.168.0.0/16
        serviceDomain: cluster.local
        services:
          cidrBlocks:
            - 10.128.0.0/12
      controlPlaneRef:
        apiVersion: controlplane.cluster.x-k8s.io/v1beta1
        kind: KubeadmControlPlane
        name: {{ .Values.cluster.name }}-control-plane
        namespace: default
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: DockerCluster
        name: {{ .Values.cluster.name }}
        namespace: default
    ---
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: DockerCluster
    metadata:
      name: {{ .Values.cluster.name }}
      namespace: default
    ---
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: DockerMachineTemplate
    metadata:
      name: {{ .Values.cluster.name }}-control-plane
      namespace: default
    spec:
      template:
        spec:
          extraMounts:
            - containerPath: /var/run/docker.sock
              hostPath: /var/run/docker.sock
    ---
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    metadata:
      name: {{ .Values.cluster.name }}-control-plane
      namespace: default
    spec:
      kubeadmConfigSpec:
        clusterConfiguration:
          apiServer:
            certSANs:
              - localhost
              - 127.0.0.1
          controllerManager:
            extraArgs:
              enable-hostpath-provisioner: "true"
        initConfiguration:
          nodeRegistration:
            criSocket: /var/run/containerd/containerd.sock
            kubeletExtraArgs:
              cgroup-driver: cgroupfs
              eviction-hard: nodefs.available<0%,nodefs.inodesFree<0%,imagefs.available<0%
        joinConfiguration:
          nodeRegistration:
            criSocket: /var/run/containerd/containerd.sock
            kubeletExtraArgs:
              cgroup-driver: cgroupfs
              eviction-hard: nodefs.available<0%,nodefs.inodesFree<0%,imagefs.available<0%
      machineTemplate:
        infrastructureRef:
          apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
          kind: DockerMachineTemplate
          name: {{ .Values.cluster.name }}-control-plane
          namespace: default
      replicas: {{ .Values.cluster.masterNodes }}
      version: {{ .Values.cluster.version }}
    ---
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: DockerMachineTemplate
    metadata:
      name: {{ .Values.cluster.name }}-md-0
      namespace: default
    spec:
      template:
        spec: {}
    ---
    apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
    kind: KubeadmConfigTemplate
    metadata:
      name: {{ .Values.cluster.name }}-md-0
      namespace: default
    spec:
      template:
        spec:
          joinConfiguration:
            nodeRegistration:
              kubeletExtraArgs:
                cgroup-driver: cgroupfs
                eviction-hard: nodefs.available<0%,nodefs.inodesFree<0%,imagefs.available<0%
    ---
    apiVersion: cluster.x-k8s.io/v1beta1
    kind: MachineDeployment
    metadata:
      name: {{ .Values.cluster.name }}-md-0
      namespace: default
    spec:
      clusterName: {{ .Values.cluster.name }}
      replicas: {{ .Values.cluster.workerNodes }}
      selector:
        matchLabels: null
      template:
        spec:
          bootstrap:
            configRef:
              apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
              kind: KubeadmConfigTemplate
              name: {{ .Values.cluster.name }}-md-0
              namespace: default
          clusterName: {{ .Values.cluster.name }}
          infrastructureRef:
            apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
            kind: DockerMachineTemplate
            name: {{ .Values.cluster.name }}-md-0
            namespace: default
          version: {{ .Values.cluster.version }}

We can parameterize four properties related to cluster creation: name of the cluster, number of master and worker nodes, or a version of Kubernetes. Since we use Helm for that, we just need to create the `values.yaml` file containing values of those parameters in YAML format. Here’s the `values.yaml` file for the first cluster. You can find it in the sample Git repository under the path `/mgmt/values-c1.yaml`.

    cluster:
      name: c1
      masterNodes: 3
      workerNodes: 3
      version: v1.21.1

Here’s the same configuration for the second cluster. As you see, there is a single master node and a single worker node. You can find it in the sample Git repository under the path `/mgmt/values-c2.yaml`.

    cluster:
      name: c2
      masterNodes: 1
      workerNodes: 1
      version: v1.21.1

Create Argo CD applications
---------------------------

Since Argo CD supports Helm, we just need to set the right `values.yaml` file in the configuration of the ArgoCD application. Except that, we also need to set the address of our Git configuration repository and the directory with manifests inside the repository. All the configuration for the management cluster is stored inside the `mgmt` directory.

    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: c1-cluster-create
    spec:
      destination:
        name: ''
        namespace: ''
        server: 'https://kubernetes.default.svc'
      source:
        path: mgmt
        repoURL: 'https://github.com/piomin/sample-kubernetes-cluster-api-argocd.git'
        targetRevision: HEAD
        helm:
          valueFiles:
            - values-c1.yaml
      project: default

Here’s a similar declaration for the second cluster:

    <meta charset="utf-8">apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: c2-cluster-create
    spec:
      destination:
        name: ''
        namespace: ''
        server: 'https://kubernetes.default.svc'
      source:
        path: mgmt
        repoURL: 'https://github.com/piomin/sample-kubernetes-cluster-api-argocd.git'
        targetRevision: HEAD
        helm:
          valueFiles:
            - values-c2.yaml
      project: default

Argo CD requires privileges to manage Cluster API objects. Just to simplify, let’s add the `cluster-admin` role to the `argocd-application-controller` `ServiceAccount` used by Argo CD.

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: cluster-admin-argocd-contoller
    subjects:
      - kind: ServiceAccount
        name: argocd-application-controller
        namespace: default
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin

After creating applications in Argo CD you may synchronize them manually (or enable the auto-sync option). It begins the process of creating workload clusters by the Cluster API tool.

![kubernetes-cluster-api-argocd-ui](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-12.38.44.png?resize=682%2C307&ssl=1)

Verify Kubernetes clusters using Cluster API CLI
------------------------------------------------

After performing synchronization with Argo CD we can verify a list of available Kubernetes clusters. To do that just use the following `kind` command:

    $ kind get clusters
    c1
    c2
    mgmt

As you see there are three running clusters! Kubernetes Cluster API installed on the management cluster has created two other clusters based on the configuration applied by Argo CD. To check if everything goes fine we may use the `clusterctl describe` command. After executing this command you would probably have a similar result to the visible below.

![](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-12.42.29.png?resize=696%2C73&ssl=1)

The control plane is not ready. It is described in the Cluster API documentation. We need to install a CNI provider on our workload clusters in the next step. Cluster API documentation suggests installing Calico as the CNI plugin. We will do it, but before we need to switch to the `kind-c1` and `kind-c2` contexts. Of course, they were not created on our local machine by the Cluster API, so first we need to export them to our `Kubeconfig` file. Let’s do that for both workload clusters.

    $ kind export kubeconfig --name c1
    $ kind export kubeconfig --name c2

I’m not sure why, but it exports contexts with `0.0.0.0` as the address of clusters. So in the next step, I also had to edit my `Kubeconfig` file and change this address to `127.0.0.1` as shown below. Now, I can connect both clusters using `kubectl` from my local machine.

![](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-13.12.52.png?resize=343%2C175&ssl=1)

And install Calico CNI on both clusters.

    $ kubectl apply -f https://docs.projectcalico.org/v3.20/manifests/calico.yaml --context kind-c1
    $ kubectl apply -f https://docs.projectcalico.org/v3.20/manifests/calico.yaml --context kind-c2

I could also automate that step in Argo CD. But for now, I just want to finish the installation. In the next section, I’m going to describe how to manage both these clusters using Argo CD running on the management cluster. Now, if you verify the status of both clusters using the `clusterctl describe command, it looks` perfectly fine.

![kubernetes-cluster-api-argocd-cli](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-12.39.26.png?resize=597%2C125&ssl=1)

Managing workload clusters with ArgoCD
--------------------------------------

In the previous section, we have successfully created two Kubernetes clusters using the Cluster API tool and ArgoCD. To clarify, all the Kubernetes objects required to perform that operation were created on the management cluster. Now, we would like to apply a simple configuration visible below to our both workload clusters. Of course, we will also use the same instance of Argo CD for it.

    apiVersion: v1
    kind: Namespace
    metadata:
      name: demo
    ---
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: demo-quota
      namespace: demo
    spec:
      hard:
        pods: '10'
        requests.cpu: '1'
        requests.memory: 1Gi
        limits.cpu: '2'
        limits.memory: 4Gi
    ---
    apiVersion: v1
    kind: LimitRange
    metadata:
      name: demo-limitrange
      namespace: demo
    spec:
      limits:
        - default:
            memory: 512Mi
            cpu: 500m
          defaultRequest:
            cpu: 100m
            memory: 128Mi
          type: Container

Unfortunately, there is no built-in integration between ArgoCD and the Kubernetes Cluster API tool. Although Cluster API creates a secret containing the `Kubeconfig` file per each created cluster, Argo CD is not able to recognize it to automatically add such a cluster to the managed clusters. If you are interested in more details, there is an interesting discussion about it [here](https://github.com/argoproj/argo-cd/issues/4651). Anyway, the goal, for now, is to add both our workload clusters to the list of clusters managed by the global instance of Argo CD running on the management cluster. To do that, we first need to log in to Argo CD. We need to use the same credentials and URL used to interact with web UI.

    $ argocd login localhost:8080

Now, we just need to run the following commands, assuming we have already exported both Kubernetes contexts to our local `Kubeconfig` file:

    $ argocd cluster add kind-c1
    $ argocd cluster add kind-c2

If you run Docker on macOS or Windows it is not such a simple thing to do. You need to use an internal Docker address of your cluster. Cluster API creates secrets containing the `Kubeconfig` file for all created clusters. We can use it to verify the internal address of our Kubernetes API. Here’s a list of secrets for our workload clusters:

    $ kubectl get secrets | grep kubeconfig
    c1-kubeconfig                               cluster.x-k8s.io/secret               1      85m
    c2-kubeconfig                               cluster.x-k8s.io/secret               1      57m

We can obtain the internal address after decoding a particular secret. For example, the internal address of my `c1` cluster is `172.20.0.3`.

![](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-15.30.03.png?resize=408%2C279&ssl=1)

Under the hood, Argo CD creates a secret related to each of managed clusters. It is recognized basing on the label name and value: `argocd.argoproj.io/secret-type: cluster`.

    apiVersion: v1
    kind: Secret
    metadata:
      name: c1-cluster-secret
      labels:
        argocd.argoproj.io/secret-type: cluster
    type: Opaque
    data:
      name: c1
      server: https://172.20.0.3:6443
      config: |
        {
          "tlsClientConfig": {
            "insecure": false,
            "caData": "<base64 encoded certificate>",
            "certData": "<base64 encoded certificate>",
            "keyData": "<base64 encoded key>"
          }
        }

If you added all your clusters successfully, you should see the following list in the _Clusters_ section on your Argo CD instance.

![](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-13.56.14.png?resize=663%2C158&ssl=1)

Create Argo CD application for workload clusters
------------------------------------------------

Finally, let’s create Argo CD applications for managing configuration on both workload clusters.

    <meta charset="utf-8">apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: c1-cluster-config
    spec:
      project: default
      source:
        repoURL: 'https://github.com/piomin/sample-kubernetes-cluster-api-argocd.git'
        path: workload
        targetRevision: HEAD
      destination:
        server: 'https://172.20.0.3:6443'

And similarly to apply the configuration on the second cluster:

    <meta charset="utf-8">apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: c2-cluster-config
    spec:
      project: default
      source:
        repoURL: 'https://github.com/piomin/sample-kubernetes-cluster-api-argocd.git'
        path: workload
        targetRevision: HEAD
      destination:
        server: 'https://172.20.0.10:6443'

Once you created both applications on Argo CD you synchronize them.

![](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-13.57.51.png?resize=696%2C424&ssl=1)

And finally, let’s verify that configuration has been successfully applied to the target clusters.

![](https://i0.wp.com/piotrminkowski.com/wp-content/uploads/2021/12/Screenshot-2021-12-03-at-14.15.32.png?resize=465%2C224&ssl=1)
