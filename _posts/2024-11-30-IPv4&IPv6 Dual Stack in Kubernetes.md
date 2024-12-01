Today, I was asked by a teammate to help troubleshoot an issue related to dual-stack in Kubernetes. It turned out that I am quite so unfamiliar with this subject. Actually, I didn't know some very basic things about it.  Hence I spent some time studying it. This post is a summary of it.

## What is IPv4/IPv6 dual-stack?

IPv4/IPv6 dual-stack networking enables the allocation of both IPv4 and IPv6 addresses to Pods and Services.

IPv4/IPv6 dual-stack networking is enabled by default for the Kubernetes cluster starting in 1.21, allowing the simultaneous assignment of both IPv4 and IPv6 addresses.

IPv4/IPv6 dual-stack on Kubernetes cluster provides the following features:

    Dual-Stack Pod networking(a single IPv4 and IPv6 address assignment per Pod)
    IPv4 and IPv6 enabled Services
    Pod off-cluster egress routing(eg. the Internet) via both IPv4 and IPv6 interfaces.
    
IPv4/IPv6 Dual-Stack Prerequisites
  Kubernetes 1.20 or later

  Provider support for dual-stack networking(Cloud provider or otherwise must be able to provide Kubernetes nodes with routable IPv4/IPv6 network interfaces)

  A network plugin that supports dual-stack networking.

## Configure IPv4/IPv6 dual-stack
To configure IPv4/IPv6 dual-stack, set dual-stack cluster network assignments:

    kube-apiserver:
                    --service-cluster-ip-range=<IPv4 CIDR>, <IPv6 CIDR>

    kube-controller-manager:
                   --cluster-cird=<IPv4 CIDR>,<IPv6 CIDR>
                   --service-cluster-ip-range=<IPv4 CIDR>,<IPv6 CIDR>
                   --node-cidr-mask-size-ipv4|--node-cidr-mask-size-ipv6 defaults to /24 for IP/4 and /64 for   IPv6
    
    kube-proxy:
                   --cluster-cidr=<IPv4 CIDR>,<IPv6 CIDR>

    kubelet:
                   --node-ip=<IPv4 IP>,<IPv6 IP>
                        This option is required for bare metal dual-stack nodes(nodes that do not define a cloud  provider with the --cloud-provider flag). If you are using a cloud provider and choose to  override the node IPs chosen by the cloud provider, set the --node-ip option.


You can check this configuration in your Kubernetes cluster. 
Firstly, run the command: 

        kubectl get nodes

Then select a control-plane node, and log on it. And run the following command on it:

         ```ps -ef | grep kube```

You will see the process detail of kube-apiserver,  kube-proxy,  kubelet, and kube-controller-manager, for example:

