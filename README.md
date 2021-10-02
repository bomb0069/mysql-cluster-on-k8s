# mysql-cluster-on-k8s

## Setup K3D Cluster

1. Create K3d Cluster with [K3d cluster create](https://k3d.io/usage/commands/k3d_cluster_create/)

    ```cmd
    k3d cluster create mysql-cluster --agents 3
    ```

    ```result
    INFO[0000] Prep: Network                                
    INFO[0000] Created network 'k3d-mysql-cluster' (928ee3548a93bc6f0d71a03ebc94d171fc473702d13ef4376b1e9dee7368360a) 
    INFO[0000] Created volume 'k3d-mysql-cluster-images'    
    INFO[0001] Creating node 'k3d-mysql-cluster-server-0'   
    INFO[0007] Pulling image 'docker.io/rancher/k3s:v1.21.3-k3s1' 
    INFO[0026] Creating node 'k3d-mysql-cluster-agent-0'    
    INFO[0026] Creating node 'k3d-mysql-cluster-agent-1'    
    INFO[0026] Creating node 'k3d-mysql-cluster-agent-2'    
    INFO[0027] Creating LoadBalancer 'k3d-mysql-cluster-serverlb' 
    INFO[0031] Pulling image 'docker.io/rancher/k3d-proxy:4.4.8' 
    INFO[0042] Starting cluster 'mysql-cluster'             
    INFO[0042] Starting servers...                          
    INFO[0042] Starting Node 'k3d-mysql-cluster-server-0'   
    INFO[0049] Starting agents...                           
    INFO[0049] Starting Node 'k3d-mysql-cluster-agent-0'    
    INFO[0062] Starting Node 'k3d-mysql-cluster-agent-1'    
    INFO[0070] Starting Node 'k3d-mysql-cluster-agent-2'    
    INFO[0077] Starting helpers...                          
    INFO[0077] Starting Node 'k3d-mysql-cluster-serverlb'   
    INFO[0079] (Optional) Trying to get IP of the docker host and inject it into the cluster as 'host.k3d.internal' for easy access 
    INFO[0081] Successfully added host record to /etc/hosts in 5/5 nodes and to the CoreDNS ConfigMap 
    INFO[0081] Cluster 'mysql-cluster' created successfully! 
    INFO[0081] --kubeconfig-update-default=false --> sets --kubeconfig-switch-context=false 
    INFO[0081] You can now use it like this:                
    kubectl config use-context k3d-mysql-cluster
    kubectl cluster-info
    ```

2. get Cluster Info

    ```cmd
    kubectl cluster-info
    ```

    ```cmd
    Kubernetes control plane is running at https://0.0.0.0:49786
    CoreDNS is running at https://0.0.0.0:49786/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    Metrics-server is running at https://0.0.0.0:49786/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
    ```

3. create MySQL namepsace

    ```cmd
    kubectl create namespace mysql-cluster
    ```

    ```cmd
    namespace/mysql-cluster created
    ```

4. set default namespace to mysql-cluster

    ```cmd
    kubectl config set-context --current --namespace=mysql-cluster
    ```

    ```cmd
    Context "k3d-mysql-cluster" modified.
    ```

## Deploy MySQL Cluster

1. deploy configMap

    ```cmd
    kubectl apply -f mysql-configmap.yaml
    ```

    ```cmd
    configmap/mysql created
    ```

2. deploy MySQL-Service

    ```cmd
    kubectl apply -f mysql-service.yaml
    ```

    ```cmd
    service/mysql created
    service/mysql-read created
    ```

3. deploy MySQL-StatefulSet

    ```cmd
    kubectl apply -f mysql-statefulset.yaml
    ```

    ```cmd
    statefulset.apps/mysql created
    ```

4. watch the startup progress by running

    ```cmd
    kubectl get pods -l app=mysql --watch
    ```

## Sending client traffic

1. Create Table and insert Test Data

    ```cmd
    kubectl run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
    mysql -h mysql-0.mysql <<EOF
    CREATE DATABASE test;
    CREATE TABLE test.messages (message VARCHAR(250));
    INSERT INTO test.messages VALUES ('hello');
    EOF
    ```

2. Use the hostname mysql-read to send test queries to any server that reports being Ready

    ```cmd
    kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
    mysql -h mysql-read -e "SELECT * FROM test.messages"
    ```

    ```cmd
    +---------+
    | message |
    +---------+
    | hello   |
    +---------+
    ```

3. To demonstrate that the mysql-read Service distributes connections across servers, you can run SELECT @@server_id in a loop

    ```cmd
    kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
    bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW()'; done"
    ```

    ```cmd
    +-------------+---------------------+
    | @@server_id | NOW()               |
    +-------------+---------------------+
    |         100 | 2021-10-02 01:35:34 |
    +-------------+---------------------+
    +-------------+---------------------+
    | @@server_id | NOW()               |
    +-------------+---------------------+
    |         100 | 2021-10-02 01:35:35 |
    +-------------+---------------------+
    +-------------+---------------------+
    | @@server_id | NOW()               |
    +-------------+---------------------+
    |         102 | 2021-10-02 01:35:36 |
    +-------------+---------------------+
    +-------------+---------------------+
    | @@server_id | NOW()               |
    +-------------+---------------------+
    |         100 | 2021-10-02 01:35:37 |
    +-------------+---------------------+
    +-------------+---------------------+
    | @@server_id | NOW()               |
    +-------------+---------------------+
    |         100 | 2021-10-02 01:35:38 |
    +-------------+---------------------+
    +-------------+---------------------+
    | @@server_id | NOW()               |
    +-------------+---------------------+
    |         101 | 2021-10-02 01:35:39 |
    +-------------+---------------------+
    +-------------+---------------------+
    | @@server_id | NOW()               |
    +-------------+---------------------+
    |         102 | 2021-10-02 01:35:40 |
    +-------------+---------------------+
    +-------------+---------------------+
    | @@server_id | NOW()               |
    +-------------+---------------------+
    |         102 | 2021-10-02 01:35:41 |
    +-------------+---------------------+
    +-------------+---------------------+
    | @@server_id | NOW()               |
    +-------------+---------------------+
    |         101 | 2021-10-02 01:35:42 |
    +-------------+---------------------+
    +-------------+---------------------+
    | @@server_id | NOW()               |
    +-------------+---------------------+
    |         102 | 2021-10-02 01:35:43 |
    +-------------+---------------------+
    ^C
    ```

    try for query from table

    ```cmd
    kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
    bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW(), message FROM test.messages'; done"
    ```

    ```cmd
    If you don't see a command prompt, try pressing enter.
    +-------------+---------------------+---------+
    | @@server_id | NOW()               | message |
    +-------------+---------------------+---------+
    |         100 | 2021-10-02 01:40:25 | hello   |
    +-------------+---------------------+---------+
    +-------------+---------------------+---------+
    | @@server_id | NOW()               | message |
    +-------------+---------------------+---------+
    |         103 | 2021-10-02 01:40:26 | hello   |
    +-------------+---------------------+---------+
    +-------------+---------------------+---------+
    | @@server_id | NOW()               | message |
    +-------------+---------------------+---------+
    |         104 | 2021-10-02 01:40:27 | hello   |
    +-------------+---------------------+---------+
    +-------------+---------------------+---------+
    | @@server_id | NOW()               | message |
    +-------------+---------------------+---------+
    |         102 | 2021-10-02 01:40:28 | hello   |
    +-------------+---------------------+---------+
    +-------------+---------------------+---------+
    | @@server_id | NOW()               | message |
    +-------------+---------------------+---------+
    |         103 | 2021-10-02 01:40:29 | hello   |
    +-------------+---------------------+---------+
    +-------------+---------------------+---------+
    | @@server_id | NOW()               | message |
    +-------------+---------------------+---------+
    |         100 | 2021-10-02 01:40:30 | hello   |
    +-------------+---------------------+---------+
    ^C
    ```

## Simulating Pod and Node downtime

1. Break the Readiness Probe # Not Work

    ```cmd
    kubectl exec mysql-2 -c mysql -- mv /usr/bin/mysql /usr/bin/mysql.off
    ```

2. Delete Pod

    ```cmd
    kubectl delete pod mysql-2
    ```

    ```cmd
    mysql-0   2/2     Running       0          49m
    mysql-1   2/2     Running       0          48m
    mysql-4   2/2     Running       0          26m
    mysql-3   2/2     Running       0          26m
    mysql-2   2/2     Terminating   0          30m
    mysql-2   0/2     Terminating   0          30m
    mysql-2   0/2     Terminating   0          30m
    mysql-2   0/2     Terminating   0          30m
    mysql-2   0/2     Pending       0          0s
    mysql-2   0/2     Pending       0          0s
    mysql-2   0/2     Init:0/2      0          0s
    mysql-2   0/2     Init:1/2      0          3s
    mysql-2   0/2     PodInitializing   0          4s
    mysql-2   1/2     Running           0          5s
    mysql-2   2/2     Running           0          10s
    ```

## Reference

- [Run a Replicated Stateful Application](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/)