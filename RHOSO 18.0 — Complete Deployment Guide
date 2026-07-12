# RHOSO 18.0 — Complete Deployment Guide (Final)

**Concepts + Operators + deployable sample YAMLs, tuned for a disconnected environment.**
*(Written so a fresher needs no follow-up questions. Replace all placeholder IPs/passwords/FQDNs with real values. Always verify against current Red Hat docs before production use.)*

---

## Glossary — Read This First

| Term | What it is |
|---|---|
| **RHOCP** | Red Hat OpenShift Container Platform — the Kubernetes cluster RHOSO runs on top of |
| **RHOSO** | Red Hat OpenStack Services on OpenShift — OpenStack's control plane, containerized, running as pods on RHOCP |
| **Multus CNI** | Built into RHOCP; lets a pod have more than one network interface (default pod network + isolated networks) |
| **NMState / NNCP** | NMState = operator/API for node-level networking. NNCP (NodeNetworkConfigurationPolicy) = the YAML that actually configures a NIC/VLAN/bond on a node (cluster-scoped) |
| **NAD (NetworkAttachmentDefinition)** | Multus resource — exposes a node-level network to pods (namespaced) |
| **MetalLB / IPAddressPool / L2Advertisement** | Bare-metal load balancer. IPAddressPool = VIP ranges it may assign. L2Advertisement = advertises those VIPs via ARP |
| **cert-manager** | Issues/manages TLS certificates for RHOSO service endpoints (TLS-everywhere) |
| **Cluster Observability Operator** | Required when Telemetry is enabled (default) |
| **NetConfig** | RHOSO resource — master definition of all isolated networks (ctlplane, internalapi, storage, tenant, external) and their subnets/VLANs |
| **ctlplane network** | **Mandatory.** Used by the OpenStack Operator for Ansible SSH to the data plane, and for instance live migration. No VLAN by default |
| **dnsmasq (dns service)** | DNS service in the control plane that serves the RHOSO nodes; gets a MetalLB VIP on the ctlplane pool |
| **OpenStackControlPlane (CR)** | Top-level CR telling `openstack-operator` which OpenStack services to deploy and how (Keystone, Nova, Galera, RabbitMQ, etc.) |
| **OpenStackDataPlaneNodeSet (CR)** | Declares the external RHEL compute/hypervisor hosts, their networks, and the ordered `services` list Ansible runs on them |
| **OpenStackDataPlaneDeployment (CR)** | The "go" button — triggers the Ansible run against the NodeSet |
| **edpm** | "External Data Plane Management" — prefix on the Ansible vars/services that configure the RHEL data plane nodes |
| **preProvisioned** | NodeSet flag: `true` = you already installed RHEL on the nodes; `false` = RHOCP's Cluster Baremetal Operator provisions them via BareMetalHost CRs |
| **Satellite** | Red Hat content/patch platform — serves **RPMs** to the RHEL data plane. Supported by RHOSO **as a package repository only, NOT as a container image registry** |
| **Mirror registry (Quay)** | The disconnected **container image** registry for all OpenShift + RHOSO images (separate from Satellite) |
| **oc-mirror / IDMS / ICSP / CatalogSource** | The disconnected image-mirroring toolchain: mirror images, then tell the cluster to redirect pulls (ImageDigestMirrorSet / ImageContentSourcePolicy) and see operators (CatalogSource) |
| **ODF / Ceph / RHCS** | Storage. ODF must run in **external mode** for RHOSO. Ceph backs Cinder (volumes), Glance (images), Manila (shares), Nova (VM disks) |
| **extraMounts** | Field in RHOSO CRs that mounts external files (e.g., Ceph keyrings) into a service's pod |

---

## 1. Prerequisites (checklist)
- RHOCP cluster **v4.18**, bare metal — compact (3 nodes, both roles) or 3 dedicated control-plane + 3 dedicated worker nodes
- `cluster-admin` access, `oc` CLI on workstation
- **Per worker node: enough NICs** — ctlplane and external don't use VLANs by default, so they need separate NICs (or native VLAN on a trunk). Internalapi/storage/tenant ride tagged VLANs. Bond NICs for redundancy where possible
- FIPS decided at OCP install time if required (cannot be toggled later)
- Multus CNI — already present in RHOCP
- **Disconnected:** a mirror registry (Quay) populated with the OpenShift + RHOSO container payload (see §2a), and a Satellite (or local RPM mirror) for the data plane RHEL content

