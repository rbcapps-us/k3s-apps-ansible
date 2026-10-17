## Test using the examples directory

If you're not already there, change into the *examples* directory.  This entire document assumes that the current working directory is the *examples* directory.
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
## Install the CloudNativePG Kubernetes Operator

Install the CloudNativePG Kubernetes Operator (only required once per K3s cluster, unless upgrading to a newer version of the operator): 
```sh
ansible-playbook -i "$INVPATH" -u ansible --become-password-file "$BECOMEPWFILE" rbcapps_us.k3s_cloud_native_pg_ansible.cnpg_operator
```

## Deploy a CloudNativePG Database Cluster

A note on cert-manager and TLS certificates.  PostgreSQL connections should be secured with TLS encryption.  TLS certificates and private keys are typically generated in a K3s cluster by using cert-manager.  If you followed the instructions for setting up a ClusterIssuer with a CA private key and certificate, then you should have a `ClusterIssuer` in the default namespace.  Check for its existence:
```sh
kubectl -n default get clusterissuers.cert-manager.io
```
If you have a functioning `ClusterIssuer`, you can use it to issue a TLS certificate to use for encrypting the PostgreSQL connection via TLS.  Also required is a CA certificate, which is the public key for the Certificate Authority (CA) which is used by cert-manager's `ClusterIssuer` to issue TLS certificates.  This should be automatically replicated to all namespaces by *reflector*, so if it exists in the default namespace, it should exist everywhere:
```sh
kubectl -n default get secret
```
Look for a secret with *ca-cert* in its name.  You should be able to look back at the cluster configuration, and if you configured cert-manager, you would have created this secret.

For our demonstration, we will assume that the `ClusterIssuer` is named *cluster-local-ca*, and the CA certificate is named *cluster-local-ca-cert*.  Let's also assume that the domain name (there will only be one) for the PostgreSQL TLS certificate will be, arbitrarily, *cnpg-test.cnpg-cluster.local* (because the domain name really doesn't matter here; it is not verified when negotiating TLS). So we have the following assumptions:
* cnpg_cluster_database_cluster_name (from inventory file): *cnpg-test-cluster*
* `ClusterIssuer`: *cluster-local-ca*
* CA certificate: *cluster-local-ca-cert*
* Domain name for PostgreSQL TLS certificate: *cnpg-test.cnpg-cluster.local*

With the above names in hand, let's alter the `inventories/k3s-test-cluster/group_vars/all.yml` file (don't commit these changes to Git).  Make the following changes:
```yaml
cnpg_cluster_enable_tls: true
cnpg_cluster_root_ca_cert_secret_name: "cluster-local-ca-cert"
cnpg_cluster_tls_secret_name: "cnpg-test-cluster-tls-cert"
cnpg_cluster_cert_manager_tls_cert_auto_generate_params:
  cluster_issuer_name: "cluster-local-ca"
  dns_names:
    - "cnpg-test.cnpg-cluster.local"
```

Deploy a CloudNativePG Database Cluster:
```sh
ansible-playbook -i "$INVPATH" -u ansible --become-password-file "$BECOMEPWFILE" rbcapps_us.k3s_cloud_native_pg_ansible.cnpg_cluster
```

After the cluster starts up, you should be able to access CloudNativePG on port 5432 of any node in the K3s cluster.

In the cluster's inventory, you can set up a Keepalived Cluster Load Balancer, and set up a TCP load balancer for port 5432, to make the CloudNativePG service highly available on a fixed IP address and port.  See the documentation in the `rbcapps_us.k3s_cluster_ansible` Ansible Galaxy collection for details.

## Playbook Defaults

Shared defaults and component installation switches are stored in [roles/init/defaults/main.yml](roles/init/defaults/main.yml) and documented in [roles/init/meta/argument_specs.yml](roles/init/meta/argument_specs.yml).  Role-specific defaults and their documentation are stored in each role's `defaults/main.yml` and `meta/argument_specs.yml` under [roles](roles).  For example, CloudNativePG Cluster settings are in [roles/cnpg_cluster/defaults/main.yml](roles/cnpg_cluster/defaults/main.yml) and [roles/cnpg_cluster/meta/argument_specs.yml](roles/cnpg_cluster/meta/argument_specs.yml).  You can override these values by setting them in the inventory, or, if you are calling roles directly from your own playbooks, by explicitly setting facts or passing vars to the roles which you call.

## Re-using Roles in your own Playbooks

If you need to write your own playbooks, as opposed to just using the playbooks which are provided in this collection, you can simply make a local copy of any of the above-referenced playbooks in your own project, and do your customizations.

## Developing this Project

Install `ansible-lint` on MacOS:
```sh
brew install ansible-lint
```

Validate playbooks and roles for lint errors:
```sh
./validate-lint
```

## Publishing a new version of the Galaxy collection

* Update the version in galaxy.yml
* Run:
    ```sh
    ansible-galaxy collection build
    ```
* Publish the collection, either by CLI:
    ```sh
    ansible-galaxy collection publish rbcapps_us-k3s_cloud_native_pg_ansible-1.0.0.tar.gz --api-key <ansible-galaxy-token-goes-here>
    ```
  or by going to [https://galaxy.ansible.com/ui/my-namespaces/](https://galaxy.ansible.com/ui/my-namespaces/), clicking *View collections* under the correct namespace, clicking *Upload collection*, and selecting the collection tarball.

* Create and push a git tag with the same version as the collection version (replace 1.0.0 with the correct version everywhere in the commands below):
    ```sh
    git tag v1.0.0
    git push origin v1.0.0
    ```
* In GitHub, create a release from the new tag.  Upload the newly built tarball to the new release.
