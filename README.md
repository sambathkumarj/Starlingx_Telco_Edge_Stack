# Starlingx_Telco_Edge_Stack
**TELCO  EDGE STACK ON STARLINGX_KUBERNETES**

**Prerequisite:**
Hardware :
bare-metal,RAN-Gnodeb(cable-free),CPE(cable-free)
Software :
5g Core network(open5gs) - NFV, Starlingx, Kubernetes, OpenStack

**STARLINGX**

Download-iso_image_Starlingx: https://docs.starlingx.io/releasenotes/r8-0-release-notes-6a6ef57f4d99.html

Reference for Setup the Starlingx: https://docs.starlingx.io/r/stx.8.0/deploy_install_guides/release/bare_metal/aio_simplex_install_kubernetes.html

**Follow these steps to set the Starlingx in the bare-metal environment:**

export CONTROLLER0_OAM_CIDR=10.X.22.102/24
export DEFAULT_OAM_GATEWAY=10.X.22.254
sudo ip address add $CONTROLLER0_OAM_CIDR dev eno2
sudo ip link set up dev eno2
sudo ip route add default via $DEFAULT_OAM_GATEWAY dev eno2

sudo vi /etc/resolv.conf
nameserver 1.1.1.1
sysadmin@localhost:~$ sudo ntpdate pool.ntp.org

sysadmin@localhost:~$ cat <<EOF > localhost.yml
system_mode: simplex
dns_servers:
- 1.1.1.1
external_oam_subnet: 10.X.22.0/24
external_oam_gateway_address: 10.X.22.254
external_oam_floating_address: 10.X.22.10X
admin_username: admin
admin_password: <yourwebadminpassword>
ansible_become_pass: <sysadminaccountpassword>
EOF

sysadmin@localhost:~$ ansible-playbook /usr/share/ansible/stx-ansible/playbooks/bootstrap.yml

sysadmin@localhost:~$ source /etc/platform/openrc
[sysadmin@localhost ~(keystone_admin)]$ OAM_IF=eno2
[sysadmin@localhost ~(keystone_admin)]$ system host-if-modify controller-0 $OAM_IF -c platform
[sysadmin@localhost ~(keystone_admin)]$ system interface-network-assign controller-0 $OAM_IF oam
[sysadmin@localhost ~(keystone_admin)]$ system ntp-modify ntpservers=0.pool.ntp.org,1.pool.ntp.org

system host-label-assign controller-0 openstack-control-plane=enabled
system host-label-assign controller-0 openstack-compute-node=enabled
system host-label-assign controller-0 openvswitch=enabled

system host-cpu-modify -f platform -p0 6 controller-0

system host-fs-list controller-0
system host-lvg-list controller-0
system host-fs-modify controller-0 docker=60
system modify --vswitch_type none

export NODE=controller-0
[sysadmin@localhost ~(keystone_admin)]$ system host-fs-add ${NODE} instances=50         

system storage-backend-add ceph --confirmed
system host-disk-list controller-0
system host-stor-add controller-0 osd <disk-uuid>
system host-stor-list controller-0
~(keystone_admin)$ system host-unlock controller-0

**Dashboard** - http://<OAM_IP>:8080
Kubernetes Setup:
https://docs.starlingx.io/r/stx.8.0/deploy_install_guides/release/kubernetes_access.html#kubernetes-access-r7

**Dashboard** - https://<OAM_IP>:32000
After the Kubernetes cluster is set, install open5gs,

**Setup Open5gs using helm charts:**
Open5gs :
Open5GS is a C-language Open Source implementation for 5G Core and EPC, i.e. the core network of LTE/NR network (Release-17) open5gs.org.
      
     Open5gs Helm charts:
Helm charts:- A Helm chart is a set of YAML manifests and templates that describe Kubernetes resources (Deployments, Secrets, CRDs, etc.) and defined configurations needed for the Kubernetes application and is also easy to deploy in a Kubernetes cluster or in a single node with just one command.
Reference link:-
https://www.datree.io/helm-chart/open5gs-cgiraldo#
    • Install Open5gs using Helm charts:
