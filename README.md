# MicroShift Ansible Pull

This repository serves as a demonstration of how one might use supported Red Hat Ansible Automation Platform (AAP) components (namely, a supported Execution Environment and supported collections) to manage a Red Hat Device Edge (RHDE) platform fielded with the Red Hat Build of MicroShift, which is an optional component of the installation. It is designed to work in a fully pull-based model, similar to execution with `ansible-pull`, without requiring a full AAP deployment with a Controller and Execution Mesh.

## Quickstart

1. Deploy a MicroShift 4.12 cluster on RHEL 8.7, using either the RPM-based installation or using a custom rpm-ostree image as defined in [the documentation](https://access.redhat.com/documentation/en-us/red_hat_build_of_microshift/4.12/html/installing/index)
2. Ensure that you have at least 11 GiB of free space in the `rhel` LVM Volume Group as described [here](https://access.redhat.com/documentation/en-us/red_hat_build_of_microshift/4.12/html/storage/microshift-storage-plugin-overview#doc-wrapper)
3. Optionally, configure DNS, port-forwarding, etc. for your own network to have your MicroShift instance exposed and available at the level you desire.
4. Fork this repository and update the following:
  - Add your Microshift node's IP address to the inventory in place of `10.1.1.11` that's there now in `inventory/hosts`
  - Decrypt the vaulted variables in `inventory/group_vars/node/vault.yml` by running the following commands from the repository root:
    ```sh
    vault_pass="definitely a secure password"
    mkdir -p /tmp/secrets
    echo "$vault_pass" > /tmp/secrets/.vault
    ansible-vault () {
        podman run --rm -it \
        --entrypoint ansible-vault \
        -v ./:/repo \
        -v /tmp/secrets/.vault:/tmp/secrets/.vault \
        -e ANSIBLE_VAULT_PASSWORD_FILE=/tmp/secrets/.vault \
        --security-opt=label=disable --privileged \
        --workdir /repo \
        registry.redhat.io/ansible-automation-platform-23/ee-supported-rhel8:1.0.0 \
        "${@}"
    }
    ansible-vault decrypt inventory/group_vars/node/vault.yml
    ```
  - Edit the file at inventory/group_vars/node/vault.yml using your editor of choice, updating `vaulted_ansible_ssh_pass` with the password of your user on MicroShift - one who is capable of escalating to root with `sudo` without a password
  - From the repository root, in the same shell where we defined the `ansible-vault` function before, run the following to encrypt your new vault variables using a password of your choice:
    ```sh
    vault_pass="a totally new password" # you can change this if you like
    echo "$vault_pass" > /tmp/secrets/.vault
    ansible-vault encrypt inventory/group_vars/node/vault.yml
    ```
  - Load the secret into your MicroShift cluster for consumption by the `CronJob` before we move on from this terminal now. If you can run `oc get nodes` and see your MicroShift cluster nodes, run something like the following from the same shell where we already set `vault_pass`:
    ```sh
    oc create namespace microshift-config
    oc create secret generic --from-literal=.vault="$vault_pass" vault-key -n microshift-config
    ```
  - In the ConfigMap named `microshift-config-env` in `deploy/cronjob.yml`, update `ANSIBLE_PULL_REPO_CONN` to point to your own fork of the repository, optionally updating the `ANSIBLE_PULL_CHECKOUT` key to your own branch or tag
  - Commit and push all of the above changes, ensuring you don't commit an unencrypted vault
5. Get ready to run Ansible against your MicroShift endpoint using `ansible-navigator` with the following:
  - Install `ansible-navigator` either from the AAP repositories on your RHEL host, pip on your non-RHEL host, or into a Python Virtual Environment at your own discretion
  - Recover the `KUBECONFIG` from your MicroShift node as listed in [the documentation](https://access.redhat.com/documentation/en-us/red_hat_build_of_microshift/4.12/html/installing/microshift-embed-in-rpm-ostree#accessing-microshift-cluster-remotely-non-admin_microshift-embed-in-rpm-ostree), placing it either in `~/.kube/config` as documented or some alternate location you will remember in the following commands
  - Place the passphrase you used to encrypt your vault somewhere on your local system, such as `/tmp/secrets/.vault` or, optionally, anywhere you like and will remember in the following commands
6. You can now use `ansible-navigator` to run playbooks leveraging the supported AAP 2.3 EE. You need to mount a few files into the EE on execution, because the configuration doesn't support some things that make it simpler to define there. Supposing you use `/tmp/secrets/.vault` for your vault passphrase, and your MicroShift `KUBECONFIG` is in `~/.kube/config`, you could run both playbooks serially like this:
  ```sh
  for playbook in playbooks/*.yml; do
      ansible-navigator run $playbook -l microshift,microshift_node --eev ~/.kube/config:/home/runner/.kube/config:Z --eev /tmp/secrets/.vault:/tmp/secrets/.vault:Z
  done
  ```
7. Observe that, after running the cluster playbook, a CronJob was created on the cluster to continuously enforce reconciliation with your forked repository. You can experiment with updating the defintions in `app/` and manually applying those updated definitions either with `ansible-navigator` or creating a fresh `Job` from your `CronJob` with a command like the following:
  ```sh
  oc delete job -n microshift-config microshift-config-001
  oc create job --from=cronjob/microshift-config -n microshift-config microshift-config-001
  ```

## Motivation

RHDE is expected to be deployed into small-form-factor edge devices with less than 16 GB of RAM and only a few CPU cores. MicroShift enables deploying applications to RHDE instances with a fully functional Kubernetes API, and some elements of the OpenShift API (namely, Security Context Constraints and Routes) in that minimal footprint.

One of the mechanisms that enables MicroShift to fit onto these devices is by stripping a large number of OpenShift features out of the final build. These include the MachineConfig operator, though, which means that RHDE deployments lack a native method to manage the underlying Operating System (OS).

One considered approach for managing the workloads on MicroShift is Red Hat Advanced Cluster Management (ACM), through which managed spoke Kubernetes deployments run "klusterlet" agents that call back to a centralized ACM Hub deployment for tasking. Because this method comes at some amount of overhead in the form of the klusterlet, and because it's incapable of managing the OS without MachineConfig, it's widely believed that AAP is the mechanism best suited for managing RHDE deployments.

The types of tasks that one might expect to perform on RHDE instances via AAP are as follows:

- Triggering and monitoring OS updates
- Triggering and monitoring application updates
- Initial configuration after late-binding provisioning (where multiple devices might perform one of many tasks determined at binding time, preventing configuration at initial install)
- Reconfiguration in the event of a change to desired fleet configuration
- Reconfiguration to support change, either of the node configuration itself or of some change in application deployment

AAP is capable of managing all of these aspects from a proper AAP Controller/Execution Mesh. The challenge then becomes that AAP is traditionally used in a push-based model, where an API is called out to or a node is SSH'd to. The nature of RHDE means that it will often be deployed into locations with poor or intermittent network connectivity. Sometimes they will be deployed on mobile platforms, and may be expected to be managed from many separate sites and need to "phone home" to receive new instructions, rather than being consistently reachable at a known IP address.

One possible solution is a VPN. Having a RHDE node with a transient/mobile connection that attempts to establish a VPN to controlled IP space and pulls a consistent and predictable IP address within that VPN would allow AAP to manage that endpoint well. One framework seeking to enable this flexibility is [Project Nexodus](https://github.com/nexodus-io/nexodus) (formerly Apex), but this project is still highly experimental, not supported by anyone, and has kinks remaining to work out - especially on the user experience and management side.

One other solution, and indeed the one that this repository seeks to demonstrate, is to leverage AAP components, MicroShift's native features, and GitOps workflows to manage the RHDE endpoint in an entirely pull-based model. This method lets us manage the underlying OS, the configuration and applications at the Kubernetes API, and do it at a massive scale by delegating management tasks to every individual managed node - without continuously running an agent, and instead only scheduling a pull and reapplication.

## Design

This repository is designed to show a few things about how this workflow is expected to work. It includes considerations for application deployment artifact distribution, inventory and secret management, and some traditional management tasks via Ansible playbooks. It includes the configurations necessary for inner-loop iterative development in a lab setting that is tightly aligned to the production GitOps-driven process, with the same AAP EE and content.

### Repository Structure

- app/
  - This directory contains the "production" application manifests. In this case, everything is managed in a monorepo. It may make sense to use an alternative deployment artifact, such as a helm chart, that is cached on the PVC and leveraged in the playbooks. This is not considered in this implementation, but the flexibility of the solution should allow it.
- deploy/cronjob.yml
  - This multi-YAML-document manifest defines everything necessary for an AAP Execution Environment (EE) to run at a regularly defined interval. The entrypoint, typically just `ansible-runner`, has been overridden to perform some `ansible-pull`-like tasks, attempting to update the repository that defines the deployment and saving it locally before executing playbooks against the host and API.
  - There are some important characteristics of this CronJob worth mentioning:
    - A ServiceAccount and the appropriate RBAC to manage the API and node are created. This is a very privileged ServiceAccount, so it is kept in an isolated namespace.
    - There are PVCs defined for the runner home directory (important for SSH key and host key management) and the project directory, for storing our Git repository to enable self-healing behavior even if the connection is out. These PVCs leverage the OpenShift LVM Operator configuration that's enabled by default in MicroShift, if you have an LVM Volume Group with adequate free space to support them.
    - A single ConfigMap defines most variables for the container. This ConfigMap describes the repository which should be tracked, the branch or tag that should be used, and information on which inventory items to limit execution to for the Node and Cluster. This kind of configuration would allow us to have a more complex inventory with our entire fleet defined, while limiting each node to manage only itself. Additionally, note that the `ansible-vault` password is defined in a specific path here.
    - Another ConfigMap is used to house our `ansible-pull.sh` script used in the modified entrypoint. The purpose of keeping this script in a ConfigMap is to allow for modification if necessary without requiring rebuilding a container image - and keeps us using only the supported EE without modification.
    - Finally, the CronJob is defined with a single container in the Pod template. This container runs all of our scripts serially, leverages `ansible-pull.sh` from our ConfigMap, pulls in environment from the other ConfigMap, mounts our PVCs, and importantly - mounts a Secret that was not defined here to `/tmp/secrets`. This secret was applied to the cluster out-of-band, and has an `ansible-vault` passphrase in it - allowing our CronJob to access the vaulted secrets in our inventory (mentioned below).
    - Notice that there is no `KUBECONFIG` definition. The `redhat.openshift` collection (and the upstream `kubernetes.core` collection it builds upon) are capable of using the ServiceAccount of their pod for normal Kubernetes API interactions. Because we're running this pod with a highly privileged ServiceAccount, it inherits those `cluster-admin` privileges automatically when executing against the cluster API. Our environment ConfigMap did override the port it uses, due to our `hostNetwork: true` configuration on the Pod template. This keeps our settings _very_ portable, because this same manifest should work against any cluster at all - as long as the inventory for the node works (SSH host, secrets, etc.).
- inventory/
  - This directory contains a typical Ansible inventory defintion, except that some important things are committed directly and some others are deliberately left out.
  - inventory/hosts
    - This is a bog-standard ini-style inventory, defining two groups and one host in each. The groups are used for convenience of reference, and the hosts are given some basic connection variables to enable the host in the `cluster` group to hit the Kubernetes API without trying to connect to any remote endpoints, and enforcing the SSH connection to the `node` group host.
  - inventory/group_vars/node/
    - This directory has two files, `vault.yml` and `vars.yml`. `vault.yml` is designed to hold secrets for connecting to the host while `vars.yml` is designed to clearly expose which secrets have been vaulted. This is a common pattern in ansible, because the contents of `vault.yml` are not readable without decryption, so the variable names encrypted within have been exposed to vars without exposing the content. Although variable content can be encrypted without encrypting a whole file using ansible-vault, it's simpler to encrypt files - and less likely to result in a leakage of secret content.
- playbooks/
  - This directory holds two simple Ansible playbooks. More could be included, and these two could be made more complex. They are deliberately simplified to enable easy understanding of the concepts without getting into the weeds of device management with Ansible.
- ansible-navigator.yml
  - This file allows one to consume the repo content, complete with the exact same Execution Environment defined
- ansible.cfg
  - This is a relatively plain Ansible configuration that defines a fixed location for our `ansible-vault` password and inventory and prevents `ansible-runner` from fussing about SSH host keys - since we're statically defining that the node only manage itself anyways.

## Feedback

This workflow is far from perfect. I am particularly concerned about the lack of observability in this workflow. Working within the constraints of a fully pull-based management model for RHDE endpoints, with their incredibly limited resources, can be a challenge. If there's a set of trade-offs that you feel would be better suited, please reach out to me at jharmison@redhat.com so we can discuss your requirements and expectations.