```
eric@eric~$ ps -ef | grep kube
root        2972    1054  1 11:09 ?        00:09:51 /var/lib/minikube/binaries/v1.31.0/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --config=/var/lib/kubelet/config.yaml --hostname-override=minikube --kubeconfig=/etc/kubernetes/kubelet.conf --node-ip=192.168.49.2
root        3516    3496  1 11:09 ?        00:08:52 etcd --advertise-client-urls=https://192.168.49.2:2379 --cert-file=/var/lib/minikube/certs/etcd/server.crt --client-cert-auth=true --data-dir=/var/lib/minikube/etcd --experimental-initial-corrupt-check=true --experimental-watch-progress-notify-interval=5s --initial-advertise-peer-urls=https://192.168.49.2:2380 --initial-cluster=minikube=https://192.168.49.2:2380 --key-file=/var/lib/minikube/certs/etcd/server.key --listen-client-urls=https://127.0.0.1:2379,https://192.168.49.2:2379 --listen-metrics-urls=http://127.0.0.1:2381 --listen-peer-urls=https://192.168.49.2:2380 --name=minikube --peer-cert-file=/var/lib/minikube/certs/etcd/peer.crt --peer-client-cert-auth=true --peer-key-file=/var/lib/minikube/certs/etcd/peer.key --peer-trusted-ca-file=/var/lib/minikube/certs/etcd/ca.crt --proxy-refresh-interval=70000 --snapshot-count=10000 --trusted-ca-file=/var/lib/minikube/certs/etcd/ca.crt
root        3580    3542  2 11:09 ?        00:15:19 kube-apiserver --advertise-address=192.168.49.2 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/var/lib/minikube/certs/ca.crt --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota --enable-bootstrap-token-auth=true --etcd-cafile=/var/lib/minikube/certs/etcd/ca.crt --etcd-certfile=/var/lib/minikube/certs/apiserver-etcd-client.crt --etcd-keyfile=/var/lib/minikube/certs/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --kubelet-client-certificate=/var/lib/minikube/certs/apiserver-kubelet-client.crt --kubelet-client-key=/var/lib/minikube/certs/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/var/lib/minikube/certs/front-proxy-client.crt --proxy-client-key-file=/var/lib/minikube/certs/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/var/lib/minikube/certs/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=8443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/var/lib/minikube/certs/sa.pub --service-account-signing-key-file=/var/lib/minikube/certs/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/var/lib/minikube/certs/apiserver.crt --tls-private-key-file=/var/lib/minikube/certs/apiserver.key
root        3674    3624  1 11:09 ?        00:06:42 kube-controller-manager --allocate-node-cidrs=true --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf --bind-address=127.0.0.1 --client-ca-file=/var/lib/minikube/certs/ca.crt --cluster-cidr=10.244.0.0/16 --cluster-name=mk --cluster-signing-cert-file=/var/lib/minikube/certs/ca.crt --cluster-signing-key-file=/var/lib/minikube/certs/ca.key --controllers=*,bootstrapsigner,tokencleaner --kubeconfig=/etc/kubernetes/controller-manager.conf --leader-elect=false --requestheader-client-ca-file=/var/lib/minikube/certs/front-proxy-ca.crt --root-ca-file=/var/lib/minikube/certs/ca.crt --service-account-private-key-file=/var/lib/minikube/certs/sa.key --service-cluster-ip-range=10.96.0.0/12 --use-service-account-credentials=true
root        3716    3691  0 11:09 ?        00:00:56 kube-scheduler --authentication-kubeconfig=/etc/kubernetes/scheduler.conf --authorization-kubeconfig=/etc/kubernetes/scheduler.conf --bind-address=127.0.0.1 --kubeconfig=/etc/kubernetes/scheduler.conf --leader-elect=false
root        4779    4683  0 11:09 ?        00:00:04 /usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/config.conf --hostname-override=minikube
root       45275       1  0 14:42 ?        00:00:01 snapfuse /var/lib/snapd/snaps/kubeadm_3513.snap /snap/kubeadm/3513 -o ro,nodev,allow_other,suid
eric      125908     687  0 21:22 pts/0    00:00:00 grep kube
eric@E-5CG1422YTH:~$ ps -ef | grep kube | grep range
root        3580    3542  2 11:09 ?        00:15:20 kube-apiserver --advertise-address=192.168.49.2 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/var/lib/minikube/certs/ca.crt --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota --enable-bootstrap-token-auth=true --etcd-cafile=/var/lib/minikube/certs/etcd/ca.crt --etcd-certfile=/var/lib/minikube/certs/apiserver-etcd-client.crt --etcd-keyfile=/var/lib/minikube/certs/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --kubelet-client-certificate=/var/lib/minikube/certs/apiserver-kubelet-client.crt --kubelet-client-key=/var/lib/minikube/certs/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/var/lib/minikube/certs/front-proxy-client.crt --proxy-client-key-file=/var/lib/minikube/certs/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/var/lib/minikube/certs/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=8443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/var/lib/minikube/certs/sa.pub --service-account-signing-key-file=/var/lib/minikube/certs/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/var/lib/minikube/certs/apiserver.crt --tls-private-key-file=/var/lib/minikube/certs/apiserver.key
root        3674    3624  1 11:09 ?        00:06:42 kube-controller-manager --allocate-node-cidrs=true --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf --bind-address=127.0.0.1 --client-ca-file=/var/lib/minikube/certs/ca.crt --cluster-cidr=10.244.0.0/16 --cluster-name=mk --cluster-signing-cert-file=/var/lib/minikube/certs/ca.crt --cluster-signing-key-file=/var/lib/minikube/certs/ca.key --controllers=*,bootstrapsigner,tokencleaner --kubeconfig=/etc/kubernetes/controller-manager.conf --leader-elect=false --requestheader-client-ca-file=/var/lib/minikube/certs/front-proxy-ca.crt --root-ca-file=/var/lib/minikube/certs/ca.crt --service-account-private-key-file=/var/lib/minikube/certs/sa.key --service-cluster-ip-range=10.96.0.0/12 --use-service-account-credentials=true
```
Note:

