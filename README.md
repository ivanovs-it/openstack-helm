# openstack-helm

How to install Openstack using [OpenStack-Helm](https://docs.openstack.org/openstack-helm/latest/readme.html) project.

### Prerequisites

| Server   | Interface / IP address  | 2nd Interface |
| :------- | :---------------------: | :-----------: |
| seed     |  ens3 / 192.168.1.100   | n/a           |
| helm-01  |  ens3 / 192.168.1.101   | ens4          |
| helm-02  |  ens3 / 192.168.1.102   | ens4          |
| helm-03  |  ens3 / 192.168.1.103   | ens4          |
| helm-04  |  ens3 / 192.168.1.104   | ens4          |

The required host OS is Ubuntu Jammy (22.04). Additionally, a deployer host is needed; this could be a laptop or a workstation.

### Deploy environment
Execute on every node and reboot:
```bash
sudo apt-get update -y && sudo apt-get upgrade -y
sudo reboot
```
To increase the number of loop devices running on the `helm-0*` nodes:
```bash
sudo sed -i 's/GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="max_loop=128"/g' /etc/default/grub
sudo update-grub2
sudo reboot
```
Generate an SSH key and distribute it to all nodes:
```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
```
The following instructions are applied on the deployer node.  
Install necessary packages:
```bash
sudo apt-get install -y acl python3-pip
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```
Install `python3-netaddr` package to avoid 'Failed to import the required Python library (netaddr)' error. This is related to the Ansible [issue](https://github.com/ansible/workshops/issues/115).
```bash
sudo apt-get install -y --no-install-recommends python3-netaddr
```
Install the required Helm objects:
```bash
helm repo add openstack-helm https://tarballs.opendev.org/openstack/openstack-helm
helm repo add openstack-helm-infra https://tarballs.opendev.org/openstack/openstack-helm-infra
helm plugin install https://opendev.org/openstack/openstack-helm-plugin
```
Clone roles git repositories and install Ansible:
```bash
mkdir ~/osh
cd ~/osh
git clone https://opendev.org/openstack/openstack-helm-infra.git
git clone https://opendev.org/zuul/zuul-jobs.git
pip install ansible
export PATH=$PATH:/home/ubuntu/.local/bin
export ANSIBLE_ROLES_PATH=~/osh/openstack-helm-infra/roles:~/osh/zuul-jobs/roles
echo -e "\nexport ANSIBLE_ROLES_PATH=/home/ubuntu/osh/openstack-helm-infra/roles:/home/ubuntu/osh/zuul-jobs/roles" >> ~/.bashrc
```
Assign `ignore_errors` to the following Ansible task, since we will be using the existing device:  
*~/osh/openstack-helm-infra/roles/deploy-env/tasks/loopback_devices.yaml*
```diff
 - name: Create loop device
   shell: |
     mknod {{ loopback_device }} b $(grep loop /proc/devices | cut -c3) {{ loopback_device | regex_search('[0-9]+') }}
+  ignore_errors: true
```

Prepare the inventory file:
```bash
cat > ~/osh/inventory.yaml <<EOF
---
all:
  vars:
    ansible_port: 22
    ansible_user: ubuntu
    ansible_ssh_private_key_file: /home/ubuntu/.ssh/id_ed25519
    ansible_ssh_extra_args: -o StrictHostKeyChecking=no
    kubectl:
      user: ubuntu
      group: ubuntu
    client_ssh_user: ubuntu
    cluster_ssh_user: ubuntu
    docker_users:
      - ubuntu
    metallb_setup: true
    loopback_setup: yes
    loopback_device: /dev/loop100
    loopback_image: /var/lib/openstack-helm/ceph-loop.img
    loopback_image_size: 12G
  children:
    primary:
      hosts:
        seed:
          ansible_host: 192.168.1.100
    k8s_cluster:
      hosts:
        helm-01:
          ansible_host: 192.168.1.101
        helm-02:
          ansible_host: 192.168.1.102
        helm-03:
          ansible_host: 192.168.1.103
        helm-04:
          ansible_host: 192.168.1.104
    k8s_control_plane:
      hosts:
        helm-01:
          ansible_host: 192.168.1.101
    k8s_nodes:
      hosts:
        helm-02:
          ansible_host: 192.168.1.102
        helm-03:
          ansible_host: 192.168.1.103
        helm-04:
          ansible_host: 192.168.1.104
EOF
```
Verify connectivity with Ansible:
```bash
ansible all -m ping -i inventory.yaml
```
Prepare a playbook and run it:
```bash
cat > ~/osh/deploy-env.yaml <<EOF
---
- hosts: all
  become: true
  gather_facts: true
  roles:
    - ensure-python
    - ensure-pip
    - clear-firewall
    - deploy-env
EOF
cd ~/osh
ansible-playbook -i inventory.yaml deploy-env.yaml
```
### Kubernetes prerequisites
The following instructions are applied on the seed node.  
Deploy the ingress controller:
```bash
tee > /tmp/openstack_namespace.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: openstack
EOF
kubectl apply -f /tmp/openstack_namespace.yaml

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
    --version="4.8.3" \
    --namespace=openstack \
    --set controller.kind=Deployment \
    --set controller.admissionWebhooks.enabled="false" \
    --set controller.scope.enabled="true" \
    --set controller.service.enabled="false" \
    --set controller.ingressClassResource.name=nginx \
    --set controller.ingressClassResource.controllerValue="k8s.io/ingress-nginx" \
    --set controller.ingressClassResource.default="false" \
    --set controller.ingressClass=nginx \
    --set controller.labels.app=ingress-api

### Ceph
Install Ceph using scripts provided by the [Rook](https://rook.io/) project.  
Execute on the seed node:
```bash
mkdir ~/osh
cd ~/osh
git clone https://opendev.org/openstack/openstack-helm-infra.git
cd ~/osh/openstack-helm-infra/tools/deployment/ceph
./ceph-rook.sh
```
Assign labels to the Kubernetes nodes:
```bash
kubectl label --overwrite nodes --all openstack-control-plane=enabled
kubectl label --overwrite nodes --all openstack-compute-node=enabled
kubectl label --overwrite nodes --all openvswitch=enabled
```
Generate the client secrets using the script `ceph-adapter-root`:
```bash
cd ~/osh/
helm dependency build openstack-helm-infra/ceph-adapter-rook
helm upgrade --install ceph-adapter-rook openstack-helm-infra/ceph-adapter-rook --namespace=openstack

helm osh wait-for-pods openstack
```

### Deploy OpenStack
Execute on the seed node:
```bash
helm repo add openstack-helm https://tarballs.opendev.org/openstack/openstack-helm

export OPENSTACK_RELEASE=2024.1
export FEATURES="${OPENSTACK_RELEASE} ubuntu_jammy"
export OVERRIDES_DIR=$(pwd)/overrides

cd ~/osh
INFRA_OVERRIDES_URL=https://opendev.org/openstack/openstack-helm-infra/raw/branch/master
for chart in rabbitmq mariadb memcached openvswitch libvirt; do
    helm osh get-values-overrides -d -u ${INFRA_OVERRIDES_URL} -p ${OVERRIDES_DIR} -c ${chart} ${FEATURES}
done

OVERRIDES_URL=https://opendev.org/openstack/openstack-helm/raw/branch/master
for chart in keystone heat glance cinder placement nova neutron horizon; do
    helm osh get-values-overrides -d -u ${OVERRIDES_URL} -p ${OVERRIDES_DIR} -c ${chart} ${FEATURES}
done
```
Execute to deploy RabbitMQ service:
```bash
helm dependency build openstack-helm-infra/rabbitmq
helm upgrade --install rabbitmq openstack-helm-infra/rabbitmq \
    --namespace=openstack \
    --set pod.replicas.server=1 \
    --timeout=600s \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c rabbitmq ${FEATURES})
```
Execute to deploy MariaDB:
```bash
helm dependency build openstack-helm-infra/mariadb
helm upgrade --install mariadb openstack-helm-infra/mariadb \
    --namespace=openstack \
    --set pod.replicas.server=1 \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c mariadb ${FEATURES})

