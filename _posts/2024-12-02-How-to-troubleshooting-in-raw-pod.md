Sometimes you need to do some troubleshooting in your Kubernetes cluster. For example, you want to check the sockets in your pod. Unfortunately, the pod you want to inspect is a "raw" pod - It doesn't contain any troubleshooting tools, like netstat, ss, tcpdump, etc.. And the worse thing is, you don't have the previlege to log on the node in which this pod is running. What can you do?

Here comes a life saver.

You can use `kubectl debug` to attach a container with those troubleshooting tools to the pod you want to inspect.

`kubectl debug pod/<your-pod-name> -n <your-namespace> -it --image=nicolaka/netshoot`

And then you can run those commands in this container to inspect the sockets.

> [!note]
> `kubectl debug` is only supported from kubectl v1.18+. So make sure you are using the right version.

If you are using older version kubectl, there's also a workaround. You can edit your pod via: kubectl edit pod <your-pod>, and then add the below container:

```yaml

  containers:
  - name:debug-container
    image:nicolaka/netshoot
    stdin:true
    tty:true
```

And then you can log on this container via: `kubectl exec -it pod <your-pod-name> -c debug-container -- sh`, and run the troubleshooting tools.

Enjoy it.