**Add Charts repository to helm :**
helm repo add gradiant-openverso https://gradiant.github.io/openverso-charts/
Install Charts : 
helm install my-open5gs gradiant-openverso/open5gs --version 0.4.1

**Dashboard** - http://<localhost>:3000

**PORT ENABLE IN CONFIG MAP :**
mme,pcrf,hss,smf  - un port = “ {‘port 3868’ – ‘sec port 3868’}”

EDIT SERVICE TYPE IN MME,UPF  “NodePort” :
$kubectl edit svc/mme,upf
In manifests, type = NodePort

**5NR – GNB(CABLE FREE)**
    • Connect Core network to Gnb(cable free) :
    • After installing open5gs on Starlingx , check whether the pod, and service are successfully deployed on the K8S cluster.
    • On the K8S Cluster, the service (cluster IP) of AMF changes by editing the Cluster IP to NodePort for Internal to External traffic.
    • Copy the Service NodePort IP of the AMF and save it for use in Gnb Configuration File(enb.cfg).

**5G NR GNB :**
Check whether the Gnb is connected to the same network as the Core network and reachable to all the microservices running the K8S cluster by adding a default route to the Gnb machine. 
      1. Verify Network Connectivity: Ensure that the Gnb machine is connected to the same network as the Core network. You can do this by checking the network configuration on the Gnb machine and comparing it with the network configuration of the Core network. Ensure that both networks have compatible IP address ranges and subnet masks.
      2. Check Default Gateway: Verify that the Gnb machine has a default gateway configured. The default gateway is the IP address of the router or gateway device that connects the Gnb machine to the wider network, including the Core network. You can check the network configuration on the Gnb machine to confirm the presence of a default gateway.(Example :- sudo ip r add 0.0.0.0/0 via <Core static ip>) 
      3. Validate Reachability: To ensure that the Gnb machine is reachable by all the microservices running on the K8S cluster, you can perform the following steps:
      
      a. From a microservice within the K8S cluster, attempt to ping the IP address of the Gnb machine. If the ping is successful, it indicates that the Gnb machine is reachable from within the K8S cluster.
      
      b. If the Gnb machine has a firewall enabled, ensure that it allows incoming connections from the IP addresses or subnets used by the K8S cluster. Check the firewall configuration on the Gnb machine and make any necessary adjustments to allow incoming connections.
      
       c. Verify Routing: Check the routing tables on the Gnb machine to ensure that it has a route to reach the IP addresses used by the K8S cluster. This includes the IP addresses assigned to the microservices running on the cluster. If the Gnb machine does not have a route to the K8S cluster's IP addresses, you may need to add a specific route or adjust the default route configuration on the Gnb machine.
    • Switch to the root user to edit the configuration files of root/lte.../config/enb.cfg by adding the AMF IP address and Changing the gtpu address of Gnb IP address (static ip where we set) , save, and exiting.
    • Check whether the Gnb and Core are connected by Cable free UI, where Gnap is connected to the service IP of AMF or the Log file generated in gnb0.log file(path nano /tmp/gnb0.log). If Gnb is connected, then check the (UE device to which the sim is attached) Cable free  outdoor CPE has connected to the Gnb in Cable-free UI.
    • Check whether the CPE(UE) has IP address .UPF (Core network – NFV) will assign an IP address for CPE(UE) by tunnel created in UPF. 
      1 . The CPE/UE can have an IP address assigned to it by the UPF (User Plane Function) in the core network, which is a component of the 5G network architecture based on NFV (Network Function Virtualization).
      2 . When a CPE/UE connects to the network, it establishes a communication link with the UPF via a tunnel. This tunnel is created between the CPE/UE and the UPF to facilitate the transmission of data packets.
      3. Once the tunnel is established, the CPE/UE can communicate with the core network and other external networks using the assigned IP address. The UPF acts as a gateway for the CPE/UE, forwarding the data packets between the CPE/UE and the appropriate network destinations.


      