> **Note:** Linux bridges are **not supported** for RHOSO data plane traffic. Use Linux **bonds** and/or dedicated NICs.

---

## 2a. Disconnected: Mirror the Container Images First ⭐ (air-gapped only)

Before OperatorHub can even show the OpenStack Operator, the images and catalog must exist in your local registry and the cluster must be told to use them.

1. **Mirror with `oc-mirror`** using an `ImageSetConfiguration` that includes the OpenShift release, the `redhat-operators` catalog (filtered to the OpenStack Operator + dependencies: NMState, MetalLB, cert-manager, Cluster Observability, ODF), and any additional images. In an air gap: mirror to disk on a connected host → carry across → push to the mirror registry.
2. Applying the `oc-mirror` output installs three things on the cluster:
   - **`ImageDigestMirrorSet` (IDMS)** / legacy `ImageContentSourcePolicy` (ICSP) — redirects `registry.redhat.io`/`quay.io` pulls to your mirror
   - **`CatalogSource`** — so OperatorHub lists the mirrored operators
   - Updated cluster **pull secret** — merged Red Hat + local registry auth
3. Add the mirror registry CA to the cluster trust (`image.config.openshift.io/cluster` `additionalTrustedCA`).

Only after this does §2 (installing operators from OperatorHub) work offline.

---

## 2. Install Required Operators (RHOCP web console → OperatorHub)

**Mandatory:**
1. **Kubernetes NMState Operator**
2. **MetalLB Operator**
3. **cert-manager Operator**
4. **Cluster Observability Operator** (unless Telemetry disabled)
5. **OpenStack Operator (`openstack-operator`)** — meta-operator into `openstack-operators` namespace; manages all service operators (Keystone, Nova, Cinder, Galera, RabbitMQ, etc.)

**Storage (choose one):**
6. **ODF Operator** (external mode only) — connects to external Ceph, or
7. **LVM Storage Operator** — if no Ceph, provides a local-volume storage class (node-local, weaker HA)

**Optional (HA hardening):** Node Health Check, Self Node Remediation, Fence Agents Remediation Operators.

**Activate NMState:**
```yaml
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
```
**Activate MetalLB:**
```yaml
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: metallb-system
```
**Why:** Installing an operator only places its controller; these instances switch the controllers "on" (NMState starts watching NNCPs; MetalLB starts its speaker pods).

---

## 3. Create the `openstack` Namespace ⭐ (before any namespaced YAML)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openstack
```
**Why:** Every RHOSO namespaced resource (NetConfig, NAD, Secrets, ControlPlane, DataPlane CRs) lives here. Kept separate from `openstack-operators` (operator pods only).

**Pre-flight check — network policy:** Ensure no NetworkPolicy blocks traffic between `openstack-operators` and `openstack`, or the control plane will never reconcile:
```bash
oc get networkpolicy -n openstack
oc get networkpolicy -n openstack-operators
```

---

## 4. Networking Setup

### Step A — `netconfig.yaml`: define ALL networks (note ctlplane + external)
```yaml
apiVersion: network.openstack.org/v1beta1
kind: NetConfig
metadata:
  name: netconfig
  namespace: openstack
spec:
  networks:
  - name: ctlplane                      # MANDATORY — Ansible SSH + live migration
    dnsDomain: ctlplane.example.com
    subnets:
    - name: subnet1
      cidr: 192.168.122.0/24
      gateway: 192.168.122.1
      allocationRanges:
      - start: 192.168.122.100
        end: 192.168.122.120
      - start: 192.168.122.150
        end: 192.168.122.200
  - name: internalapi
    dnsDomain: internalapi.example.com
    subnets:
    - name: subnet1
      cidr: 172.17.0.0/24
      vlan: 20
      allocationRanges:
      - start: 172.17.0.100
        end: 172.17.0.250
  - name: storage
    dnsDomain: storage.example.com
    subnets:
    - name: subnet1
      cidr: 172.18.0.0/24
      vlan: 21
      allocationRanges:
      - start: 172.18.0.100
        end: 172.18.0.250
  - name: tenant
    dnsDomain: tenant.example.com
    subnets:
    - name: subnet1
      cidr: 172.19.0.0/24
      vlan: 22
      allocationRanges:
      - start: 172.19.0.100
        end: 172.19.0.250
  - name: external                      # optional but usual (FIPs / provider nets)
    dnsDomain: external.example.com
    subnets:
    - name: subnet1
      cidr: 10.0.0.0/24
      gateway: 10.0.0.1
      allocationRanges:
      - start: 10.0.0.100
        end: 10.0.0.250