An example of an IPv4 CIDR: 10.244.0.0/16

An example of an IPv6 CIDR: fdxy:IJKL:MNOP:15::/64


## Services
You can create Services that can use IPv4, IPv6, or both.

The address family of a Service defaults to the address family of the first service cluster IP range(configured via the --service-cluster-ip-range flag to the kube-apiserver)

When you define a Service you can optionally configure it as dual stack. To specify the behavior you want, you set the .spec.ipFamilyPolicy field to one of the following values:

    SingleStack: Single-stack service. The control plane allocates a cluster IP for the Service, using the first configured service cluster IP range.

    PreferDualStack: Allocate both IPv4 and IPv6 cluster IPs for the Service when dual-stack is enabled. If dual-stack is not enabled or supported, it falls back to single-stack behavior.

    RequiredDualStack: Allocate Service .spec.clusterIPs from both IPv4 and IPv6 address ranges when dual-stack is enabled. If dual-stack is not enabled or supported, the Service API object creation failes.
            Selects the .spec.clusterIP from the list of .spec.clusterIPs based on the address family of the first element in the .spec.ipFamilies array.

If you would like to define which IP family to use for single stack or define the order of IP families for dual-stack, you can choose the address families by setting an optional field, .spec.ipFamilies, on the Service.

NOTE:
The .spec.ipFamilies field is conditonally mutable: you can add or remove a secondary IP address family, but you cannot change the primary IP address family of an existing Service.

