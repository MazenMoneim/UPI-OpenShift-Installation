
# OpenShift Installation with User Provisioned Infrastructure (UPI) in Disconnected Lab



<p align="center">
  <strong>
    A step-by-step guide to deploying OpenShift Container Platform in a disconnected environment using UPI method.
  </strong>
  <br><br>
  Covers infrastructure preparation, DNS, load balancing, Quay registry setup, image mirroring, certificates, and cluster installation.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Red%20Hat-OpenShift-EE0000?logo=redhat&logoColor=white" alt="OpenShift"/>
  <img src="https://img.shields.io/badge/Installation-UPI-blue" alt="UPI"/>
  <img src="https://img.shields.io/badge/Environment-Disconnected-orange" alt="Disconnected Environment"/>
  <img src="https://img.shields.io/badge/Registry-Quay-FF5A1F?logo=redhat&logoColor=white" alt="Quay"/>
  <img src="https://img.shields.io/badge/Platform-Linux-black?logo=linux&logoColor=white" alt="Linux"/>
  <img src="https://img.shields.io/badge/Automation-Bash-4EAA25?logo=gnubash&logoColor=white" alt="Bash"/>
</p>

---



A 3-master / 2-worker OpenShift 4.18 cluster installed with the UPI method (`platform: none`) on VMware Workstation VMs, pulling all content from a local Red Hat Quay mirror registry.