helm osh wait-for-pods openstack
```
Execute to deploy Memcached:
```bash
helm dependency build openstack-helm-infra/memcached
helm upgrade --install memcached openstack-helm-infra/memcached \
    --namespace=openstack \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c memcached ${FEATURES})

helm osh wait-for-pods openstack
```
To deploy the Keystone service run the following:
```bash
helm upgrade --install keystone openstack-helm/keystone \
    --namespace=openstack \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c keystone ${FEATURES})
```
The Glance deployment commands are as follows:
```bash
tee ${OVERRIDES_DIR}/glance/values_overrides/glance_pvc_storage.yaml <<EOF
storage: pvc
volume:
  class_name: general
  size: 10Gi
EOF

helm upgrade --install glance openstack-helm/glance \
    --namespace=openstack \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c glance glance_pvc_storage ${FEATURES})
```
To deploy the OpenStack Cinder use the following:
```bash
helm upgrade --install cinder openstack-helm/cinder \
    --namespace=openstack \
    --timeout=600s \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c cinder ${FEATURES})
```
To deploy the OpenvSwitch service use the following:
```bash
helm dependency build openstack-helm-infra/openvswitch
helm upgrade --install openvswitch openstack-helm-infra/openvswitch \
    --namespace=openstack \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c openvswitch ${FEATURES})

