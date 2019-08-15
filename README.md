# k8s-deploy
My Notes and helper scripts to deploy Kubernetes and KubeFlow on private data center

## Ubuntu 18.04 setup

### Individual nodes setup:
- If behind corporate `proxy` please ensure `/etc/environment` file has the right proxy information for your network. While updating the file make sure to add individual node names to the `NO_PROXY`/`no_proxy` lists.
- Disable `iptables`:
```
$ sudo iptables -t nat -F
$ sudo iptables -t mangle -F
$ sudo iptables -P FORWARD ACCEPT
$ sudo iptables -P OUTPUT ACCEPT
$ sudo ip6tables -t mangle -F
$ sudo ip6tables -P INPUT ACCEPT
$ sudo ip6tables -P FORWARD ACCEPT
$ sudo ip6tables -P OUTPUT ACCEPT
```
- Disable `Uncomplicated Firewall`:
```
$ sudo systemctl stop ufw
$ sudo systemctl disable ufw
```
- Symlink `/etc/resolv.conf` to `/run/systemd/resolve/`:
```
$ sudo mkdir -p /run/systemd/resolve
$ sudo ln -sf /etc/resolv.conf /run/systemd/resolve/
```
- If you want the symlink to persist accros reboots create the following crontab:
```
@reboot mkdir -p /run/systemd/resolve && ln -sf /etc/resolv.conf /run/systemd/resolve/
```
- A user(`ANSIBLE_USER`) with `passwordless sudo` privileges have been created on all nodes.

### head node setup:
- Clone the [kubespray](https://github.com/kubernetes-sigs/kubespray) repo:
```
$ git clone https://github.com/kubernetes-sigs/kubespray.git
```
- `cd` to `kubespray` directory and checkout the latest tag:
```
$ cd kubespray
$ git fetch --tags
$ latest_tag=$(git describe --tags $(git rev-list --tags --max-count=1))
$ git checkout $latest_tag
```
- Setup a `Python` virtual environment and activate it:
```
$ virutalenv -p python3 kubespray-venv
$ . kubespray-venv/bin/activate
```
- Install all the dependencies:
```
$ pip install -r requirements.txt
```
- In file `inventory/sample/group_vars/all/all.yml` and for `http_proxy`, `https_proxy` and `no_proxy` parameters provide accurate values if needed if you are behind corporate proxy.
- in file `roles/bootstrap-os/defaults/main.yml` ensure to set the value of `override_system_hostname: false` otherwise your hostnames will be changed.
- Now run the following commands to deploy your `k8s` cluster:
```
$ cp -rp inventory/sample inventory/mycluster
$ declare -a IPS=(SPACE_SEPERATED_LIST_OF_ALL_NODES_IP_ADDRESSES>)
$ CONFIG_FILE=inventory/mycluster/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
$ ansible-playbook -i inventory/mycluster/hosts.yml cluster.yml -b -v --private-key=<PATH_TO_PRIVATE_KEY_TO_ACCESS_ALL_NODES> --user=<ANSIBLE_USER>
```

### Deploy metrics server
- Clone `metrics-server` repo: git clone https://github.com/kubernetes-incubator/metrics-server.git and checkout `v0.3.3` tag.
- Edit `deploy/1.8+/metrics-server-deployment.yaml` so that it reads:
```
containers:
- name: metrics-server
    image: k8s.gcr.io/metrics-server-amd64:v0.3.3
    command:
      - /metrics-server
      - --kubelet-insecure-tls
      - --kubelet-preferred-address-types=InternalIP
```
- Finaly deploy the `metrics-server`:
```
$ kubectl create -f deploy/1.8+
```
# Deploy local path provisioner
- In order to let Kubernetes utilize local host's paths in each node, deploy `local-path-provisioner` as described here:
https://github.com/rancher/local-path-provisioner#deployment

### Deploy and login to Kubernetes dashboard:
- Deploy the dashboard:
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
```
- Create a k8s dashboard `admin-user`:
```
$ kubectl create -f dashboard-admin-user.yaml
```
- Retrive `admin-user` token:
```
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```
- Starts a proxy to the Kubernetes API server
```
kubectl proxy
```
- Navigate to http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
- Use the token retrived above and login to dashboard

## Troubleshooting:
- If `playbook` fails to run and `masters` or `workers` fail to run, copy the certificates under `/etc/kubernetes/ssl/` from `master` node to `worker` nodes.