```
**Why:** Single source of truth for every network. **Rule:** NetConfig allocationRange must NOT overlap the MetalLB IPAddressPool range or the NAD ipam range.

### Step B — `nncp.yaml`: configure node NICs (bond + ctlplane + VLANs)
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: worker-0-netcfg
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-0
  desiredState:
    interfaces:
    # Bond for redundancy (LACP; use active-backup if switch lacks LACP)
    - name: bond0
      type: bond
      state: up
      link-aggregation:
        mode: 802.3ad
        options:
          miimon: "100"
          lacp_rate: fast
        port:
        - enp1s0
        - enp2s0
      ipv4:
        enabled: false
      ipv6:
        enabled: false
    # ctlplane — no VLAN by default; IP on the bond (or a dedicated NIC)
    - name: ctlplane
      type: linux-bridge          # or place ctlplane IP directly on bond0 / dedicated NIC
      state: up
      ipv4:
        enabled: true
        address:
        - ip: 192.168.122.10
          prefix-length: 24
      bridge:
        port:
        - name: bond0
    # InternalAPI VLAN on the bond
    - name: internalapi
      type: vlan
      state: up
      vlan:
        base-iface: bond0
        id: 20
      ipv4:
        enabled: true
        address:
        - ip: 172.17.0.10
          prefix-length: 24
    # Storage VLAN
    - name: storage
      type: vlan
      state: up
      vlan:
        base-iface: bond0
        id: 21
      ipv4:
        enabled: true
        address:
        - ip: 172.18.0.10
          prefix-length: 24
    # Tenant VLAN
    - name: tenant
      type: vlan
      state: up
      vlan:
        base-iface: bond0
        id: 22
      ipv4:
        enabled: true
        address:
        - ip: 172.19.0.10
          prefix-length: 24
```
**Why:** NetConfig only describes networks; NNCP makes them real on each node. IPs sit on the VLAN/ctlplane interfaces, not the raw bond. One NNCP per node. **NNCP apply order matters** — if you split into multiple policies, name them so alphanumeric sorting brings dependencies (the bond) up before what rides on them. (Linux bridge shown only as the ctlplane carrier here; bridges are not used for tenant/data-plane traffic.)

### Step C — `nad.yaml`: expose each network to pods (one per network)
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: internalapi
  namespace: openstack
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "internalapi",
      "type": "macvlan",
      "master": "internalapi",
      "ipam": {
        "type": "whereabouts",
        "range": "172.17.0.0/24",
        "range_start": "172.17.0.30",
        "range_end": "172.17.0.70"
      }
    }
```
Repeat for `ctlplane`, `storage`, `tenant` (and `octavia` if used). **Why:** NNCP configured the network at OS level; NAD lets service pods attach a second interface to it via Multus.

### Step D — MetalLB VIPs
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ctlplane
  namespace: metallb-system
spec:
  addresses:
  - 192.168.122.80-192.168.122.90
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: internalapi
  namespace: metallb-system
spec:
  addresses:
  - 172.17.0.80-172.17.0.90
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: openstack-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - ctlplane
  - internalapi
```
**Why:** Reserves + advertises VIPs for control plane service endpoints (Keystone internal, dnsmasq, etc.). Ranges must not overlap NetConfig/NAD ranges.

---

## 5. Storage Class

**Option A — ODF external mode:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ocs-storagecluster-ceph-rbd
provisioner: openshift-storage.rbd.csi.ceph.com
parameters:
  clusterID: openshift-storage
  pool: ocs-storagecluster-cephblockpool
reclaimPolicy: Delete
volumeBindingMode: Immediate
```
**Option B — LVM Storage (no Ceph):**
```yaml
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: lvmcluster
  namespace: openshift-storage
spec:
  storage:
    deviceClasses:
    - name: vg1
      default: true
      thinPoolConfig:
        name: thin-pool-1
        sizePercent: 90
        overprovisionRatio: 10