helm osh wait-for-pods openstack
```
To deploy the Libvirt service use the following:
```bash
helm dependency build openstack-helm-infra/libvirt
helm upgrade --install libvirt openstack-helm-infra/libvirt \
    --namespace=openstack \
    --set conf.ceph.enabled=true \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c libvirt ${FEATURES})
```
To deploy the OpenStack Placement service use the following:
```bash
helm upgrade --install placement openstack-helm/placement \
    --namespace=openstack \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c placement ${FEATURES})
```
To deploy the OpenStack Nova service use the following:
```bash
helm upgrade --install nova openstack-helm/nova \
    --namespace=openstack \
    --set bootstrap.wait_for_computes.enabled=true \
    --set conf.ceph.enabled=true \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c nova ${FEATURES})
```
To deploy the OpenStack Neutron service use the following:
```bash
PROVIDER_INTERFACE=ens4
tee ${OVERRIDES_DIR}/neutron/values_overrides/neutron_simple.yaml << EOF
conf:
  neutron:
    DEFAULT:
      l3_ha: False
      max_l3_agents_per_router: 1
  auto_bridge_add:
    br-ex: ${PROVIDER_INTERFACE}
  plugins:
    ml2_conf:
      ml2_type_flat:
        flat_networks: public
    openvswitch_agent:
      ovs:
        bridge_mappings: public:br-ex
EOF

helm upgrade --install neutron openstack-helm/neutron \
    --namespace=openstack \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c neutron neutron_simple ${FEATURES})

helm osh wait-for-pods openstack
```
To deploy the OpenStack Heat service use the following:
```bash
helm upgrade --install heat openstack-helm/heat \
    --namespace=openstack \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c heat ${FEATURES})
```
To deploy the OpenStack Horizon service use the following:
```bash
helm upgrade --install horizon openstack-helm/horizon \
    --namespace=openstack \
    $(helm osh get-values-overrides -p ${OVERRIDES_DIR} -c horizon ${FEATURES})
helm osh wait-for-pods openstack
```
### OpenStack client
Execute on the seed node:
```bash
python3 -m venv ~/openstack-client
source ~/openstack-client/bin/activate
pip install python-openstackclient
```
Prepare the OpenStack client configuration file:
```bash
mkdir -p ~/.config/openstack
tee ~/.config/openstack/clouds.yaml << EOF
clouds:
  openstack_helm:
    region_name: RegionOne
    identity_api_version: 3
    auth:
      username: 'admin'
      password: 'password'
      project_name: 'admin'
      project_domain_name: 'default'
      user_domain_name: 'default'
      auth_url: 'http://keystone.openstack.svc.cluster.local/v3'
EOF
```
To use the OpenStack client try to run this:
```
openstack --os-cloud openstack_helm endpoint list --sort-column 'Service Name' --sort-column Interface
```
