MariaDB High Availibility

This architecture attempts to implement the following diagram. 
![MariaDB High Availibility Architecture](/screenshot/architecture.png)

Components:
- Maxscale Cluster (2 Maxscale nodes, Active/Passive config for failover)
- Keepalived (Enables VirtualIP that both Maxscale points to for failover)
- Galera Cluster (Multi-Master Synchronous Replication of 3 Nodes in Active/Active configuration)

Note: chart deploys in the namespace `cluster1`

I. Setup Instructions

Namespace
This project will deploy everything under the `cluster1` namespace. This allows us to eventually add more clusters to our deployment ie `cluster2`.
```bash
# in project root
kubectl apply -f namespace.yaml
```
2. Galera Cluster
We will deploy galera using pvc claim name below. The stateful set will generate the corresponding PVCs for each of the 3 galera nodes.
```
Deploy the galera cluster named `galera-1`.
```bash
helm install galera-1 -n cluster1 \
  --set rootUser.password=r00t \
  --set db.user=user \
  --set db.name=test \
    bitnami/mariadb-galera

# check claims were created
kubectl get pv
kubectl get pvc -n cluster1
```
3. Maxscale
This uses a custom maxscale docker file which installs keepalived within the docker container (Note: I am unsure if this is correct as I cannot get Virtual IPs to test keepalived).
Build the container:
```bash
cd /charts/maxscale-helm/docker
# build the image
docker build -t youruser/maxscale-keepalived:latest .
# upload to mirantis
docker tag youruser/maxscale-keepalived:latest
# login and upload to dockerhub
docker login s
docker push youruser/maxscale-keepalived:latest
# Update values.yaml image to match the above 
```
2. Obtain Galera Cluster IPs you wish to loadbalance
    ```bash
    kubectl get all -n your-namespace
    ### Example Output:
    NAME                                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
    service/galera-1-mariadb-galera            ClusterIP   10.96.129.73   <none>        3306/TCP                     5d15h
    service/galera-2-mariadb-galera            ClusterIP   10.96.23.75    <none>        3306/TCP                     5d15h
    ```
3. In the [values.yaml](values.yaml) file:
    a. Change Server 1 and Server2 in `maxscaleConfiguration` to match Cluster IP from Step 2  
    b. Change virtual_ipaddress in `maxscaleConfiguration` to desited Virtual IP
4. Attach to each cluster and create the following maxscale user (default: maxscaleuser) and passwords (Default: maxscaleuser12):
    For each cluster create and grant user:
    ```bash
    # on cluster 1
    create user 'maxscaleuser'@'10.96.129.73' identified by 'maxscaleuser12';
    grant all on *.* to 'maxscaleuser'@'10.96.129.73';
    # on cluster 2
    create user 'maxscaleuser'@'10.96.23.75' identified by 'maxscaleuser12';
    grant all on *.* to 'maxscaleuser'@'10.96.23.75';
    ```
4. Install using Helm 3
    ```bash
    helm3 install maxscale-cluster -n cluster1 .
    ```
5. Attach to container using `maxctrl` command:
    ```bash
    kubectl get all -n cluster1
    kubectl exec -it pod/maxscale-cluster-0 -n cluster1 -- maxctrl
    # show servers
    maxctrl list servers
    ```

II. Cleaning Up:
```
# Remove Galera Nodes gracefully:
kubectl scale sts -n my-namespace galera-1 --replicas=0
helm3 delete galera -n cluster1

# Remove the 3 Persistent Volumes Claim (PVC) for each Node
kubectl get pvc -n cluster1
kubectl delete pvc -n cluster1 PVC-NAME
# Check and delete any PVs that still exist 
kubectl get pv
kubectl delete pv PV-NAME

# Remove Maxscale
helm3 delete 
```

III. References:
- [MariaDB High Availibility Failover](https://www.nitorinfotech.com/blog/your-all-in-one-guide-to-ensuring-mariadb-high-availability-failover/) MariaDB HA end to end configuration setup article
- [How to Configure Maxscale Proxy](https://severalnines.com/blog/how-install-and-configure-maxscale-mariadb/)
- [Virtual IP ELI5](https://serverfault.com/questions/1104895/how-to-configure-keepalived-virtual-ip) In a nutshell how Virtual IP works.
- [Redhat Keepalived Basic Config](https://www.redhat.com/sysadmin/keepalived-basics) Keepalived in a standard configuration
- [Redhat Keepalived Advanced HA Config](https://www.redhat.com/sysadmin/advanced-keepalived) Keepalived in a Primary/Secondary or Active/Passive config for failover
- [Keepalived VRRP Explained](https://www.pentestpartners.com/security-blog/how-to-use-keepalived-for-high-availability-and-load-balancing/) _ Keepalived uses VRRP protocol. This article explains more about it under the hood
- [KubeVip](https://inductor.medium.com/say-good-bye-to-haproxy-and-keepalived-with-kube-vip-on-your-ha-k8s-control-plane-bb7237eca9fc) Article details how you can replace the traditional Keepalived+Proxy configuration with KubeVip Control Planes