You can set .spec.ipFamilies to any of the following array values:

    ["IPv4"]
    ["IPv6"]
    ["IPv4", "IPv6"]
    ["IPv6:, "IPv4"]

The first family you list is used for the legacy .spec.clusterIP field.

The process described above is implemented in the initIPFamilyFields function of alloc.go in Kubernetes source code:

```go
// attempts to default service ip families according to cluster configuration
// while ensuring that provided families are configured on cluster.
func (al *Allocators) initIPFamilyFields(after After, before Before) error {
    oldService, service := before.Service, after.Service

    // can not do anything here
    if service.Spec.Type == api.ServiceTypeExternalName {
        return nil
    }

    // We don't want to auto-upgrade (add an IP) or downgrade (remove an IP)
    // PreferDualStack services following a cluster change to/from
    // dual-stackness.
    //
    // That means a PreferDualStack service will only be upgraded/downgraded
    // when:
    // - changing ipFamilyPolicy to "RequireDualStack" or "SingleStack" AND
    // - adding or removing a secondary clusterIP or ipFamily
    if isMatchingPreferDualStackClusterIPFields(after, before) {
        return nil // nothing more to do.
    }

    // If the user didn't specify ipFamilyPolicy, we can infer a default.  We
    // don't want a static default because we want to make sure that we never
    // change between single- and dual-stack modes with explicit direction, as
    // provided by ipFamilyPolicy.  Consider these cases:
    //   * Create (POST): If they didn't specify a policy we can assume it's
    //     always SingleStack.
    //   * Update (PUT): If they didn't specify a policy we need to adopt the
    //     policy from before.  This is better than always assuming SingleStack
    //     because a PUT that changes clusterIPs from 2 to 1 value but doesn't
    //     specify ipFamily would work.
    //   * Update (PATCH): If they didn't specify a policy it will adopt the
    //     policy from before.
    if service.Spec.IPFamilyPolicy == nil {
        if oldService != nil && oldService.Spec.IPFamilyPolicy != nil {
            // Update from an object with policy, use the old policy
            service.Spec.IPFamilyPolicy = oldService.Spec.IPFamilyPolicy
        } else if service.Spec.ClusterIP == api.ClusterIPNone && len(service.Spec.Selector) == 0 {
            // Special-case: headless + selectorless defaults to dual.
            requireDualStack := api.IPFamilyPolicyRequireDualStack
            service.Spec.IPFamilyPolicy = &requireDualStack
        } else {
            // create or update from an object without policy (e.g.
            // ExternalName) to one that needs policy
            singleStack := api.IPFamilyPolicySingleStack
            service.Spec.IPFamilyPolicy = &singleStack
        }
    }
    // Henceforth we can assume ipFamilyPolicy is set.

    // Do some loose pre-validation of the input.  This makes it easier in the
    // rest of allocation code to not have to consider corner cases.
    // TODO(thockin): when we tighten validation (e.g. to require IPs) we will
    // need a "strict" and a "loose" form of this.
    if el := validation.ValidateServiceClusterIPsRelatedFields(service); len(el) != 0 {
        return errors.NewInvalid(api.Kind("Service"), service.Name, el)
    }

    //TODO(thockin): Move this logic to validation?
    el := make(field.ErrorList, 0)

    // Update-only prep work.
    if oldService != nil {
        if getIPFamilyPolicy(service) == api.IPFamilyPolicySingleStack {
            // As long as ClusterIPs and IPFamilies have not changed, setting
            // the policy to single-stack is clear intent.
            // ClusterIPs[0] is immutable, so it is safe to keep.
            if sameClusterIPs(oldService, service) && len(service.Spec.ClusterIPs) > 1 {
                service.Spec.ClusterIPs = service.Spec.ClusterIPs[0:1]
            }
            if sameIPFamilies(oldService, service) && len(service.Spec.IPFamilies) > 1 {
                service.Spec.IPFamilies = service.Spec.IPFamilies[0:1]
            }
        } else {
            // If the policy is anything but single-stack AND they reduced these
            // fields, it's an error.  They need to specify policy.
            if reducedClusterIPs(After{service}, Before{oldService}) {
                el = append(el, field.Invalid(field.NewPath("spec", "ipFamilyPolicy"), service.Spec.IPFamilyPolicy,
                    "must be 'SingleStack' to release the secondary cluster IP"))
            }
            if reducedIPFamilies(After{service}, Before{oldService}) {
                el = append(el, field.Invalid(field.NewPath("spec", "ipFamilyPolicy"), service.Spec.IPFamilyPolicy,
                    "must be 'SingleStack' to release the secondary IP family"))
            }
        }
    }

    // Make sure ipFamilyPolicy makes sense for the provided ipFamilies and
    // clusterIPs.  Further checks happen below - after the special cases.
    if getIPFamilyPolicy(service) == api.IPFamilyPolicySingleStack {
        if len(service.Spec.ClusterIPs) == 2 {
            el = append(el, field.Invalid(field.NewPath("spec", "ipFamilyPolicy"), service.Spec.IPFamilyPolicy,
                "must be 'RequireDualStack' or 'PreferDualStack' when multiple cluster IPs are specified"))
        }
        if len(service.Spec.IPFamilies) == 2 {
            el = append(el, field.Invalid(field.NewPath("spec", "ipFamilyPolicy"), service.Spec.IPFamilyPolicy,
                "must be 'RequireDualStack' or 'PreferDualStack' when multiple IP families are specified"))
        }
    }

    // Infer IPFamilies[] from ClusterIPs[].  Further checks happen below,
    // after the special cases.
    for i, ip := range service.Spec.ClusterIPs {
        if ip == api.ClusterIPNone {
            break
        }

        // We previously validated that IPs are well-formed and that if an
        // ipFamilies[] entry exists it matches the IP.
        fam := familyOf(ip)

        // If the corresponding family is not specified, add it.
        if i >= len(service.Spec.IPFamilies) {
            // Families are checked more later, but this is a better error in
            // this specific case (indicating the user-provided IP, rather
            // than than the auto-assigned family).
            if _, found := al.serviceIPAllocatorsByFamily[fam]; !found {
                el = append(el, field.Invalid(field.NewPath("spec", "clusterIPs").Index(i), service.Spec.ClusterIPs,
                    fmt.Sprintf("%s is not configured on this cluster", fam)))
            } else {
                // OK to infer.
                service.Spec.IPFamilies = append(service.Spec.IPFamilies, fam)
            }
        }
    }

    // If we have validation errors, bail out now so we don't make them worse.
    if len(el) > 0 {
        return errors.NewInvalid(api.Kind("Service"), service.Name, el)
    }

    // Special-case: headless + selectorless.  This has to happen before other
    // checks because it explicitly allows combinations of inputs that would
    // otherwise be errors.
    if service.Spec.ClusterIP == api.ClusterIPNone && len(service.Spec.Selector) == 0 {
        // If IPFamilies was not set by the user, start with the default
        // family.
        if len(service.Spec.IPFamilies) == 0 {
            service.Spec.IPFamilies = []api.IPFamily{al.defaultServiceIPFamily}
        }

        // this follows headful services. With one exception on a single stack
        // cluster the user is allowed to create headless services that has multi families
        // the validation allows it
        if len(service.Spec.IPFamilies) < 2 {
            if *(service.Spec.IPFamilyPolicy) != api.IPFamilyPolicySingleStack {
                // add the alt ipfamily
                if service.Spec.IPFamilies[0] == api.IPv4Protocol {
                    service.Spec.IPFamilies = append(service.Spec.IPFamilies, api.IPv6Protocol)
                } else {
                    service.Spec.IPFamilies = append(service.Spec.IPFamilies, api.IPv4Protocol)
                }
            }
        }

        // nothing more needed here
        return nil
    }

    //
    // Everything below this MUST happen *after* the above special cases.
    //

    // Demanding dual-stack on a non dual-stack cluster.
    if getIPFamilyPolicy(service) == api.IPFamilyPolicyRequireDualStack {
        if len(al.serviceIPAllocatorsByFamily) < 2 {
            el = append(el, field.Invalid(field.NewPath("spec", "ipFamilyPolicy"), service.Spec.IPFamilyPolicy,
                "this cluster is not configured for dual-stack services"))
        }
    }

    // If there is a family requested then it has to be configured on cluster.
    for i, ipFamily := range service.Spec.IPFamilies {
        if _, found := al.serviceIPAllocatorsByFamily[ipFamily]; !found {
            el = append(el, field.Invalid(field.NewPath("spec", "ipFamilies").Index(i), ipFamily, "not configured on this cluster"))
        }
    }

    // If we have validation errors, don't bother with the rest.
    if len(el) > 0 {
        return errors.NewInvalid(api.Kind("Service"), service.Name, el)
    }

    // nil families, gets cluster default
    if len(service.Spec.IPFamilies) == 0 {
        service.Spec.IPFamilies = []api.IPFamily{al.defaultServiceIPFamily}
    }

    // If this service is looking for dual-stack and this cluster does have two
    // families, append the missing family.
    if *(service.Spec.IPFamilyPolicy) != api.IPFamilyPolicySingleStack &&
        len(service.Spec.IPFamilies) == 1 &&
        len(al.serviceIPAllocatorsByFamily) == 2 {

        if service.Spec.IPFamilies[0] == api.IPv4Protocol {
            service.Spec.IPFamilies = append(service.Spec.IPFamilies, api.IPv6Protocol)
        } else if service.Spec.IPFamilies[0] == api.IPv6Protocol {
            service.Spec.IPFamilies = append(service.Spec.IPFamilies, api.IPv4Protocol)
        }
    }

    return nil
}
```
