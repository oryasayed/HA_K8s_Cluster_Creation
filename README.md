# HA_K8s_Cluster_Creation

To set up a highly available Kubernetes cluster with 3 master nodes and 2 worker nodes without using a cloud load balancer,
 you can use a virtual machine to act as a load balancer for the API server. Here are the detailed steps for setting up such a cluster:

### Prerequisites
- 3 master nodes aws Ubuntu t2.M
- 2 worker nodes aws Ubuntu t2.M
- 1 load balancer node all aws Ubuntu t2.M
- All nodes should be running a Linux distribution like Ubuntu
so we are creating 
create a security group with these ports
<img width="869" alt="image" src="https://github.com/user-attachments/assets/e269cb5a-7800-4b5c-8e61-28bc42326c0c" />

### Step 1: Prepare the Load Balancer Node
1. **Install HAProxy:**
   ```bash
   sudo apt-get update
   sudo apt-get install -y haproxy
   ```

2. **Configure HAProxy:**
   Edit the HAProxy configuration file (`/etc/haproxy/haproxy.cfg`):
   ```bash
  Vi  /etc/haproxy/haproxy.cfg
  <img width="721" alt="image" src="https://github.com/user-attachments/assets/617c425d-cbf7-4772-9518-97ae5939f789" />

   ``` Add the following configuration: ,,haproxy <img width="409" alt="image" src="https://github.com/user-attachments/assets/40c8aabb-23e4-4cdc-826e-c362f8032454" />


   frontend kubernetes-frontend
       bind *:6443
       option tcplog
       mode tcp
       default_backend kubernetes-backend

   backend kubernetes-backend
       mode tcp
       balance roundrobin
       option tcp-check
       server master1 <MASTER1_IP>:6443 check
       server master2 <MASTER2_IP>:6443 check
   ```
<img width="422" alt="image" src="https://github.com/user-attachments/assets/c8f7cd85-61d8-4307-8c62-214505dc6e44" />

3. **Restart HAProxy:**
   ```bash
   sudo systemctl restart haproxy
   ```
now moving to master and working

### Step 2: Prepare All Nodes (Masters and Workers)
1. **Install Docker, kubeadm, kubelet, and kubectl:**
   ```bash vi name.sh 2 permission sudo chmod +x name.shadd the commands, 
   sudo apt-get update
   sudo apt install docker.io -y
   sudo chmod 666 /var/run/docker.sock
   sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
   sudo mkdir -p -m 755 /etc/apt/keyrings
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   sudo apt update
   sudo apt install -y kubeadm=1.30.0-1.1 kubelet=1.30.0-1.1 kubectl=1.30.0-1.1
   ```

./name.sh
master 1, 2, 3, also on worker nodes.


### Step 3: Initialize the First Master Node to get join token for master and nodes
1. **Initialize the first master node:** we have to change the loadbalancer IP instead of  "LOAD_BALANCER_IP:6443" we should use the IP
   ```bash
   sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_IP:6443" --upload-certs --pod-network-cidr=10.244.0.0/16
   ```
   <img width="1047" alt="image" src="https://github.com/user-attachments/assets/444a82d8-e483-4ae8-8571-d4935f9660f4" />
we add this token to other two masters with sudo
<img width="1291" alt="image" src="https://github.com/user-attachments/assets/36df3e52-895a-4ab2-9034-4414a9b857f4" />

<img width="646" alt="image" src="https://github.com/user-attachments/assets/f85602c1-89d8-4bcd-a1cc-d52e216a1fdb" />

if you see these warning it's okay
<img width="793" alt="image" src="https://github.com/user-attachments/assets/871e8dde-558e-400f-832d-23c8387dc152" />


3. **Set up kubeconfig for the first master node:**
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```



4. **Install Calico network plugin:**
   ```bash
   kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
   ```
you can install master 1 or you can install in all worker node
5. **Install Ingress-NGINX Controller:**
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml
   ```
you can install master 1 or you can install in all worker node

4. Install Ingress-NGINX Controller:
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.49.0/deploy/static/provider/baremetal/deploy.yaml


### Step 4: Join the Second & third Master Node
1. **Get the join command and certificate key from the first master node:**
   ```bash
   kubeadm token create --print-join-command --certificate-key $(kubeadm init phase upload-certs --upload-certs | tail -1)
   ```

2. **Run the join command on the second master node:**
   ```bash
   sudo kubeadm join LOAD_BALANCER_IP:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <certificate-key>
   ```

3. **Set up kubeconfig for the second master node:**
   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```

### Step 5: Join the Worker Nodes
1. **Get the join command from the first master node:**
   ```bash
   kubeadm token create --print-join-command
   ```
<img width="1245" alt="image" src="https://github.com/user-attachments/assets/970e03f3-8867-4d99-93ea-84fdfdc71374" />

