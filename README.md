# openshift-vm-guest

```bash
#!/bin/bash
[[ -z "${RHSM_USERNAME}" ]] && exit 1
[[ -z "${RHSM_PASSWORD}" ]] && exit 1
[[ -z "${OC_USERNAME}" ]] && exit 1
[[ -z "${OC_PASSWORD}" ]] && exit 1
[[ -z "${RHEL_IMAGE}" ]] && exit 1
[[ -z "${RHEL_CHECKSUM}" ]] && exit 1
[[ -z "${EXT_IP_SUBNET}" ]] && exit 1
[[ -z "${EXT_IP_BOOT_VM}" ]] && exit 1

ssh-keygen -f "~/.ssh/known_hosts" -R "${EXT_IP_BOOT_VM}" &>/dev/null

[[ -f "${RHEL_IMAGE}" ]] || exit 1
ISO_CHECKSUM=$(sha256sum ${RHEL_IMAGE}|cut -d" " -f1|xargs)
[[ "${RHEL_CHECKSUM,,}" == "${ISO_CHECKSUM,,}" ]] || exit 1 
[[ -f "boot.qcow2" ]] && rm -f boot.qcow2 &>/dev/null
cp -f ${RHEL_IMAGE} boot.qcow2 &>/dev/null
QCOW2_VSIZE=$(qemu-img info boot.qcow2 |grep "virtual size:"|cut -d":" -f2|cut -d" " -f2|xargs)
[[ "${QCOW2_VSIZE}" != "128" ]] && qemu-img resize boot.qcow2 128G

oc whoami &>/dev/null
[[ $? -eq 1 ]] && oc login -u ${OC_USERNAME} -p ${OC_PASSWORD} --server=https://api.<cluster>:6443 --insecure-skip-tls-verify=true

oc get project -q example &>/dev/null
if [[ $? -eq 1 ]] ; then
  oc new-project example &>/dev/null 
  oc adm policy add-scc-to-group anyuid system:serviceaccounts:example &>/dev/null
fi

# install MetalLB if not already deployed
oc project -q metallb-system &>/dev/null
if [[ $? -eq 1 ]] ; then
  cat << EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
EOF
  cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: metallb-operator
  namespace: metallb-system
EOF
  cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: metallb-operator-sub
  namespace: metallb-system
spec:
  channel: stable
  name: metallb-operator
  source: redhat-operators 
  sourceNamespace: openshift-marketplace
EOF
  oc label ns metallb-system "openshift.io/cluster-monitoring=true"
  while : ; do
    METALLB_STATUS=$(oc get clusterserviceversion -n metallb-system -o custom-columns=Name:.metadata.name,Phase:.status.phase|grep -i metallb-operator|awk '{print $2}')
    [[ "${METALLB_STATUS,,}" == "succedded" ]] && break
    sleep 30s
  done
  cat << EOF | oc apply -f -
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: metallb-system
EOF
  while : ; do
    oc get deployment -n metallb-system controller &>/dev/null && \
    oc get daemonset -n metallb-system speaker &>/dev/null && \
    break
    sleep 30s
  done
fi

oc describe -n metallb-system IPAddressPool ip-pool-example &>/dev/null
if [[ $? -eq 1 ]] ; then
  cat << EOF | oc apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  namespace: metallb-system
  name: ip-pool-example
spec:
  addresses:
  - ${EXT_IP_SUBNET}
  autoAssign: false
  avoidBuggyIPS: true
  serviceAllocation:
    namespaces:
      - example
EOF
fi

oc get -n metallb-system l2advertisement l2-adv-example &>/dev/null
if [[ $? -eq 1 ]] ; then
 cat << EOF | oc apply -f -
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-adv-example
  namespace: metallb-system
spec:
  ipAddressPools:
    - ip-pool-example
EOF
fi

oc project example &>/dev/null
oc get dv boot &>/dev/null
[[ $? -eq 1 ]] && virtctl image-upload dv boot \
                          --size=250Gi \
                          --storage-class=lvms-vg1 \
                          --image-path=boot.qcow2 \
                          --insecure \
                          --force-bind

oc get vm boot &>/dev/null
if [ $? -eq 1 ] ; then
  # using sshAuthorizedKeys via OCP/KV cloud-init blocks connections,
  # so generate authorizedKeys via file contents instead.
  # large cloudInit user-config is limited to 2048 rather than 16384,
  # so must run cloudInit runcmd's via SSH.
  cat << EOF | oc create -f - 
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  annotations:
    openshift.io/display-name: "Guest VM (boot) example"
    openshift.io/provider-display-name: "LC"
    kubevirt.io/latest-observed-api-version: v1
    kubevirt.io/storage-observed-api-version: v1alpha3
    vm.kubevirt.io/validations: |
      [
        {
          "name": "minimal-required-memory",
          "path": "jsonpath::.spec.domain.memory.guest",
          "rule": "integer",
          "message": "This VM requires more memory. (4GB)",
          "min": 419430400
        }
      ]
  generation: 1
  labels:
    example/app: mgmt-appliance
    kubevirt.io/dynamic-credentials-support: "true"
    vm.kubevirt.io/template: rhel9-server-medium
    vm.kubevirt.io/template.namespace: openshift
    vm.kubevirt.io/template.revision: "1"
    vm.kubevirt.io/template.version: v0.26.0
  name: boot
  namespace: example
spec:
  running: true
  template:
    metadata:
      annotations:
        vm.kubevirt.io/flavor: medium
        vm.kubevirt.io/os: rhel9
        vm.kubevirt.io/workload: server
      labels:
        example/app: mgmt-appliance
        kubevirt.io/domain: boot
        kubevirt.io/size: medium
    spec:
      architecture: amd64
      RunStrategy: Always
      domain:
        cpu:
          cores: 4
          sockets: 1
          threads: 1
        devices:
          autoattachGraphicsDevice: true
          disks:
            - disk:
                bus: virtio
              name: disk-boot
            - disk:
                bus: virtio
              name: cloudinitdisk
          interfaces:
            - name: default
              masquerade: {}
              model: virtio
              ports:
                - name: ssh
                  protocol: TCP
                  port: 22
                - name: dns
                  protocol: UDP
                  port: 53
                - name: dhcpds
                  protocol: UDP
                  port: 67
                - name: dhcpdc
                  protocol: UDP
                  port: 68
                - name: http
                  protocol: TCP
                  port: 80
                - name: ks
                  protocol: TCP
                  port: 3000
                - name: mr
                  protocol: TCP
                  port: 8443
          networkInterfaceMultiqueue: true
          rng: {}
        features:
          acpi: {}
          smm:
            enabled: true
        firmware:
          bootloader:
            efi: {}
        machine:
          type: pc-q35-rhel9.2.0
        memory:
          guest: 4Gi
        resources: {}
      hostname: boot
      networks:
        - name: default
          pod: {}
      terminationGracePeriodSeconds: 180
      volumes:
        - name: disk-boot
          persistentVolumeClaim:
            claimName: boot
        - name: cloudinitdisk
          cloudInitNoCloud:
            networkData: |
              ethernets: 1
                eth0:
                  dhcpd4: true
            userData: |
              #cloud-config
              user: root
              password: Pass1234!1
              ssh_pwauth: False
              chpasswd:
                expire: false
              growpart:
                mode: auto
                devices: ["/"]
                ignore_growroot_disabled: false
              write_files:
              - encoding: b64
                path: /root/.ssh/authorized_keys
                content: ...==
                owner: "root:root"
                permissions: "0600"
              - encoding: b64
                path: /root/.ssh/example
                content: ...==
                owner: "root:root"
                permissions: "0400"
              - encoding: b64
                path: /root/.ssh/example.pub
                content: ...==
                owner: "root:root"
                permissions: "0444"
              - encoding: b64
                path: /root/.ssh/config
                content: ...==
                owner: "root:root"
                permissions: "0400"
EOF
fi
[[ $? -eq 0 ]] || exit

cat << EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
  name: boot
  namespace: example
  annotations:
    metallb.universe.tf/ip-allocated-from-pool: ip-pool-example
spec:
  externalTrafficPolicy: Cluster
  ipFamilies:
    - IPv4
  ipFamilyPolicy: SingleStack
  allocateLoadBalancerNodePorts: true
  ports:
  - port: 22
    protocol: TCP
    targetPort: 22
    name: ssh
  - port: 53
    protocol: UDP
    targetPort: 53
    name: dns
  - port: 67
    protocol: UDP
    targetPort: 67
    name: dhcpds
  - port: 68
    protocol: UDP
    targetPort: 68
    name: dhcpdc
  - port: 80
    protocol: TCP
    targetPort: 80
    name: http
  - port: 3000
    protocol: TCP
    targetPort: 3000
    name: ks
  - port: 8443
    protocol: TCP
    targetPort: 8443
    name: mr
  selector:
    example/app: mgmt-appliance
    kubevirt.io/domain: boot
    kubevirt.io/size: medium
  type: LoadBalancer
  loadBalancerIP: ${EXT_IP_BOOT_VM}
EOF

ssh-keygen -f ~/.ssh/known_hosts -R "vmi/boot.example" &>/dev/null

while : ; do
 virtctl ssh --namespace=example -l=root -i=~/.ssh/example -t "-o StrictHostKeyChecking=accept-new" -t "-o ConnectTimeout=30" -c "exit" boot &>/dev/null
 [[ $? -eq 0 ]] && break
 sleep 30s
done

virtctl ssh --namespace=example -l=root -i=~/.ssh/example-deploy \
        -c "subscription-manager register --username ${RHSM_USERNAME} --password ${RHSM_PASSWORD}" boot 
virtctl ssh --namespace=example -l=root -i=~/.ssh/example -c "dnf -y update 2>&1|tee /var/log/bootstrap.log" boot
virtctl ssh --namespace=example -l=root -i=~/.ssh/example -c "dnf install -y git git-lfs qemu-guest-agent 2>&1|tee -a /var/log/bootstrap.log" boot
virtctl ssh --namespace=example -l=root -i=~/.ssh/example -c "ssh -o StrictHostKeyChecking=accept-new -T -p 22 github.com-bootstrap 2>&1|tee -a /var/log/bootstrap.log" boot
virtctl ssh --namespace=example -l=root -i=~/.ssh/example -c "git clone git@github.com-bootstrap:<org>/<repo>bootstrap.git /root/<repo> 2>&1|tee -a /var/log/bootstrap.log" boot
virtctl ssh --namespace=example -l=root -i=~/.ssh/example -c "cd /root/<repo>/ && git lfs pull 2>&1|tee -a /var/log/bootstrap.log" boot
virtctl ssh --namespace=example -l=root -i=~/.ssh/example -c "chmod +x /root/<repo>/bootstrap/*.sh /root/<repo>/kickstart/*.sh 2>&1|tee -a /var/log/bootstrap.log" boot
virtctl ssh --namespace=example -l=root -i=~/.ssh/example -c "/root/<repo>/bootstrap/install.sh 2>&1|tee -a /var/log/bootstrap.log" boot
```