- [Architecture](#architecture)
- [Quay Installation and OpenShift Image Mirroring](#quay-installation-and-openShift-image-mirroring)
- [Helper Node Services](#helper-node-services)
- [Prepare the install-config](#prepare-the-install-config)
- [Generate the Ignition files](#generate-the-ignition-files)
- [Deploy OpenShift](#deploy-openshift)
- [Monitor the Bootstrap Process](#monitor-the-bootstrap-process)
- [Remove the Bootstrap Node](#remove-the-bootstrap-node)
- [Join Worker Nodes](#join-worker-nodes)
- [Wait for Installation to Complete](#wait-for-installation-to-complete)
- [Reuse the Bootstrap VM as worker02](#reuse-the-bootstrap-vm-as-worker02)
- [Post-install](#post-install)
- [Troubleshooting – Issues Hit During This Install](#troubleshooting--issues-hit-during-this-install)
- [Lessons Learned](#lessons-learned)

---

## Architecture
<img width="2215" height="3116" alt="CamScanner 15-10-2026 16 47" src="https://github.com/user-attachments/assets/1718c3a3-f32d-4d73-9ce3-196f64bf19a2" />



---

## Quay Installation and OpenShift Image Mirroring

### 1. Install Quay

Download the `mirror-registry` utility from the Red Hat OpenShift
downloads page, then run:

``` bash
chmod +x ./mirror-registry
./mirror-registry install --quayHostname registry.ocp4.example.com --quayRoot "$HOME/quay-install"
```

Check the registry containers:

``` bash
podman ps
```

### 2. Check DNS and TLS

Ensure `registry.ocp4.example.com` resolves to the registry host IP from
the workstation and cluster nodes.

If Quay uses a private CA, install its CA certificate on the mirroring
host:

``` bash
sudo cp registry-ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

### 3. Log in

``` bash
podman login registry.redhat.io
podman login registry.access.redhat.com
podman login registry.ocp4.example.com
```

Use your Red Hat credentials for source registries and your Quay
credentials for the destination.

### 4. Mirror the images

Review the lab configuration (no changes needed):

``` bash
cat ~/DO0032L/labs/install-prepare/image-set-config.yaml
```
``` bash
---
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v2alpha1
mirror:
  platform:
    channels:
      - name: stable-4.18
        type: ocp
        minVersion: 4.18.6
        maxVersion: 4.18.6
    graph: true
  operators:
    - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.18
      packages:
        - name: node-maintenance-operator  1
          channels:
            - name: stable
              minVersion: 5.4.0
              maxVersion: 5.4.0
    - catalog: registry.redhat.io/redhat/certified-operator-index:v4.18
      packages:
        - name: crunchy-postgres-operator  2
          channels:
            - name: v5
              minVersion: 5.8.1
              maxVersion: 5.8.1
  additionalImages:
    - name: registry.access.redhat.com/ubi9/ubi:latest
    - name: registry.access.redhat.com/ubi9/toolbox:latest
```


Run `oc mirror`:

``` bash
oc mirror --v2 \
  --config ~/DO0032L/labs/install-prepare/image-set-config.yaml \
  --workspace file://mirror/ \
  docker://registry.ocp4.example.com
```

Keep the `mirror/` workspace; it contains resources needed for cluster
installation. If transient image-copy errors occur, review the failed
items and rerun the command.

### 5. Verify

Open `https://registry.ocp4.example.com`, sign in, and verify that the
`ubi9/toolbox` repository exists.

Check generated resources:

``` bash
ls mirror/working-dir/cluster-resources/
```

Common output includes `idms-oc-mirror.yaml` and `updateService.yaml`.

---
## Helper Node Services

### 1. DNS 
```bash

dnf install bind bind-utils -y
```

`/etc/named.conf`:

```
options {
        directory       "/var/named";
        allow-query { localhost; 192.168.50.0/24; 172.25.250.254; };
        recursion yes;
        forwarders { 172.25.250.254; };
        ...
};

zone "ocp4.example.com" {
    type master;
    file "ocp4.example.com.db";
    allow-update { none; };
};

zone "50.168.192.in-addr.arpa" IN {
    type master;
    file "ocp4.example.com.reverse.db";
    allow-update { none; };
};
```

`/var/named/ocp4.example.com.db` – forward zone (install phase, with bootstrap):

```
$TTL  1D
@     IN  SOA dns.ocp4.example.com. root.example.com. (
                2019022404 ; serial  <-- increment on EVERY change
                3h         ; refresh
                15         ; retry
                1w         ; expire
                3h         ; minimum
              )
         IN NS dns.ocp4.example.com.
dns      IN A 192.168.50.254
registry IN A 192.168.50.50
git      IN A 192.168.50.50
idm      IN A 192.168.50.40

; ---- cluster "upi" : api, api-int and *.apps MUST point to the load balancer ----
api.upi       IN A 192.168.50.254
api-int.upi   IN A 192.168.50.254
*.apps.upi    IN A 192.168.50.254

master01.upi  IN A 192.168.50.10
master02.upi  IN A 192.168.50.11
master03.upi  IN A 192.168.50.12
worker01.upi  IN A 192.168.50.13
bootstrap.upi IN A 192.168.50.14
```

`/var/named/ocp4.example.com.reverse.db` – reverse zone:

```
$TTL  1D
@     IN  SOA dns.ocp4.example.com. root.example.com. (
                2019022403 ; serial  <-- increment on EVERY change
                3h         ; refresh
                15         ; retry
                1w         ; expire
                3h         ; minimum
              )
          IN NS dns.ocp4.example.com.
254  IN PTR api.upi.ocp4.example.com.
254  IN PTR api-int.upi.ocp4.example.com.
50   IN PTR registry.ocp4.example.com.
40   IN PTR idm.ocp4.example.com.
10   IN PTR master01.upi.ocp4.example.com.
11   IN PTR master02.upi.ocp4.example.com.
12   IN PTR master03.upi.ocp4.example.com.
13   IN PTR worker01.upi.ocp4.example.com.
14   IN PTR bootstrap.upi.ocp4.example.com.
```


Validate and reload:

```bash
named-checkzone ocp4.example.com /var/named/ocp4.example.com.db
named-checkzone 50.168.192.in-addr.arpa /var/named/ocp4.example.com.reverse.db
systemctl reload named

dig +short api.upi.ocp4.example.com @192.168.50.254        # 192.168.50.254
dig +short api-int.upi.ocp4.example.com @192.168.50.254    # 192.168.50.254
dig +short test.apps.upi.ocp4.example.com @192.168.50.254  # 192.168.50.254
dig +short -x 192.168.50.10 @192.168.50.254                # master01.upi.ocp4.example.com.
```

### 2. DHCP + PXE

```bash
dnf install dhcp-server -y
```

`/etc/dhcp/dhcpd.conf` – fixed addresses per MAC; nodes PXE-boot from the utility (`next-server 192.168.50.254`):

```
ddns-update-style interim;
ignore client-updates;
authoritative;
allow booting;
allow bootp;
allow unknown-clients;
default-lease-time -1;
max-lease-time -1;

subnet 192.168.50.0 netmask 255.255.255.0 {
        range 192.168.50.100 192.168.50.149;
        option routers 192.168.50.254;
        option ntp-servers 103.16.182.23,103.16.182.214;
        option domain-search "ocp4.example.com";
        option domain-name-servers 192.168.50.254;
        filename "pxelinux.0";
        next-server 192.168.50.254;

        host master01.ocp4.example.com  { hardware ethernet 52:54:00:00:32:0A; fixed-address 192.168.50.10; option host-name "master01";  option domain-name "upi.ocp4.example.com"; }
        host master02.ocp4.example.com  { hardware ethernet 52:54:00:00:32:0B; fixed-address 192.168.50.11; option host-name "master02";  option domain-name "upi.ocp4.example.com"; }
        host master03.ocp4.example.com  { hardware ethernet 52:54:00:00:32:0C; fixed-address 192.168.50.12; option host-name "master03";  option domain-name "upi.ocp4.example.com"; }
        host worker01.ocp4.example.com  { hardware ethernet 52:54:00:00:32:0D; fixed-address 192.168.50.13; option host-name "worker01";  option domain-name "upi.ocp4.example.com"; }
        host bootstrap.ocp4.example.com { hardware ethernet 52:54:00:00:32:0E; fixed-address 192.168.50.14; option host-name "bootstrap"; option domain-name "upi.ocp4.example.com"; }
}
```

```bash
systemctl restart dhcpd
```

### 3. Web server (httpd on port 8080)
```bash
dnf install httpd -y
```
```bash
sed -i 's/Listen 80/Listen 0.0.0.0:8080/' /etc/httpd/conf/httpd.conf
```
HAProxy owns port 80, so httpd listens on 8080. It serves the RHCOS live `rootfs.img` used by the PXE boot (`coreos.live.rootfs_url=http://192.168.50.254:8080/attach_iso/rootfs.img`), and can also serve ignition files if you install with `--ignition-url`.

```bash
grep ^Listen /etc/httpd/conf/httpd.conf     # Listen 8080
systemctl enable --now httpd
curl -sI http://192.168.50.254:8080/attach_iso/rootfs.img | head -1   # 200 OK
```

> httpd is only needed while nodes are being booted/installed. The running cluster never uses it.

### 4. HAProxy
```bash
dnf install haproxy -y
```

`/etc/haproxy/haproxy.cfg` – OpenShift section, **install phase** (bootstrap included in 6443 and 22623):

```
listen stats
    bind :32700
    stats enable
    stats uri /
    stats hide-version
    stats auth admin:<password>

frontend k8s_api_ocp4
  bind 192.168.50.254:6443
  mode tcp
  default_backend k8s_api_backend_ocp4
backend k8s_api_backend_ocp4
  balance roundrobin
  mode tcp
  server bootstrap 192.168.50.14:6443 check
  server master01  192.168.50.10:6443 check
  server master02  192.168.50.11:6443 check
  server master03  192.168.50.12:6443 check

frontend machine_config_ocp4
  bind 192.168.50.254:22623
  mode tcp
  default_backend machine_config_backend_ocp4
backend machine_config_backend_ocp4
  balance roundrobin
  mode tcp
  server bootstrap 192.168.50.14:22623 check
  server master01  192.168.50.10:22623 check
  server master02  192.168.50.11:22623 check
  server master03  192.168.50.12:22623 check

frontend ingress_insecure_ocp4
  bind 192.168.50.254:80
  mode tcp
  default_backend ingress_insecure_backend_ocp4
backend ingress_insecure_backend_ocp4
  balance roundrobin
  mode tcp
  server master01 192.168.50.10:80 check
  server master02 192.168.50.11:80 check
  server master03 192.168.50.12:80 check
  server worker01 192.168.50.13:80 check

frontend ingress_secure_ocp4
  bind 192.168.50.254:443
  mode tcp
  default_backend ingress_secure_backend_ocp4
backend ingress_secure_backend_ocp4
  balance roundrobin
  mode tcp
  server master01 192.168.50.10:443 check
  server master02 192.168.50.11:443 check
  server master03 192.168.50.12:443 check
  server worker01 192.168.50.13:443 check
```

Notes:

- **6443 (API) and 22623 (Machine Config Server)** → bootstrap + masters only. Workers never serve these ports.
- **80 / 443 (ingress)** → the nodes that run router pods. Masters are included because they are schedulable in this cluster (see `compute.replicas: 0` below).

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg
setsebool -P haproxy_connect_any 1
systemctl enable --now haproxy
```


### 5. Firewall (if firewalld is enabled on the utility node)

```bash
firewall-cmd --add-port={6443,22623,8080}/tcp --add-service={http,https,dns,dhcp,tftp} --permanent
firewall-cmd --reload
```

---

## Prepare the install-config

On the **workstation**:

```bash
mkdir ~/upi-ocp4
```

`~/upi-ocp4/install-config.yaml`:

```yaml
apiVersion: v1
metadata:
  name: upi
baseDomain: ocp4.example.com
compute:
  - name: worker
    replicas: 0          # UPI: workers are added manually; with 0, masters become schedulable
controlPlane:
  name: master
  replicas: 3
networking:
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  networkType: OVNKubernetes
  serviceNetwork:
    - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths":{"registry.ocp4.example.com":{"auth":"<base64 user:password>"}}}'
sshKey: 'ssh-rsa AAAA...VJ1 student@workstation'      # contents of ~/.ssh/openshift.pub
imageDigestSources:
- mirrors:
  - registry.ocp4.example.com/openshift/release
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - registry.ocp4.example.com/openshift/release-images
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - registry.ocp4.example.com/workload-availability
  source: registry.redhat.io/workload-availability
- mirrors:
  - registry.ocp4.example.com/crunchydata
  source: registry.connect.redhat.com/crunchydata
```

### Add the mirror registry's CA (`additionalTrustBundle`)

The Quay registry's TLS certificate was signed by the lab's own root CA (`Classroom Root CA`, self-signed). OpenShift doesn't trust it by default, so without this every image pull from the mirror fails with `x509: certificate signed by unknown authority`.

```bash
echo 'additionalTrustBundle: |' >> ~/upi-ocp4/install-config.yaml
sed 's/^/  /' ~/DO0031L/labs/ca/ca-cert.pem >> ~/upi-ocp4/install-config.yaml
```

Result:

```yaml
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIJxjCCBa6gAwIBAgIUP2JFsrg4AgEeNZDtn1Ifr3GoMJMwDQYJKoZIhvcNAQEL
  ...
  Lw6wxPyuXifM+Q==
  -----END CERTIFICATE-----
```
Think of it like an ID card:
- Quay is a person showing an ID card (its certificate) that says “I am registry.ocp4.example.com”.
- Your CA is the office that issued and stamped that ID card. You created this office yourself in the earlier lesson.
- OpenShift is the security guard checking the ID.

The guard only accepts IDs stamped by offices it knows. It has never heard of your homemade office, so it rejects Quay’s ID with an error.
additionalTrustBundle is you handing the guard a sample of your office’s stamp and saying “trust IDs with this stamp.” Now when Quay shows its ID, the guard recognizes the stamp and lets it in.

**Back up the file** – the installer consumes (deletes) it:

```bash
cp ~/upi-ocp4/install-config.yaml ~/install-config.yaml.bak
```

---

## Generate the Ignition files

```bash
openshift-install create manifests --dir ~/upi-ocp4
openshift-install create ignition-configs --dir ~/upi-ocp4
ls ~/upi-ocp4
# auth/  bootstrap.ign  master.ign  metadata.json  worker.ign
```

> The certificates inside the ignition files are valid for **24 hours**. If the install isn't finished within that window, delete the install directory and regenerate everything.

SSH access to the nodes uses a non-default key name, so add it to `~/.ssh/config`:

```bash
cat >> ~/.ssh/config <<'EOF'
Host 192.168.50.*
    User core
    IdentityFile ~/.ssh/openshift
EOF
chmod 600 ~/.ssh/config
```

---

## Deploy OpenShift

> **Every master and worker must be installed to disk.** A node that only boots the RHCOS *live* image runs from RAM with a read-only OS and never joins the cluster (see [Issue 4](#issue-4--no-nodes-and-no-csrs-masters-running-live)). Only the bootstrap may run live.

### Bootstrap

PXE-boot the bootstrap VM (192.168.50.14) with `bootstrap.ign`. It can run from the live image, or be installed to disk the same way as the masters.

### Masters

1. PXE-boot each master VM into the RHCOS live environment.
2. From the **workstation**, copy `master.ign` to each node and install RHCOS to disk:

   ```bash
   for ip in 10 11 12; do
     scp ~/upi-ocp4/master.ign core@192.168.50.$ip:
     ssh core@192.168.50.$ip \
       'sudo coreos-installer install /dev/sda --ignition-file /var/home/core/master.ign && sudo poweroff'
   done
   ```

3. Power the masters on. On first boot from disk they fetch their full config from `api-int.upi.ocp4.example.com:22623`, run firstboot, reboot once, and start the kubelet.

#### Alternative: self-installing ISO per role (no httpd, no typing)

```bash
ISO=~/Downloads/rhcos-418.94.202501221327-0-live.x86_64.iso
for role in bootstrap master worker; do
  coreos-installer iso customize --dest-device /dev/sda \
    --dest-ignition ~/upi-ocp4/$role.ign -o $role.iso $ISO
done
```

Boot each VM from its role ISO → it installs to `/dev/sda` → detach the ISO → boot from disk.

> `--dest-ignition` / `coreos.inst.ignition_url=` → installs to disk 
> `ignition.config.url=` / `iso ignition embed` with a node config → configures the **live** system only 

---

## Monitor the Bootstrap Process

From the **workstation**:

```bash
openshift-install --dir ~/upi-ocp4 wait-for bootstrap-complete --log-level=debug
```
<img width="1118" height="442" alt="image" src="https://github.com/user-attachments/assets/bb151cb6-9f7b-46a5-9d98-ef8bd8cc800a" />

In a second terminal:

```bash
export KUBECONFIG=~/upi-ocp4/auth/kubeconfig
oc get nodes                 # master01-03 should appear and go Ready
oc get csr                   # master CSRs are auto-approved during bootstrap
oc get co
```
<img width="716" height="184" alt="image" src="https://github.com/user-attachments/assets/2c5a82b9-ed69-4d16-a897-0d2ef15234f1" />
<img width="884" height="223" alt="image" src="https://github.com/user-attachments/assets/4110731d-aa5c-4db1-90b4-b6a93bbb1655" />

On the bootstrap node itself:

```bash
ssh core@192.168.50.14 'journalctl -b -f -u release-image.service -u bootkube.service'
```

Bootstrap completes once etcd runs on **all three** masters. Typical duration: 20–30 minutes (the 45 min in the installer output is only a timeout). Re-running `wait-for bootstrap-complete` is safe.

---

## Remove the Bootstrap Node

1. On the **utility** node, delete the two `server bootstrap ...` lines (6443 and 22623 backends):

   ```bash
   vi /etc/haproxy/haproxy.cfg
   haproxy -c -f /etc/haproxy/haproxy.cfg && systemctl reload haproxy
   ```

2. Power off the bootstrap VM.

---

## Join Worker Nodes

1. PXE-boot worker01 (192.168.50.13) into the live environment.
2. From the **workstation** (the `scp` runs on the workstation, not on the node):

   ```bash
   scp ~/upi-ocp4/worker.ign core@192.168.50.13:
   ssh core@192.168.50.13 \
     'sudo coreos-installer install /dev/sda --ignition-file /var/home/core/worker.ign && sudo poweroff'
   ```

3. Disconnect the CD / set the disk first in the boot order, then power on.
4. Approve CSRs – **two rounds** (client cert first, then `kubelet-serving` about a minute later):

   ```bash
   watch "oc get csr | grep Pending"
   oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs -r oc adm certificate approve
   # wait ~1 minute, run the approve command again
   oc get nodes
   ```


---

## Wait for Installation to Complete

```bash
openshift-install --dir ~/upi-ocp4 wait-for install-complete
watch oc get co        # all AVAILABLE=True, PROGRESSING=False, DEGRADED=False
```

The output prints the console URL and the `kubeadmin` password (also in `~/upi-ocp4/auth/kubeadmin-password`).
<img width="1294" height="266" alt="image" src="https://github.com/user-attachments/assets/10e5caad-3b9d-41d5-88f9-cbea6cb65464" />

---

## Reuse the Bootstrap VM as worker02

The bootstrap VM keeps its MAC (`52:54:00:00:32:0E`) and IP (`.14`), and is re-provisioned as a worker.

**DNS** – replace the bootstrap records and **bump both serials**:

```
; ocp4.example.com.db
worker02.upi IN A 192.168.50.14

; ocp4.example.com.reverse.db
14  IN PTR worker02.upi.ocp4.example.com.
```

**DHCP** – rename the host entry:

```
host worker02.ocp4.example.com { hardware ethernet 52:54:00:00:32:0E; fixed-address 192.168.50.14; option host-name "worker02"; option domain-name "upi.ocp4.example.com"; }
```

**HAProxy** – add worker02 to the **ingress backends only** (80 → port 80, 443 → port 443):

```
backend ingress_insecure_backend_ocp4
  ...
  server worker02 192.168.50.14:80 check

backend ingress_secure_backend_ocp4
  ...
  server worker02 192.168.50.14:443 check
```

Apply on the utility node:

```bash
named-checkzone ocp4.example.com /var/named/ocp4.example.com.db
named-checkzone 50.168.192.in-addr.arpa /var/named/ocp4.example.com.reverse.db
systemctl reload named
systemctl restart dhcpd
haproxy -c -f /etc/haproxy/haproxy.cfg && systemctl reload haproxy
```

---

## Post-install

### Disable the default OperatorHub catalogs (disconnected)

The default catalog pods try to pull from `registry.redhat.io` and loop in `Init:ImagePullBackOff` in `openshift-marketplace`. Harmless, but noisy:

```bash
oc patch operatorhub cluster --type merge -p '{"spec":{"disableAllDefaultSources":true}}'
```

To install operators later, mirror a catalog into `registry.ocp4.example.com` with `oc-mirror` and create a `CatalogSource` pointing at it.

### Image registry storage

On `platform: none` the internal registry starts as `Removed`. For a lab:

```bash
oc patch configs.imageregistry.operator.openshift.io cluster --type merge \
  -p '{"spec":{"managementState":"Managed","storage":{"emptyDir":{}}}}'
```

(`emptyDir` loses images when the pod restarts; use a PVC for anything persistent.)

### Console access from the workstation

```bash
sudo tee -a /etc/hosts <<'EOF'
192.168.50.254 api.upi.ocp4.example.com api-int.upi.ocp4.example.com
192.168.50.254 console-openshift-console.apps.upi.ocp4.example.com oauth-openshift.apps.upi.ocp4.example.com
EOF
```

Console: `https://console-openshift-console.apps.upi.ocp4.example.com`
<img width="1714" height="927" alt="Untitled picture" src="https://github.com/user-attachments/assets/2ed17a21-0de8-4300-b150-781bed61239e" />

### Clean up

Remove ignition files from any web root (`bootstrap.ign` contains cluster credentials). httpd can be stopped if no more nodes will be added.

---

## Troubleshooting – Issues Hit During This Install

### Issue 1 – Masters stuck fetching ignition: `connection refused` on 22623

**Symptom** (master console):

```
ignition[798]: GET error: Get "https://api-int.upi.ocp4.example.com:22623/config/master":
dial tcp 192.168.50.10:22623: connect: connection refused
```

**Cause:** the zone had `api.upi` / `api-int.upi` as **CNAMEs to `master01.upi`** (and `*.apps.upi` → .10). master01 was asking *itself* for its config. The correct `api`/`api-int` → .254 records existed, but for the wrong name (`api.ocp4.example.com`, without the cluster name `upi`).

**Fix:** replace the CNAMEs with A records pointing to HAProxy, bump the serial, reload:

```
api.upi      IN A 192.168.50.254
api-int.upi  IN A 192.168.50.254
*.apps.upi   IN A 192.168.50.254
```

> BIND refuses to load a zone that has both a CNAME and other records for the same name, so delete the CNAMEs, don't just add A records next to them. If `dig` returns nothing after a change, run `named-checkzone`.

### Issue 2 – `wait-for` keeps hitting the old IP after DNS was fixed

**Symptom** (workstation):

```
Still waiting for the Kubernetes API: Get "https://api.upi.ocp4.example.com:6443/version":
dial tcp 192.168.50.10:6443: connect: connection refused
```

**Cause:** the workstation resolves through the classroom DNS **172.25.250.254**, not the utility node. That server had cached the old CNAME with a 1-day TTL (`dig` showed TTL `83622`, counting down).

**Diagnose:**

```bash
getent hosts api.upi.ocp4.example.com
cat /etc/resolv.conf
dig api.upi.ocp4.example.com      # check ";; SERVER:" and the TTL
```

**Fix:** bypass the cache with `/etc/hosts` (see [Console access](#console-access-from-the-workstation)), or `rndc flushname api.upi.ocp4.example.com` on 172.25.250.254 if you have root there. Don't edit `/etc/resolv.conf` – cloud-init manages it.

### Issue 3 – `ssh core@<node>`: `Permission denied (publickey)`

**Cause:** the cluster key is `~/.ssh/openshift`, which SSH doesn't offer by default.

**Diagnose:** compare the key the masters are served with yours:

```bash
curl -sk https://192.168.50.254:22623/config/master | jq -r '.passwd.users[].sshAuthorizedKeys[]'
cat ~/.ssh/openshift.pub
```

**Fix:** `ssh -i ~/.ssh/openshift core@...`, or add it to `~/.ssh/config` (see [Generate the Ignition files](#generate-the-ignition-files)).

### Issue 4 – No nodes and no CSRs (masters running live)

**Symptom:** the bootstrap API is up for a long time, but:

```
$ oc get nodes
No resources found
$ oc get csr
No resources found
```

On a master, `machine-config-daemon-firstboot` loops forever and the kubelet is `inactive`:

```
error: failed to remove pending deployment: error running rpm-ostree cleanup -p:
... Read-only file system
Sleeping 1 minute for retry
```

**Cause:** the masters were PXE-booted into the RHCOS **live** image with the ignition embedded and were never installed to disk. The live root is read-only, so the MCO firstboot pivot can't complete, the kubelet never starts, and no CSR is ever sent.

**Diagnose:**

```bash
ssh core@192.168.50.10 'cat /proc/cmdline; lsblk; sudo rpm-ostree status'
```

Live if: `coreos.live.rootfs_url=` in the cmdline, `/` and `/sysroot` on `loop` devices, and `sda` with no partitions.

**Fix:** install to disk with `coreos-installer` and boot from disk (see [Masters](#masters)). After that the masters joined within minutes and bootstrap completed.

### Issue 5 – `coreos-installer`: `opening source Ignition config ... No such file or directory`

**Cause:** the `scp` of `worker.ign` was skipped – it must run from the **workstation**, not on the node.

**Fix:** `scp ~/upi-ocp4/worker.ign core@192.168.50.13:` from the workstation, then run the install.

> Before running `coreos-installer` on a node, make sure it is really running live (`cat /proc/cmdline | grep coreos.live`). Running it against the disk an installed node booted from wipes that node.

### Issue 6 – `openshift-marketplace` pods in `Init:ImagePullBackOff`

**Cause:** default catalog index images come from `registry.redhat.io`, which a disconnected cluster can't reach.

**Fix:** see [Disable the default OperatorHub catalogs](#disable-the-default-operatorhub-catalogs-disconnected). Check the exact reason with `oc describe pod -n openshift-marketplace <pod> | grep -A3 -i failed`.


---

## Lessons Learned

1. `api`, `api-int` and `*.apps` for **`<cluster>.<baseDomain>`** must point to the load balancer – never to a single node.
2. Bump the zone serial and run `named-checkzone` on every DNS change.
3. Know which resolver each machine uses; upstream caches can serve stale records for the full TTL.
4. Bootstrap may run live; **masters and workers must be installed to disk** (`coreos-installer`, `coreos.inst.*` kargs, or `iso customize --dest-ignition`).
5. "No nodes and no CSRs" means the problem is on the nodes, not the bootstrap – SSH in and check `machine-config-daemon-firstboot` and `kubelet`.
6. HAProxy: 6443 / 22623 → bootstrap + masters; 80 / 443 → nodes running routers.
7. Ignition certificates expire after 24 hours – finish the install (or regenerate) within that window.