2. **Run the join command on each worker node:**
   ```bash
   sudo kubeadm join LOAD_BALANCER_IP:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
   ```
<img width="1245" alt="image" src="https://github.com/user-attachments/assets/ec69fb4d-31d8-4620-a141-4ce6950ccc44" />

<img width="655" alt="image" src="https://github.com/user-attachments/assets/dcbf7c71-8095-433b-a5ed-05dd055435f7" />

### Step 6: Verify the Cluster
1. **Check the status of all nodes:**
   ```bash
   kubectl get nodes
   ```
<img width="787" alt="image" src="https://github.com/user-attachments/assets/ee8bc5be-fb6e-45e8-957f-49948ae6064d" />

2. **Check the status of all pods:**
   ```bash
   kubectl get pods --all-namespaces
   ```

By following these steps, you will have a highly available Kubernetes cluster with two master nodes and three worker nodes, and a load balancer distributing traffic between the master nodes. This setup ensures that if one master node fails, the other will continue to serve the API requests.



# Verification

### Step 1: Install etcdctl
1. **Install etcdctl using apt:**
   ```bash
   sudo apt-get update
   sudo apt-get install -y etcd-client
   ```

### Step 2: Verify Etcd Cluster Health
1. **Check the health of the etcd cluster:**
   the ip is localhost
   ```bash
   ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key endpoint health
   ```

3. **Check the cluster membership:**
   ```bash
   ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key member list
   ```

### Step 3: Verify HAProxy Configuration and Functionality
1. **Configure HAProxy Stats:**
   <img width="299" alt="image" src="https://github.com/user-attachments/assets/ac8612fe-97f5-4c97-8466-4da64167c26e" />
   
   - Add the stats configuration to `/etc/haproxy/haproxy.cfg`:
     ```haproxy
    <img width="338" alt="image" src="https://github.com/user-attachments/assets/ba9c4a5b-6ae6-4278-8adb-602bf99146a3" />
      
   

     listen stats
         bind *:8404
         mode http
         stats enable
         stats uri /
         stats refresh 10s
         stats admin if LOCALHOST
     ```
<img width="371" alt="image" src="https://github.com/user-attachments/assets/25483f35-abd6-4eb0-b8a9-ef8e806712b5" />

<img width="388" alt="image" src="https://github.com/user-attachments/assets/c013f5a1-aed9-4d73-b09a-8ed89845b62b" />

2. **Restart HAProxy:**
   ```bash
   sudo systemctl restart haproxy
   ```
 
3. **Check HAProxy Stats:**
   - Access the stats page at `http://<LOAD_BALANCER_IP>:8404`.
   

<img width="1266" alt="image" src="https://github.com/user-attachments/assets/46cf6782-71b8-4801-95e0-82ca090e7a77" />

### Step 4: Test High Availability
1. **Simulate Master Node Failure:**

do the deployment on master 3

<img width="684" alt="image" src="https://github.com/user-attachments/assets/68c2cc79-d192-4ce3-9e9e-fa4c0cb9259c" />

kubectl get all

<img width="1073" alt="image" src="https://github.com/user-attachments/assets/0c7ec3ae-d3c6-495a-8b0c-1fd1bfb3b7b3" />

the external ip is in pending we can use the pod ip to access for now 
kubectl describe pod (pod ID)

<img width="714" alt="image" src="https://github.com/user-attachments/assets/990fe00b-dcb4-497e-ab43-4932a34af4a9" />


this is the private ip of node
<img width="666" alt="image" src="https://github.com/user-attachments/assets/a5d34755-170f-4efd-a422-b6afe3b59bf2" />

you can check the worker noed ip with port as well 



   - Stop the kubelet service and Docker containers on one of the master nodes to simulate a failure:
     ```bash optional console will be better 
     sudo systemctl stop kubelet
     sudo docker stop $(sudo docker ps -q)

     ```
or go to console stop the master 3 check we can still access the application 
<img width="777" alt="image" src="https://github.com/user-attachments/assets/68b65751-79bc-4840-8968-b1350cdfd34c" />

3. **Verify Cluster Functionality:**
   - Check the status of the cluster from a worker node or the remaining master node:
     ```bash
     kubectl get nodes
     kubectl get pods --all-namespaces
     ```

   - The cluster should still show the remaining nodes as Ready, and the Kubernetes API should be accessible.

4. **HAProxy Routing:**
   - Ensure that HAProxy is routing traffic to the remaining master node. Check the stats page or use curl to test:
     ```bash
     curl -k https://<LOAD_BALANCER_IP>:6443/version
     ```

### Summary
By installing `etcdctl` and using it to check the health and membership of the etcd cluster, you can ensure that your HA setup is working correctly. 
Additionally, configuring HAProxy to route traffic properly and simulating master node failures will help verify the resilience and high availability of your Kubernetes cluster.

