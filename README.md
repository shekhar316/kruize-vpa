# Kruize - VPA

Kruize VPA Recommender for Kubernetes/ Openshift

#### Acknowledgments: 

This project is inspired from [CLEVER](https://github.com/sustainable-computing-io/clever) (Container Level Energy-efficient VPA Recommender) for Kubernetes.

## Prerequisites
- Kubernetes 1.22+ or OpenShift 4.11
- Kubernetes Vertical Pod Autoscaler or OpenShift Vertical Pod Autoscaler Operator
- Kruize Running in Local Monitoring Mode

## Installation

#### Install VPA
  - Follow the instructions [here](https://github.com/kubernetes/autoscaler/blob/master/vertical-pod-autoscaler/README.md) to install the VPA on the Kubernetes Cluster. 

#### Install Kruize in Local Monitoring Mode
  - Follow the instructions [here](https://github.com/kruize/kruize-demos/tree/main/monitoring/local_monitoring) to install Kruize in local monitoring mode and sample tfb workloads.

#### Install Kruize VPA Recommender 
  - In this step, we use the kruize-vpa to deploy it as a customized recommender to run with the default VPA controllers. Clone the Kruize VPA repository. 
    ```commandline
    git clone https://github.com/kruize/kruize-vpa.git
    cd kruize-vpa
    ```
  - Update the Kruize URL in the recommenderConstants file:
    - For OpenShift, retrieve the routes:
    ```commandline
    oc get routes -n openshift-tuning
    ```
    - For Minikube, use the format [minikube_ip]:PORT. 
    - _Note_: This URL is also logged in the kruize-demo script.
    
  - Build the Docker Image. 
    ```commandline
    docker login quay.io/${user_id}
    docker build -t quay.io/${user_id}/kruize-vpa:latest .
    docker push -t quay.io/${user_id}/kruize-vpa:latest .
    ```
  - Update the image in the manifest file. Look for the line#76 containing:
    `image: quay.io/${user_id}/kruize-vpa-patcher:poc`

  - Apply the manifest
    ```commandline
    kubectl apply -f manifests/kruize-vpa.yaml
    ```
    
### Updating Recommendations with Kruize VPA Recommender:
Deploy the example application that selects the Kruize recommender for VPA.
```commandline
kubectl apply -f examples/sysbench.yaml
```
The example YAML also defines a VPA Custom Resource with the following configuration to auto update 

```commandline
apiVersion: "autoscaling.k8s.io/v1"
kind: VerticalPodAutoscaler
metadata:
  name: sysbench-vpa
spec:
  recommenders:
    - name: kruize
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: sysbench
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        controlledResources: ["cpu", "memory"]
```

Kruize VPA will monitor all deployments that have the VPA custom resource defined. It generates and applies recommendations based on the specified resource policies. 

You can also check the logs for the recommendations generated by Kruize VPA to understand the adjustments being made:

```commandline
$ oc logs -n kube-system kruize-vpa-86d6f97f76-j8mcm

VerticalPodAutoscaler CRD is present!
VPA tfb-vpa has chosen kruize recommender.
Selected VPA tfb-vpa in namespace default and deployment tfb-qrh-sample.
Generating recommendations - 
Experiment for VPA exists.
Fetched latest recommendations.
Selected containers to update - 
[{'container_image_name': 'kruize/tfb-qrh:1.13.2.F_et17', 'container_name': 'tfb-server'}]
CPU Recommendations: 0.2879953244444444
Memory Recommendations: 226993766.4
[{'containerName': 'tfb-server', 'lowerBound': {'cpu': '287m', 'memory': '216Mi'}, 'target': {'cpu': '287m', 'memory': '216Mi'}, 'upperBound': {'cpu': '287m', 'memory': '216Mi'}}]
Patching vpa object with available recommendations.
VPA has been patched to apply recommendations.
Sleeping for 60 seconds ...

******************************************************************
```
