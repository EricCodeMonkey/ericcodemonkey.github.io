I recently read [Istio in Action](https://www.manning.com/books/istio-in-action). This post summarizes some good points.

1. Service mesh is a relatively recent term used to describe a decentralized application-networking infrastructure that allows applications to be secure, resilient, observable,
   and controllable. <mark>It describes an architecture made up of a data plane that uses application-layer proxies to manage networking traffic on behalf of an application and a control
   plane to manage proxies</mark>. This architecture lets us build important application-networking capabilities outside of the application without relying on a particular programming language or framework.

2. Istio is an open source implementation of a service mesh.
   
3. Istio's data plane is made up of service proxies, based on the Envoy proxy, that live alongside the applications. Those act as intermediaries between the application and affect the networking behavior according to the configuration sent by the control plane.

4. Istio is intended for microservices or service-oriented architecture (SOA)-style architectures, but it is not limited to those. The reality is, most organizations have a lot of investment in existing applications and platforms. They'll most likely build services architectures around their existing applications, and this is where Istio really shines. With Istio, we can implement these application-networking concerns without forcing changes in existing systems.

5. .....Some patterns have evolved to help mitigate these types of scenarios and help make applications more resilient to unplanned, unexpected failures:
   
   Client-side load balancing - Give the client the list of possible endpoints, and let it decide which one to call.
   
   Service discovery - A mechanism for finding the periodically updated list of healthy endpoints for a particular logical services.
   
   Circuit Breaking - Shed load for a period of time to a service that appears to be misbehaving.
   
   Bulkheading - Limit client resource usage with explicit thresholds(connections, threads, sessions, and so on) when making calls to a service.
   
   Timeouts - Enforce time limitations on requests, sockets, liveness, and so on when making calls to a service.
   
   Retries - Retry a failed request.
   
   Retry budgets - Apply constraints to retries: that is, limit the number of retries in a given period (for example, only retry 50% of the calls in a 10-second window).
   
   Deadlines - Give requests context about how long a response may still be useful; if outside the deadline, disregard processing the request.

   Collectively, these types of patterns can be thought of as application networking. They have a lot of overlap with similar constructs at lower layers of the networking stack, except that they operate at the layer of messages instead of packets.