```
**Why:** RHOSO control plane pods (Galera, RabbitMQ, OVN DBs) need PVs. ODF external = production; LVM = labs/POC, node-local, weaker HA.

---

## 6. Storage Backend (Ceph) Preparation — before Control Plane
1. Verify external Ceph is healthy
2. Create pools per service — `vms` (Nova), `volumes` (Cinder), `images` (Glance)
3. Create a Ceph client keyring + obtain the FSID
4. Package keyring + `ceph.conf` into a Secret (§7) for services to mount via `extraMounts`

---

## 7. Secrets (in the `openstack` namespace)

**`osp-secret.yaml`** — all service passwords:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: osp-secret
  namespace: openstack
type: Opaque
data:
  AdminPassword: cGFzc3dvcmQ=
  DbRootPassword: cGFzc3dvcmQ=
  KeystoneDatabasePassword: cGFzc3dvcmQ=
  CinderPassword: cGFzc3dvcmQ=
  NovaPassword: cGFzc3dvcmQ=
  NeutronPassword: cGFzc3dvcmQ=
  PlacementPassword: cGFzc3dvcmQ=
  GlancePassword: cGFzc3dvcmQ=
```
*(base64 for "password" — real use: `echo -n 'yourpassword' | base64`)*

**`ceph-secret.yaml`** (name `ceph-conf-files`): the base64 keyring + `ceph.conf`.

**Data plane secrets** (used in §9):
- `dataplane-ansible-ssh-private-key-secret` — SSH key Ansible uses to reach compute hosts
- `subscription-manager` — registration inputs for the RHEL nodes
- `redhat-registry` — pull credentials for data plane container images

---

## 8. Control Plane

**`openstackcontrolplane.yaml`** — now includes the DB, messaging, cache, and DNS layers that were missing before:
```yaml
apiVersion: core.openstack.org/v1beta1
kind: OpenStackControlPlane
metadata:
  name: openstack
  namespace: openstack
spec:
  secret: osp-secret
  storageClass: ocs-storagecluster-ceph-rbd

  # TLS-everywhere via cert-manager
  tls:
    podLevel:
      enabled: true

  # DNS service (dnsmasq) — gets a ctlplane VIP
  dns:
    template:
      override:
        service:
          metadata:
            annotations:
              metallb.universe.tf/address-pool: ctlplane
              metallb.universe.tf/allow-shared-ip: ctlplane
              metallb.universe.tf/loadBalancerIPs: 192.168.122.80
          spec:
            type: LoadBalancer
      options:
      - key: server
        values:
        - 192.168.122.1        # upstream DNS reachable from the dnsmasq pod
      replicas: 2

  # MariaDB Galera — one for services, one for Compute cell1
  galera:
    templates:
      openstack:
        secret: osp-secret
        storageRequest: 5000M
        replicas: 3
      openstack-cell1:
        secret: osp-secret
        storageRequest: 5000M
        replicas: 3

  rabbitmq:
    templates:
      rabbitmq:
        replicas: 3
      rabbitmq-cell1:
        replicas: 3

  memcached:
    templates:
      memcached:
        replicas: 3

  keystone:
    template:
      databaseInstance: openstack
      secret: osp-secret

  glance:
    template:
      databaseInstance: openstack
      storage:
        storageRequest: 10G
      extraMounts:
      - name: ceph
        extraVol:
        - propagation: [Glance]
          extraVolType: Ceph
          volumes:
          - name: ceph
            secret:
              secretName: ceph-conf-files
          mounts:
          - name: ceph
            mountPath: "/etc/ceph"
            readOnly: true

  cinder:
    template:
      cinderVolumes:
        volume1:
          customServiceConfig: |
            [volume1]
            volume_backend_name=volume1
            volume_driver=cinder.volume.drivers.rbd.RBDDriver
            rbd_pool=volumes

  placement:
    template: {}

  nova:
    template: {}

  neutron:
    template: {}

  ovn:
    template: {}

  telemetry:
    template: {}
```
**Why:** This single CR is reconciled by `openstack-operator` into all services. Galera (databases), RabbitMQ (messaging), Memcached (cache), and dnsmasq (DNS) are **infrastructure the API services depend on** — omitting them means the control plane never comes up. `tls.podLevel` drives cert-manager. An **OpenStackClient pod** is created automatically for CLI access (see §10).

---

## 9. Data Plane (RHEL 9.4 compute/hypervisor hosts)

The control plane is OpenStack's brain (pods on RHOCP). The data plane is separate RHEL servers running customer VMs — configured remotely via Ansible over the ctlplane network.

