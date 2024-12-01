In my last [post]([url](https://ericcodemonkey.github.io/2024/11/30/What-happens-in-Socket-Binding.html)) about the socket binding, I anaylized what happens under the hood when we binding the address to socekt. However, there's one detail missed which is what happens when we create a socket in Java. Today, let's delve into it.

As the same as last post, we use the below program(just for demo pupose):

```java

import java.io.IOException;
import java.io.InputStream;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.net.Socket;

public class Server {
    public static void main(String... args) throws IOException{
        ServerSocket server = new ServerSocket();
        server.bind(new InetSocketAddress("0.0.0.0", 8888));
        while(true){
            Socket client = server.accept();
            InputStream input = client.getInputStream();
            int receivedMsg;
            while((receivedMsg = input.read()) != -1){
                System.out.print((char)receivedMsg);
            }
            input.close();
            client.close();
        }
    }
    
}

```

Let's get into the `new ServerSocket()` construct:

```java

 /**
     * Creates an unbound server socket.
     *
     * @throws    IOException IO error when opening the socket.
     * @revised 1.4
     */
    public ServerSocket() throws IOException {
        setImpl();
    }

```

And then get into `setImpl()` method of `ServerSockt`:

```java

private void setImpl() {
        SocketImplFactory factory = ServerSocket.factory;
        if (factory != null) {
            impl = factory.createSocketImpl();
        } else {
            impl = SocketImpl.createPlatformSocketImpl(true);
        }
    }
```

Since we haven't set the `SocketImplFactory`, we will run into `SocketImpl.createPlatformSocketImpl(true)`:

```java

/**
     * Creates an instance of platform's SocketImpl
     */
    @SuppressWarnings("unchecked")
    static <S extends SocketImpl & PlatformSocketImpl> S createPlatformSocketImpl(boolean server) {
        if (USE_PLAINSOCKETIMPL) {
            return (S) new PlainSocketImpl(server);
        } else {
            return (S) new NioSocketImpl(server);
        }
    }

```

Here, we will check `USE_PLAINSOCKETIMPL`,  which is a static field of SocketImpl, it is determined by a Java property-`jdk.net.usePlainSocketImpl`:

```java

public abstract class SocketImpl implements SocketOptions {
    private static final boolean USE_PLAINSOCKETIMPL = usePlainSocketImpl();

    private static boolean usePlainSocketImpl() {
        PrivilegedAction<String> pa = () -> NetProperties.get("jdk.net.usePlainSocketImpl");
        @SuppressWarnings("removal")
        String s = AccessController.doPrivileged(pa);
        return (s != null) && !s.equalsIgnoreCase("false");
    }
```

The property `jdk.net.usePlainSocketImpl` is stored in `<JAVA_HOME>/conf/net.properties`, if not set, then it is false.

So in our case, `createPlatformSocketImpl` will return `new NioSocketImpl(server)`.

Until now, what we got is a Java object `ServerSocket`, with a `SocketImpl` field which is a `NioSocketImpl` object.

Actually, we haven't touch the "real" Socket yet. Because, so far we haven't any native method called which will create the "real" socket.

Don't worry. Keep going.

Let's get into `ServerSocket.bind()` method:

```java

/**
     *
     * Binds the {@code ServerSocket} to a specific address
     * (IP address and port number).
     * <p>
     * If the address is {@code null}, then the system will pick up
     * an ephemeral port and a valid local address to bind the socket.
     *
     * @param   endpoint        The IP address and port number to bind to.
     * @throws  IOException if the bind operation fails, or if the socket
     *                     is already bound.
     * @throws  SecurityException       if a {@code SecurityManager} is present and
     * its {@code checkListen} method doesn't allow the operation.
     * @throws  IllegalArgumentException if endpoint is a
     *          SocketAddress subclass not supported by this socket
     * @since 1.4
     */
    public void bind(SocketAddress endpoint) throws IOException {
        bind(endpoint, 50);
    }
```

```java

/**
     *
     * Binds the {@code ServerSocket} to a specific address
     * (IP address and port number).
     * <p>
     * If the address is {@code null}, then the system will pick up
     * an ephemeral port and a valid local address to bind the socket.
     * <P>
     * The {@code backlog} argument is the requested maximum number of
     * pending connections on the socket. Its exact semantics are implementation
     * specific. In particular, an implementation may impose a maximum length
     * or may choose to ignore the parameter altogether. The value provided
     * should be greater than {@code 0}. If it is less than or equal to
     * {@code 0}, then an implementation specific default will be used.
     * @param   endpoint        The IP address and port number to bind to.
     * @param   backlog         requested maximum length of the queue of
     *                          incoming connections.
     * @throws  IOException if the bind operation fails, or if the socket
     *                     is already bound.
     * @throws  SecurityException       if a {@code SecurityManager} is present and
     * its {@code checkListen} method doesn't allow the operation.
     * @throws  IllegalArgumentException if endpoint is a
     *          SocketAddress subclass not supported by this socket
     * @since 1.4
     */
    public void bind(SocketAddress endpoint, int backlog) throws IOException {
        if (isClosed())
            throw new SocketException("Socket is closed");
        if (isBound())
            throw new SocketException("Already bound");
        if (endpoint == null)
            endpoint = new InetSocketAddress(0);
        if (!(endpoint instanceof InetSocketAddress epoint))
            throw new IllegalArgumentException("Unsupported address type");
        if (epoint.isUnresolved())
            throw new SocketException("Unresolved address");
        if (backlog < 1)
          backlog = 50;
        try {
            @SuppressWarnings("removal")
            SecurityManager security = System.getSecurityManager();
            if (security != null)
                security.checkListen(epoint.getPort());
            getImpl().bind(epoint.getAddress(), epoint.getPort());
            getImpl().listen(backlog);
            bound = true;
        } catch(SecurityException e) {
            bound = false;
            throw e;
        } catch(IOException e) {
            bound = false;
            throw e;
        }
    }
```
In above code, we focus on the below:

`getImpl().bind(epoint.getAddress(), epoint.getPort());`

Let's get into `getImpl()` method:

```java

 /**
     * Get the {@code SocketImpl} attached to this socket, creating
     * it if necessary.
     *
     * @return  the {@code SocketImpl} attached to that ServerSocket.
     * @throws SocketException if creation fails.
     * @since 1.4
     */
    SocketImpl getImpl() throws SocketException {
        if (!created)
            createImpl();
        return impl;
    }
```

As we know from previous step, there is no "real" socket created, so we will get into `createImpl()` method:

```java

 /**
     * Creates the socket implementation.
     *
     * @throws SocketException if creation fails
     * @since 1.4
     */
    void createImpl() throws SocketException {
        if (impl == null)
            setImpl();
        try {
            impl.create(true);
            created = true;
        } catch (IOException e) {
            throw new SocketException(e.getMessage());
        }
    }
```

Since our `impl` is a `NioSocketImpl`, it is not null, so `NioSocketImpl's create` method will be called:

```java

 /**
     * Creates the socket.
     * @param stream {@code true} for a streams socket
     */
    @Override
    protected void create(boolean stream) throws IOException {
        synchronized (stateLock) {
            if (state != ST_NEW)
                throw new IOException("Already created");
            if (!stream)
                ResourceManager.beforeUdpCreate();
            FileDescriptor fd;
            try {
                if (server) {
                    assert stream;
                    fd = Net.serverSocket(true);
                } else {
                    fd = Net.socket(stream);
                }
            } catch (IOException ioe) {
                if (!stream)
                    ResourceManager.afterUdpClose();
                throw ioe;
            }
            Runnable closer = closerFor(fd, stream);
            this.fd = fd;
            this.stream = stream;
            this.cleaner = CleanerFactory.cleaner().register(this, closer);
            this.state = ST_UNCONNECTED;
        }
    }
```

See, we are closing to the "real" socket!

Let's continue. Since we are create a "Server" socket, we will get into `Net.serverSocket(true)`:

```java

static FileDescriptor serverSocket(boolean stream) {
    return serverSocket(UNSPEC, stream);
}

static FileDescriptor serverSocket(ProtocolFamily family, boolean stream) {
    boolean preferIPv6 = isIPv6Available() &&
    (family != StandardProtocolFamily.INET);
    return IOUtil.newFD(socket0(preferIPv6, stream, true, fastLoopback));
}
```
Firstly, we will check whether IPv6 available and its ProtocolFamily(IPv4 or IPv6, or Unix Domain).

Check whether IPv6 available:

```java

/**
     * Tells whether dual-IPv4/IPv6 sockets should be used.
     */
    static boolean isIPv6Available() {
        if (!checkedIPv6) {
            isIPv6Available = isIPv6Available0();
            checkedIPv6 = true;
        }
        return isIPv6Available;
    }
```
It will call a native method:

`private static native boolean isIPv6Available0();`

We have discussed this native method in [last post](https://ericcodemonkey.github.io/2024/11/30/What-happens-in-Socket-Binding.html). Acually, the implementation of this native method is very straightforward, it will try to create a IPv6 socket, if it fails, that means IPv6 is unsupported, if it sucesses, that means IPv6 is supported, that is it.

Since my machine is dualstack, so the IPv6 is available.

Then it will check the ProtocolFamily, in the previous step, "UNSPEC" is passed, this gives the chance for kernel to dertermine. 

So in our case, the `preferIPv6` will be `true`.

And then the native method which is responsible to create the "real" socket will be called:

`private static native int socket0(boolean preferIPv6, boolean stream, boolean reuse,
                                      boolean fastLoopback);`

Please notes here the `preferIPv6` is true, and `stream` is true as well.

And then let's get into the native method of `socket0` in [Net.c in JDK](https://github.com/openjdk/jdk/blob/master/src/java.base/unix/native/libnio/ch/Net.c): 

```c

JNIEXPORT jint JNICALL
Java_sun_nio_ch_Net_socket0(JNIEnv *env, jclass cl, jboolean preferIPv6,
                            jboolean stream, jboolean reuse, jboolean ignored)
{
    int fd;
    int type = (stream ? SOCK_STREAM : SOCK_DGRAM);
    int domain = (ipv6_available() && preferIPv6) ? AF_INET6 : AF_INET;

    fd = socket(domain, type, 0);
    if (fd < 0) {
        return handleSocketError(env, errno);
    }

    /*
     * If IPv4 is available, disable IPV6_V6ONLY to ensure dual-socket support.
     */
    if (domain == AF_INET6 && ipv4_available()) {
        int arg = 0;
        if (setsockopt(fd, IPPROTO_IPV6, IPV6_V6ONLY, (char*)&arg,
                       sizeof(int)) < 0) {
            JNU_ThrowByNameWithLastError(env,
                                         JNU_JAVANETPKG "SocketException",
                                         "Unable to set IPV6_V6ONLY");
            close(fd);
            return -1;
        }
    }

    if (reuse) {
        int arg = 1;
        if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (char*)&arg,
                       sizeof(arg)) < 0) {
            JNU_ThrowByNameWithLastError(env,
                                         JNU_JAVANETPKG "SocketException",
                                         "Unable to set SO_REUSEADDR");
            close(fd);
            return -1;
        }
    }

#if defined(__linux__)
    if (type == SOCK_DGRAM) {
        int arg = 0;
        int level = (domain == AF_INET6) ? IPPROTO_IPV6 : IPPROTO_IP;
        if ((setsockopt(fd, level, IP_MULTICAST_ALL, (char*)&arg, sizeof(arg)) < 0) &&
            (errno != ENOPROTOOPT)) {
            JNU_ThrowByNameWithLastError(env,
                                         JNU_JAVANETPKG "SocketException",
                                         "Unable to set IP_MULTICAST_ALL");
            close(fd);
            return -1;
        }
    }

    if (domain == AF_INET6 && type == SOCK_DGRAM) {
        /* By default, Linux uses the route default */
        int arg = 1;
        if (setsockopt(fd, IPPROTO_IPV6, IPV6_MULTICAST_HOPS, &arg,
                       sizeof(arg)) < 0) {
            JNU_ThrowByNameWithLastError(env,
                                         JNU_JAVANETPKG "SocketException",
                                         "Unable to set IPV6_MULTICAST_HOPS");
            close(fd);
            return -1;
        }

        /* Disable IPV6_MULTICAST_ALL if option supported */
        arg = 0;
        if ((setsockopt(fd, IPPROTO_IPV6, IPV6_MULTICAST_ALL, (char*)&arg, sizeof(arg)) < 0) &&
            (errno != ENOPROTOOPT)) {
            JNU_ThrowByNameWithLastError(env,
                                     JNU_JAVANETPKG "SocketException",
                                     "Unable to set IPV6_MULTICAST_ALL");
            close(fd);
            return -1;
        }
    }
#endif

#ifdef __APPLE__
    /**
     * Attempt to set SO_SNDBUF to a minimum size to allow sending large datagrams
     * (net.inet.udp.maxdgram defaults to 9216).
     */
    if (type == SOCK_DGRAM) {
        int size;
        socklen_t arglen = sizeof(size);
        if (getsockopt(fd, SOL_SOCKET, SO_SNDBUF, &size, &arglen) == 0) {
            int minSize = (domain == AF_INET6) ? 65527  : 65507;
            if (size < minSize) {
                setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &minSize, sizeof(minSize));
            }
        }
    }
#endif

    return fd;
}
```

Let's analyze the implementation in this method:

```c

 int type = (stream ? SOCK_STREAM : SOCK_DGRAM);
 int domain = (ipv6_available() && preferIPv6) ? AF_INET6 : AF_INET;

```
From the above code, we can see that if IPv6 is avaiable and the `preferIPv6` is true, then we will use `AF_INET6` as domain to create the socket.

`fd = socket(domain, type, 0);`

Here, the system call `socket` will be invoked, and the "real" socket get createdi, fianlly!

And there's a very important step:

```java
 /*
     * If IPv4 is available, disable IPV6_V6ONLY to ensure dual-socket support.
     */
    if (domain == AF_INET6 && ipv4_available()) {
        int arg = 0;
        if (setsockopt(fd, IPPROTO_IPV6, IPV6_V6ONLY, (char*)&arg,
                       sizeof(int)) < 0) {
            JNU_ThrowByNameWithLastError(env,
                                         JNU_JAVANETPKG "SocketException",
                                         "Unable to set IPV6_V6ONLY");
            close(fd);
            return -1;
        }
    }
```

This step will set the `IPV_V6ONLY` to false, so that this AF_INET6 socket can handle both IPv4 and IPv6 traffic simultuously, what we call is "Dual Stack Support"!.

>[!NOTE]
> IPV6_V6ONLY (since Linux 2.4.21 and 2.6)
> If this flag is set to true (nonzero), then the socket is
> restricted to sending and receiving IPv6 packets only.  In
> this case, an IPv4 and an IPv6 application can bind to a
> single port at the same time.
> If this flag is set to false (zero), then the socket can
> be used to send and receive packets to and from an IPv6
> address or an IPv4-mapped IPv6 address.
> The argument is a pointer to a boolean value in an integer.
> The default value for this flag is defined by the contents
> of the file /proc/sys/net/ipv6/bindv6only.  The default
> value for that file is 0 (false).

I think the post makes the complete process of how to create a socket in Java very clearly.

Hope you like it.





