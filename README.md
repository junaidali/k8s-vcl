# Kubernetes Apache VCL

This project allows for deploying [Apache VCL](https://vcl.apache.org/) to a [Kubernetes](https://kubernetes.io) cluster

# Pre-Requisites

You will need the following setup before you can use VCL in Kubernetes

1. Working Kubernetes cluster

```bash
kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    10d       v1.8.4+coreos.0
w1        Ready     node      10d       v1.8.4+coreos.0
w2        Ready     node      10d       v1.8.4+coreos.0
```

2. [CNI Genie Network Plugin](https://github.com/Huawei-PaaS/CNI-Genie)

```bash
kubectl get pod -n kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
calico-node-9qfl6                       1/1       Running   0          10d
calico-node-dqjb8                       1/1       Running   0          10d
calico-node-lvgk4                       1/1       Running   0          10d
genie-5bczq                             1/1       Running   0          10d <--- Genie Pod
genie-hhbql                             1/1       Running   0          10d <--- Genie Pod
kube-apiserver-master                   1/1       Running   0          10d
kube-controller-manager-master          1/1       Running   0          10d
kube-dns-cf9d8c47-gbn29                 3/3       Running   0          10d
kube-dns-cf9d8c47-r94ct                 3/3       Running   0          10d
kube-proxy-master                       1/1       Running   0          10d
kube-proxy-w1                           1/1       Running   0          10d
kube-proxy-w2                           1/1       Running   0          10d
kube-scheduler-master                   1/1       Running   0          10d
kubedns-autoscaler-86c47697df-wqm5v     1/1       Running   0          10d
kubernetes-dashboard-85d88b455f-q92t4   1/1       Running   0          10d
nginx-proxy-w1                          1/1       Running   0          10d
nginx-proxy-w2                          1/1       Running   0          10d
```

3. [macvlan](https://github.com/containernetworking/plugins/tree/master/plugins/main/macvlan) plugin configured on the worker nodes where the VCL management daemon pod will be running. This is needed for providing access to the private VM Network on the management nodes. A sample macvlan configuration file (/etc/cni/net.d/20-macvlan.conf) is shown below. This file should be available on each minion where the management node pod will be running. Replace the "master" with the network interface that is connected to the private VM Network.

```bash
{
	"name": "vmprivate",
	"type": "macvlan",
	"master": "ens192",
	"ipam": {
		"type": "dhcp"
	}
}
```

4. The network interface where the management daemon pod will try to get DHCP address needs to have promiscuous mode enabled. This setting should be enabled on each minion where the management node pod will be running.

```bash
sudo ip link set ens192 promisc on
```

5. The VMWare switch where this network interface will be connected also needs to have promiscuous mode enabled.

![VMWare Promiscuous Mode](images/vmware-promiscuous-mode.png)

6. CNI DHCP Server Daemon configured. This can be setup on systemd enabled system using below steps:

  a. Create new file /etc/systemd/system/cni-dhcp.service

  ```bash
  [Unit]
  Description=CNI DHCP Server
  After=network.target

  [Service]
  Type=simple
  ExecStartPre=/bin/rm -f /run/cni/dhcp.sock
  ExecStart=/opt/cni/bin/dhcp daemon
  ExecStopPost=/bin/rm -f /run/cni/dhcp.sock
  Restart=always
  RestartSec=10s

  [Install]
  WantedBy=multi-user.target
  ```

  b. Enable the cni-dhcp Service

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl enable cni-dhcp.service
  ```

  c. Start the cni-dhcp Service

  ```bash
  sudo systemctl start cni-dhcp.service
  ```

# Deploying VCL to kubernetes
This section describes the deployment of the VCL application

## Configuration
You will need to perform storage and secrets configuration before the VCL service can be launched in Kubernetes.

### Update storage Configuration
You will need to create persistent storage for the MySQL data. Refer to [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) for more information.

For testing purposes, you can use local storage using the local-volumes.yaml file that stores data in the /tmp/data/pv-1 volume.

### Update Secrets
VCL website and management node information is stored within secrets. Refer to [Kubernetes secrets](https://kubernetes.io/docs/concepts/configuration/secret/) for more information.

Update the secrets.yaml file with the following information. all the below secrets can be generated using, where `<text>` is the text that needs to be encrypted.

```bash
echo "<text>" | base64
```

mysql_root_pass: Root password for MySQL
mysql_user_pass: vcl user MySQL password
web_crypt_key: cryptography key
web_pem_key: cryptography key
xmlrpc_pass: XML RPC password
xmlrpc_username: XML RPC username

## Deployment
### Storage and configuration details
Create the local storage persistent volumes and secrets used by the pods.

```bash
// If using local volumes, else use your storage file,
kubectl create -f local-volumes.yml

kubectl create -f secrets.yaml
```

### MySQL database
Create the MySQL pod

```bash
kubectl create -f mysql-deployment.yaml
```

Verify if the MySQL resources have been created successfully

```bash
kubectl get pod
NAME                         READY     STATUS    RESTARTS   AGE
vcl-mysql-764776f4b8-pd59x   1/1       Running   0          12d
```

Verify if the MySQL database services have been created and exposed

```bash
kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
kubernetes   ClusterIP   10.233.0.1      <none>        443/TCP                      13d
vcl-mysql    NodePort    10.233.38.120   <none>        3306:31359/TCP               4d
```

Load the VCL database using the database file provided as part of the [Apache VCL Download](https://vcl.apache.org/downloads/download.cgi).

```bash
kubectl exec -it <vcl-mysql-pod-name> mysql -u root -p <MySQL Root user password from secrets.yaml> vcl < ~/apache-VCL-2.5/mysql/vcl.sql

// e.g. if the pod name is vcl-mysql-764776f4b8-pd59x and the mysql root password is s3cr3t,
kubectl exec -it vcl-mysql-764776f4b8-pd59x mysql -u root -p s3cr3t vcl < ~/apache-VCL-2.5/mysql/vcl.sql
```

Verify if the database is loaded correctly

```bash
kubectl exec -it <vcl-mysql-pod-name> sh
# mysql -u root -p
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| vcl                |
+--------------------+
mysql> use vcl;
mysql> show tables;
+---------------------------+
| Tables_in_vcl             |
+---------------------------+
| IMtype                    |
| OS                        |
| OSinstalltype             |
| OStype                    |
| addomain                  |
| adminlevel                |
| affiliation               |
| blockComputers            |
| blockRequest              |
| blockTimes                |
| blockWebDate              |
| blockWebTime              |
| changelog                 |
| clickThroughs             |
| computer                  |
| computerloadflow          |
| computerloadlog           |
| computerloadstate         |
| connectlog                |
| connectmethod             |
| connectmethodmap          |
| connectmethodport         |
| continuations             |
| cryptkey                  |
| cryptsecret               |
| documentation             |
| image                     |
| imageaddomain             |
| imagemeta                 |
| imagerevision             |
| imagerevisioninfo         |
| imagetype                 |
| localauth                 |
| log                       |
| loginlog                  |
| managementnode            |
| module                    |
| nathost                   |
| nathostcomputermap        |
| natlog                    |
| natport                   |
| oneclick                  |
| openstackcomputermap      |
| openstackimagerevision    |
| platform                  |
| privnode                  |
| provisioning              |
| provisioningOSinstalltype |
| querylog                  |
| request                   |
| reservation               |
| reservationaccounts       |
| resource                  |
| resourcegroup             |
| resourcegroupmembers      |
| resourcemap               |
| resourcepriv              |
| resourcetype              |
| schedule                  |
| scheduletimes             |
| semaphore                 |
| serverprofile             |
| serverrequest             |
| shibauth                  |
| sitemaintenance           |
| state                     |
| statgraphcache            |
| subimages                 |
| sublog                    |
| user                      |
| usergroup                 |
| usergroupmembers          |
| usergrouppriv             |
| usergroupprivtype         |
| userpriv                  |
| userprivtype              |
| variable                  |
| vcldsemaphore             |
| vmhost                    |
| vmprofile                 |
| vmtype                    |
| winKMS                    |
| winProductKey             |
| xmlrpcLog                 |
+---------------------------+
84 rows in set (0.00 sec)
mysql> \q
Bye
exit
```

### Website & Management Daemon
Once the MySQL database is created, we can create the website and management node daemon pods as below

```bash
kubectl create -f frontend-deployment.yaml
kubectl create -f managementnode-deployment.yaml
```

Verify if the pods have been created

```bash
kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
vcl-mgmt-6f5c585777-s9h49    1/1       Running   0          4d
vcl-mysql-764776f4b8-pd59x   1/1       Running   0          12d
vcl-web-7f9df965bb-b5rfn     1/1       Running   0          10d
```

Verify if the VCL website services have been created and exposed

```bash
kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
kubernetes   ClusterIP   10.233.0.1      <none>        443/TCP                      13d
vcl-mysql    NodePort    10.233.38.120   <none>        3306:31359/TCP               4d
vcl-web      NodePort    10.233.29.129   <none>        80:32682/TCP,443:30758/TCP   10d
```


## VCL Configuation
Once the resources have been deployment, perform the following configuration. Refer to [Administering VCL](https://cwiki.apache.org/confluence/display/VCL/Administering+VCL) for more information.

### Add management nodes to allManagementNode group


### Configure VMHost Profiles


### Add VMHosts


### Add VM's


### Create Base Images
