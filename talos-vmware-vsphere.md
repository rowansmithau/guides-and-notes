# talos build process for vmware vsphere

## sources: 
 - https://www.talos.dev/v1.3/introduction/getting-started - this will be referenced as GSG (getting started guide) in steps below.
 - https://www.talos.dev/v1.3/talos-guides/install/virtualized-platforms/vmware/ - this will be referenced as VMG (vmware guide) in steps below.
 - the official docs are lacking

## assumptions made:
 - you have `talosctl` installed
 - you have `kubectl` installed
 - you have `govc` installed
 - you know what IP(s) will be used
 - they will be static IP's
 - you have created vsphere credentials that can create libraries, vm's, etc.
 - you know what an FQDN is, and what value you are going to use for the control plane node FQDN / VIP FQDN.
 - 'tk' is the name I used for this demo cluster.

## This guide will use steps from both docs linked above

1. (GSG) Complete the `Decide the Kubernetes Endpoint` step and create any required DNS records.
2. (GSG) Generate and save secrets via `talosctl gen secrets -o secrets.yaml`.
3. (VMG) Complete the `Generating Base Configurations` step: download `cp.patch.yaml`, apply your customisations to `cp.patch.yaml` (I removed the VIP, set dhcp to false, defined address statically etc).
   - The syntax I used for generating initial config was `talosctl gen config --with-secrets secrets.yaml tk https://control-plane.fqdn.com:6443 --config-patch-control-plane @cp.patch.yaml`.
   - You will now have a base configuration for the control-plane-1 node defined.
4. (GSG) Begin the `Machine Configs as Templates` step, create as many worker patch files (worker1.patch, worker2.patch) as needed, customise them accordingly (i.e. static IP)
5. (GSG) Finish the `Machine Configs as Templates` step by creating the worker yaml's: `for i in {1..3}; do talosctl gen config --with-secrets secrets.yaml --config-patch-worker @worker$i.patch --output-types worker -o worker-$i.yaml tk https://control-plane.fqdn.com:6443; done`
   - The command above will read `worker1.patch`, `worker2.patch` and `worker3.patch` and output files called `worker-1.yaml, worker-2.yaml and worker-3.yaml`.
6. (VMG) Validate your configs: `talosctl validate --config controlplane.yaml --mode cloud` and `for i in {1..3}; do talosctl validate --config worker-$i.yaml --mode cloud; done`.
6. (VMG) Set your `govc` environment variables.
7. (VMG) Set the TALOS_VERSION environment variable and confirm it matches your desired version, i.e. `export TALOS_VERSION=v1.3.0; echo $TALOS_VERSION`.
8. (VMG) In the `Manual Approach` step, complete all steps under the `Import the OVA into vCenter` heading.
10. (VMG) Complete the `Create the Bootstrap Node` and `Update Hardware Resources for the Bootstrap Node` steps.
    - A one liner that can achieve this: `govc library.deploy <library name>/talos-${TALOS_VERSION} control-plane-1; govc vm.change -e "guestinfo.talos.config=$(cat controlplane.yaml | base64)" -e "disk.enableUUID=1" -vm control-plane-1; govc vm.change -c 2 -m 4096 -vm control-plane-1; govc vm.disk.change -vm control-plane-1 -disk.name disk-1000-0 -size 10G; govc vm.power -on control-plane-1`
11. (VMG) Complete the `Create the Remaining Control Plane Nodes` step.
    - A one liner that can achieve this: `for i in {1..3}; do govc library.deploy tk/talos-${TALOS_VERSION} worker-$i; govc vm.change -e "guestinfo.talos.config=$(base64 worker-$i.yaml)" -e "disk.enableUUID=1" -vm worker-$i; govc vm.change -c 2 -m 8192 -vm worker-$i; govc vm.disk.change -vm worker-$i -disk.name disk-1000-0 -size 50G; done`
12. (GSG) Move the `talosconfig` file into it's home: `talosctl config merge ./talosconfig`.
13. (VMG) Complete the `Bootstrap Cluster` step: `talosctl --talosconfig talosconfig bootstrap -e <control plane IP> -n <control plane IP>`
14. (VMG) Complete the `Retrieve the kubeconfig` step: 
    - `talosctl --talosconfig talosconfig config endpoint <control plane IP>`
    - `talosctl --talosconfig talosconfig config node <control plane IP>`
    - `talosctl --talosconfig talosconfig kubeconfig .`
15. (GSG) Have `talosctl` write the kubeconfig file to enable `kubectl` functionality: `talosctl kubeconfig`
16. Missing from docs: Ensure that you get a `Ready` result when querying the control plane node via `kubectl get no`
17. (VMG) Complete the `Configure talos-vmtoolsd` step: `talosctl --talosconfig talosconfig -n <control plane IP> config new vmtoolsd-secret.yaml --roles os:admin; kubectl -n kube-system create secret generic talos-vmtoolsd-config --from-file=talosconfig=./vmtoolsd-secret.yaml; rm vmtoolsd-secret.yaml; kubectl -n kube-system delete po -l=app=talos-vmtoolsd`. 
18. Power on the worker nodes: `for i in {1..3}; do govc vm.power -on worker-$i; done`
19. Give it a couple of minutes and check that the nodes have joined the cluster and report their status as `Ready` via `kubectl get no`
20. Confirm vmtoolsd daemonset is now running (one pod for each node): `kubectl get po -n kube-system -l=app=talos-vmtoolsd`
