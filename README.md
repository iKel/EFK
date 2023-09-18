<h1 align="center">Implement Logging Solution</h1>

***

**Main purpose:** Implementing centralized logging solution for EKS cluster by configuring EFK (Elasticsearch, FluentD, Kibana)

***

**How did I approach it?**

- reviewing Final Project Guideline
- attending standups/meetings
- collaborating with Project manager and team members
- reading specific documentation and following best practices to configure EFK

***

**Why EFK logging solution?** 

EFK is the next evolution of the ELK Stack (Elasticsearch, Logstash and Kibana). In recent years, Fluentd has gained popularity as an alternative to Logstash in this stack.

For Kubernetes environments, Fluentd seems ideal due to its built-in Docker logging driver and parser – which doesn’t require an extra agent to be present on the container to push logs to Fluentd. In comparison with Logstash, this makes the architecture less complex and also makes it less risky for logging mistakes.

***

**EFK in Project Architecture**

- stack for Centralized Log Aggregation
- collect, store, and analyze log data from all the pods and nodes in the cluster

![My Image](Final_Project_Diagram.jpg)

***

**Deployment Architecture:**

![My Remote Image](https://miro.medium.com/v2/resize:fit:1220/format:webp/0*r_NYvFLNCcI1TpJm.png)

-    **Elasticsearch**
        - NoSQL database
        - storage for large volumes of logs and retrieve logs from Fluentd

-    **Fluentd**
        - integrate with different data sources and can collect logs from various applications running in containers

-    **Kibana**   
        - powerful visualization tool to analyze and monitor logs from different sources in real-time

***

**Prerequisites**

1. Install [Homebrew](https://brew.sh/)
2. Install [Kubernetes](https://kubernetes.io/docs/tasks/tools/)
3. Install [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and use your 312shool credentials
4. Install [Helm](https://helm.sh/docs/intro/install/)
5. Install KubeDB:

* getting cluster ID

        kubectl get ns kube-system -o jsonpath='{.metadata.uid}'

* getting Appscode License, going to [Appscode Licence Server](https://license-issuer.appscode.com/?_gl=1*1iu1r2v*_ga*NDY2NDM0NDI5LjE2ODk2NDgzODg.*_ga_R5J3WVDEFB*MTY5MTUzNTA4OS45LjAuMTY5MTUzNTA4OS42MC4wLjA.)

* installing KubeDB:

        helm repo add appscode https://charts.appscode.com/stable/
        helm repo update
        helm search repo appscode/kubedb
        helm install kubedb appscode/kubedb \
             --version v2023.04.10 \
             --namespace kubedb --create-namespace \
              --set kubedb-provisioner.enabled=true \
             --set kubedb-ops-manager.enabled=true \
             --set kubedb-autoscaler.enabled=true \
             --set kubedb-dashboard.enabled=true \
             --set kubedb-schema-manager.enabled=true \
             --set-file global.license=/path/to/the/license.txt

***

**Workflow**

1. Install KubeDB 
2. Created namespace 'logging' for Elasticsearch and Kibana
3. Deployed Elasticsearch Topology Cluster 
4. Deployed Kibana
5. Deployed Fluentd
6. Ready to discover Kubernetes logs in Kibana

***

**Issues and Troubleshooting**

1. Warning  FailedScheduling  18s   default-scheduler  0/4 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/4 nodes are available: 4 No preemption victims found for incoming pod..

    - When? 
While creating Elasticsearch statefulset (elasticsearch-data, elasticsearch-ingest, elasticsearch-master) and pods respectively out of them

    - How to troubleshoot? 

        * kubectl describe sts sts-name -n namespace-name
    
        * kubectl describe pod-name -n namespace-name

    - Explanation of the problem:
A pod creates PersistentVolumeClaim (PVC) and bounds to PersistantVolume (PV) which is automatically created if there is a Provisioner in a cluster. Each StorageClass (SC) has a Provisioner that determines what volume plugin is used for provisioning PVs. In this case, PVC couldn't bound to PV, because PV was't provisioned by SC.

    - How to fix?
SC (ebs-sc1) was created with AWS-EBS provisioner and gp2 volume type. PVs were created and provisioned by SC (ebs-sc1). And then PVCs were manually bounded to PVs. Pods status from 'Pending' changed to 'Running'. 

2. Warning  FailedScheduling  3m40s (x6 over 39m)  default-scheduler  0/4 nodes are available: 4 Insufficient memory. preemption: 0/4 nodes are available: 4 No preemption victims found for incoming pod..

    - When?
While scheduling the Pods 

    - How to troubleshoot? 

        * kubectl describe pod-name -n namespace-name

    - Explanation of the problem:
After Pod is created, it is supposed to be assigned to Node. The problem occured because a Pod request more resources than Node can provide. 

    - How to fix?
3 more Nodes were created, so now totally 6 Nodes are in EKS-cluster. Also the amount of storage each Pod requested was reduced to 1Gi. 

3. Pods automatically terminate -> Lost access to Kibana

    - When?
While scheduling the Pods 

    - How to troubleshoot? 

        * kubectl describe pod-name -n namespace-name

        * kubectl describe node-name

    - Explanation of the problem:
If a Node is in NotReady state, my Pod in a respective Node terminates automatically, and therefore I'm loosing access to Kibana. It seemed that the problem in kubelet. The node awas in the MemoryPressure, DiskPressure, or PIDPressure 'Unknown' status, then we must manage the resources to allow additional pods to be scheduled on the node.

    - How to fix?
Adjusting Nodes and its resources, but it is a problem out of my individual scope, as it may affect other team members' work and EKS-cluster, so I tried wait and restart.
Restart kubelet component in the Node, or deleting and recreating Pod again.
I just waited for 1-2 hours and made a break. The problem was solved by itself. Seems like most of the team members were working on tickets that time  and EKS-cluster was simply overloaded. 

***

**How to access Kibana?**

1. Log in your AWS account from CLI 

        aws sso login --profile your-profile-nickname

2. Connect to EKS cluster 

        aws eks --region us-east-1 update-kubeconfig --name EKS-cluster

3. In your terminal use the command 

        kubectl port-forward -n logging svc/kibana 5601

4. In your web browser visit (http://localhost:5601) and enter the credentials stored in AWS SecretsManager under the name 'kibana-secret'
5. From Home side panel go to "Discover" section and you will find the index pattern to visualize incoming logs from kubernetes nodes and containers


***

**References:**
    1. [Kubernetes Documentation](https://kubernetes.io/docs/home/)
    2. [Git Documentation](https://git-scm.com/doc)

