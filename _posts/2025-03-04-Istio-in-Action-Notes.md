I recently read [Istio in Action](https://www.manning.com/books/istio-in-action). This post summarizes some good points.

1. Service mesh is a relatively recent term used to describe a decentralized application-networking infrastructure that allows applications to be secure, resilient, observable,
   and controllable. <mark>It describes an architecture made up of a data plane that uses application-layer proxies to manage networking traffic on behalf of an application and a control
   plane to manage proxies</mark>. This architecture lets us build important application-networking capabilities outside of the application without relying on a particular programming language or framework.

2. Istio is an open source implementation of a service mesh.
   
3. Istio's data plane is made up of service proxies, based on the Envoy proxy, that live alongside the applications. Those act as intermediaries between the application and affect the networking behavior according to the configuration sent by the control plane.

4. Istio is intended for microservices or service-oriented architecture (SOA)-style architectures, but it is not limited to those. The reality is, most organizations have a lot of investment in existing applications and platforms. They'll most likely build services architectures around their existing applications, and this is where Istio really shines. With Istio, we can implement these application-networking concerns without forcing changes in existing systems.