### 9a. RPM registration — CDN vs Satellite (disconnected) ⭐
Each hypervisor must register for RPMs. This runs inside the NodeSet's `edpm_bootstrap_command` (which only executes because `bootstrap` is in the `services` list).

**Disconnected (Satellite 6.13)** — only the first two lines differ from the CDN flow (katello CA + activation key instead of CDN user/pass):
```bash
rpm -Uvh http://<satellite-fqdn>/pub/katello-ca-consumer-latest.noarch.rpm
subscription-manager register --org="<ORG_LABEL>" --activationkey="<ACTIVATION_KEY>"
subscription-manager release --set=9.4
subscription-manager repos --disable=*
subscription-manager repos \
  --enable=rhel-9-for-x86_64-baseos-eus-rpms \
  --enable=rhel-9-for-x86_64-appstream-eus-rpms \
  --enable=rhel-9-for-x86_64-highavailability-eus-rpms \
  --enable=fast-datapath-for-rhel-9-x86_64-rpms \
  --enable=rhoso-18.0-for-rhel-9-x86_64-rpms \
  --enable=rhceph-7-tools-for-rhel-9-x86_64-rpms
```
**Satellite prep (before applying the NodeSet):** sync those 6 repos → publish a Content View + Lifecycle Environment → create an Activation Key → ensure hypervisors can resolve/route to the Satellite FQDN over ctlplane. **Boundaries:** Satellite = RPMs only (never container images — those come from your mirror registry via `redhat-registry`). Its edge is **content pinning** so RPMs never drift out of sync with container versions.

### 9b. `openstackdataplanenodeset.yaml` — with `services`, `preProvisioned`, per-node networks, Satellite bootstrap
```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneNodeSet
metadata:
  name: openstack-edpm
  namespace: openstack
spec:
  preProvisioned: true          # true = RHEL already installed; false = provision via BareMetalHost
  networkAttachments:
  - ctlplane
  # Ordered Ansible services that run on each node
  services:
  - bootstrap
  - download-cache
  - configure-network
  - validate-network
  - install-os
  - configure-os
  - ssh-known-hosts
  - run-os
  - install-certs
  - ovn
  - neutron-metadata
  - libvirt
  - nova
  - telemetry
  nodeTemplate:
    ansibleSSHPrivateKeySecret: dataplane-ansible-ssh-private-key-secret
    ansible:
      ansibleUser: cloud-admin
      ansiblePort: 22
      ansibleVarsFrom:
      - prefix: subscription_manager_
        secretRef:
          name: subscription-manager
      - secretRef:
          name: redhat-registry
      ansibleVars:
        edpm_ceph_client_config: true
        edpm_network_config_template: |
          ---
          {% set mtu_list = [ctlplane_mtu] %}
          {% for network in nodeset_networks %}
          {{ mtu_list.append(lookup('vars', networks_lower[network] ~ '_mtu')) }}
          {%- endfor %}
          {% set min_viable_mtu = mtu_list | max %}
          network_config:
          - type: ovs_bridge
            name: br-ex
            use_dhcp: false
            addresses:
            - ip_netmask: {{ ctlplane_ip }}/{{ ctlplane_cidr }}
            routes: {{ ctlplane_host_routes }}
            members:
            - type: interface
              name: nic1
              primary: true
            {% for network in nodeset_networks %}
            - type: vlan
              vlan_id: {{ lookup('vars', networks_lower[network] ~ '_vlan_id') }}
              addresses:
              - ip_netmask: {{ lookup('vars', networks_lower[network] ~ '_ip') }}/{{ lookup('vars', networks_lower[network] ~ '_cidr') }}
            {% endfor %}
        edpm_bootstrap_command: |
          rpm -Uvh http://<satellite-fqdn>/pub/katello-ca-consumer-latest.noarch.rpm
          subscription-manager register --org="<ORG_LABEL>" --activationkey="<ACTIVATION_KEY>"
          subscription-manager release --set=9.4
          subscription-manager repos --disable=*
          subscription-manager repos \
            --enable=rhel-9-for-x86_64-baseos-eus-rpms \
            --enable=rhel-9-for-x86_64-appstream-eus-rpms \
            --enable=rhel-9-for-x86_64-highavailability-eus-rpms \
            --enable=fast-datapath-for-rhel-9-x86_64-rpms \
            --enable=rhoso-18.0-for-rhel-9-x86_64-rpms \
            --enable=rhceph-7-tools-for-rhel-9-x86_64-rpms
  nodes:
    edpm-compute-0:
      hostName: edpm-compute-0
      ansible:
        ansibleHost: 192.168.122.100
      networks:
      - name: ctlplane
        subnetName: subnet1
        fixedIP: 192.168.122.100
        defaultRoute: true
      - name: internalapi
        subnetName: subnet1
        fixedIP: 172.17.0.100
      - name: storage
        subnetName: subnet1
        fixedIP: 172.18.0.100
      - name: tenant
        subnetName: subnet1
        fixedIP: 172.19.0.100
    edpm-compute-1:
      hostName: edpm-compute-1
      ansible:
        ansibleHost: 192.168.122.101
      networks:
      - name: ctlplane
        subnetName: subnet1
        fixedIP: 192.168.122.101
        defaultRoute: true
      - name: internalapi
        subnetName: subnet1
        fixedIP: 172.17.0.101
      - name: storage
        subnetName: subnet1
        fixedIP: 172.18.0.101
      - name: tenant
        subnetName: subnet1
        fixedIP: 172.19.0.101
```
**Why:** The `services` list is the ordered Ansible run; `bootstrap` is what fires the Satellite registration. `preProvisioned: true` says you installed RHEL yourself (set `false` + add `baremetalSetTemplate`/`bmhLabelSelector` to have RHOCP provision the hosts). Each node needs an explicit `networks` block with predictable `fixedIP`s inside the NetConfig allocation ranges, and `defaultRoute: true` on ctlplane.

