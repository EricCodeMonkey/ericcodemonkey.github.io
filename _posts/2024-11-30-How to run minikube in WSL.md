Recently, I need to run minikube in WSL. I followed below 2 guides:

[Kubernetes Setup with Minikube on WSL2](https://gaganmanku96.medium.com/kubernetes-setup-with-minikube-on-wsl2-2023-a58aea81e6a3 "Kubernetes Setup with Minikube on WSL2")

[Getting Minikube on WSL2 Ubuntu working](https://gist.github.com/wholroyd/748e09ca0b78897750791172b2abb051)



Most steps work; however, there are many pitfalls you need to fix. Here are some tips I collected when I tried to run Minikube in WSL.



## 1. If you run into the below error when you prepare the environment in WSL, for example, run "sudo apt install" for some required components:

   `Failed to take /etc/passwd lock: Invalid argument`


    
   Try to run the below command:

      
       sudo mv /var/lib/dpkg/info /var/lib/dpkg/info_silent
       sudo mkdir /var/lib/dpkg/info
       sudo apt-get update
       sudo apt-get -f install
       sudo mv /var/lib/dpkg/info/* /var/lib/dpkg/info_silent
       sudo rm -rf /var/lib/dpkg/info
       sudo mv /var/lib/dpkg/info_silent /var/lib/dpkg/info
       sudo apt-get update
       sudo apt-get upgrade
    

   I don't know why it works, it just works. Big thanks to this tip from Stackoverflow, who saved my day:)


## 2. Make sure you are running WSL in version 2.
    
   Run: 
       
       wsl  -l  -v
      
   
   Check the version in the output.

   If the version is not 2, run the below command in Windows Powershell:

             wsl  --set-version  <your wsl image name in the output of "wsl -l -v">  2

   If the above command failed due to the below error:

            Error code: Wsl/Service/CreateVm/HCS/HCS_E_SERVICE_NOT_AVAILABLE

  Run below commands:
 
         dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

         dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
 
  
And then restart your machine.

After the above 2 steps, you can continue to follow the 2 referenced guides to install minikube in WSL.


## 3. If you run into a "failed to download images" likes error when you run "minikube start", you need to check your wsl's DNS server configuration in /etc/resolv.conf. You may need to add a valid DNS server to this file.

After the DNS server configuration is corrected, you should start the minikube successfully.


## 4. After the minikube is ready, if you encounter "Error: ImagePullBackoff" when deploying the pod via helm install, you may need to check the CoreDNS pods and config map.

Firstly, check the CoreDNS pods:

      kubectl get  pods -n kube-system -l k8s-app=kube-dns

The output should show Running status for CoreDNS pods:

NAME                        READY   STATUS    RESTARTS   AGE
coredns-78fcd69978-xxxxx    1/1     Running   0          10m

If CoreDNS is not running, troubleshoot by:

     a. Restarting the CoreDNS deployment:
 
       kubectl rollout restart deployment coredns -n kube-system

     b. Checking CoreDNS logs

And if the CoreDNS pod is OK, then you can check  the CoreDNS ConfigMap;

    kubectl edit configmap coredns -n kube-system
 
The output should be like this:
```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        log
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        hosts {
           192.168.11.12  host.minikube.internal
           fallthrough
        }
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2024-11-20T17:37:38Z"
  name: coredns
  namespace: kube-system
  resourceVersion: "6439"
  uid: c5fd087f-e789-4503-87a0-a4ced040bd54
```

Correct the highlighted IP in the hosts section with a valid DNS server and restart the CoreDNS pod.

With all the above pitfalls fixed, you will be ready to go with minikube:)

Hope you enjoy it.
