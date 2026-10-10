# Ansible Collection - roncemer.k3s_apps_ansible

An Ansible collection to easily install various applications on a K3s cluster which has been provisioned by the `roncemer/k3s_cluster_ansible` Ansible Galaxy collection

## Test using the examples directory

Change into the *examples* directory.  You will be working from there.
```sh
cd examples
```

Install Ansible:
```sh
brew install ansible
```

Install the required Galaxy module(s):
```sh
ansible-galaxy install -r requirements.yml
```

Create a file named *inventories/k3s-test-cluster/ansible-become-password.txt*, and put the password for the *ansible* user which you created when you set up the VMs.

NOTE: If you want to use the local copy of this collection, you can run the following command from the *examples* directory:
```sh
./install-local
```
This is also useful when making changes to the collection's playbooks or roles.  In that case, you can run the *install-local* before each playbook run, and that will ensure that you're always using the latest copy of the local playbooks and roles from the collection.

## Set the Inventory Environment Variables; Install the CloudNativePG Kubernetes Operator; Deploy a CloudNativePG Database Cluster

Set environment variables for the example inventory:
```sh
. ./set-ansible-inventory-k3s-test-cluster.sh
```

Install the CloudNativePG Kubernetes Operator (only required once per K3s cluster, unless upgrading to a newer version of the operator): 
```sh
ansible-playbook -i "$INVPATH" -u ansible --become-password-file "$BECOMEPWFILE" roncemer.k3s_apps_ansible.cloud_native_pg_operator
```

Deploy a CloudNativePG Database Cluster:
```sh
ansible-playbook -i "$INVPATH" -u ansible --become-password-file "$BECOMEPWFILE" roncemer.k3s_apps_ansible.cloud_native_pg_cluster
```

After the cluster starts up, you should be able to access CloudNativePG on port 5432 of any node in the K3s cluster.

In the cluster's inventory, you can set up a Keepalived Cluster Load Balancer, and set up a TCP load balancer for port 5432, to make the CloudNativePG service highly available on a fixed IP address and port.  See the documentation in the `roncemer/k3s_cluster_ansible` Ansible Galaxy collection for details.