### 9c. `openstackdataplanedeployment.yaml` — the trigger
```yaml
apiVersion: dataplane.openstack.org/v1beta1
kind: OpenStackDataPlaneDeployment
metadata:
  name: openstack-edpm-deploy
  namespace: openstack
spec:
  nodeSets:
  - openstack-edpm
```
**Why:** Applying this runs the Ansible services on every node — register to Satellite, pull RPMs, configure networking, join Ceph, start nova/libvirt/ovn. Without it the NodeSet is an inert declaration.

---

## 10. Validation
```bash
# Control plane pods healthy
oc get pods -n openstack

# CLI runs INSIDE the auto-created OpenStackClient pod
oc rsh -n openstack openstackclient
  openstack endpoint list
  openstack network agent list
  exit

# Data plane reconciled
oc get openstackdataplanenodeset,openstackdataplanedeployment -n openstack

# On a compute node: confirm Satellite registration + repos
subscription-manager status
dnf repolist
```
Then launch a test instance to validate Nova + Neutron + Glance + Cinder end-to-end.

---

## Full Deployment Order (final)
1. RHOCP v4.18 bare-metal cluster ready (NIC/VLAN + FIPS decided at install)
2. **Disconnected:** mirror container images with `oc-mirror`; apply IDMS/ICSP + CatalogSource + trusted CA; prep Satellite (repos + Content View + Activation Key)
3. Install operators: NMState, MetalLB, cert-manager, Cluster Observability, ODF or LVM, OpenStack Operator (+ optional HA trio)
4. Activate NMState + MetalLB instances
5. **Create `openstack` namespace** ⭐; verify no blocking NetworkPolicy
6. Apply `NetConfig` (**ctlplane + internalapi + storage + tenant + external**)
7. Apply `NNCP` per node (**bond + ctlplane + VLANs**)
8. Apply `NAD` for each network
9. Apply MetalLB `IPAddressPool`(s) + `L2Advertisement`
10. Apply `StorageClass`; prep Ceph pools/keyring
11. Create secrets: `osp-secret`, `ceph-conf-files`, `dataplane-ansible-ssh-private-key-secret`, `subscription-manager`, `redhat-registry`
12. Apply `OpenStackControlPlane` (**incl. galera, rabbitmq, memcached, dns, telemetry, tls**) — control plane comes up
13. Apply `OpenStackDataPlaneNodeSet` (**services list + preProvisioned + per-node networks + Satellite bootstrap**)
14. Apply `OpenStackDataPlaneDeployment` — nodes register to Satellite, pull RPMs, get configured
15. Validate via `oc rsh openstackclient`, `subscription-manager status`, test VM launch

---

## Two Supply Lines (the mental model)
- **Container images** (OpenShift + RHOSO control plane pods) → disconnected **Quay/mirror-registry** (via `oc-mirror` + IDMS/CatalogSource)
- **RPMs** (RHEL data plane / hypervisor nodes) → **Satellite** (package repo only, never a container registry)
