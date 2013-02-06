---
layout: post
title: "android 如何创建socket套接字的原理"
description: ""
category: "work android linux"
tags: []
sumarry: |
  本文的目的在于分析android的执行java程序的原理,由于android是基于linux系统的,因此,它执行的任何的java程序,涉及到系统的,例如socket套接字的操作,或者遍历目录等等的,这些操作,最终的都调用到linux的socket函数和fstat函数. 那具体的是如何调用到linux的api函数的呢,这就是本文要分析的.看看android源码对这个是如何的进行设置的.
---
{% include JB/setup %}

#摘要
  本文的目的在于分析android的执行java程序的原理,由于android是基于linux系统的,因此,它执行的任何的java程序,涉及到系统的,例如socket套接字的操作,或者遍历目录等等的,这些操作,最终的都调用到linux的socket函数和fstat函数. 那具体的是如何调用到linux的api函数的呢,这就是本文要分析的.看看android源码对这个是如何的进行设置的.
 

  
#实例分析
  为了更好的来分析这一个过程,我们将通过下面的代码来跟踪具体的执行过程.
  

<pre class="brush: js;">    
    public class hello{ 

    		public static DatagramSocket sendSocket ;  
    		public static DatagramPacket sendPacket ; 

    		public static  String data ;

    		public static InetAddress destIPAddress ; 		
    		public static int destPort ; 		
    		public static byte[] buffer = new byte[1024];



    		public static void main(String args[])
    		{
    			try  {
    				destPort = 9999;
    				System.out.println("hello world  I am ok now hahahhahahahahh 1");
    				sendSocket = new DatagramSocket(destPort);
    				data = "hello world";
    				byte out [] = data.getBytes();
    				sendPacket = new DatagramPacket(buffer, 1024);
    				sendPacket.setData(out);
    				sendPacket.setLength(out.length);
    				sendPacket.setPort(destPort);
    				destIPAddress = InetAddress.getByName("192.168.1.144");
    				sendPacket.setAddress(destIPAddress);  
    				System.out.println("hello world  I am ok now hahahhahahahahh 2");
    				sendSocket.send(sendPacket);			
    			    System.out.println("hello world  I am ok now hahahhahahahahh 3");
    		    }
    		    catch (SocketException e) {
    		    	e.printStackTrace();  
    		    }
    		    catch (IOException e)  {
    		    	e.printStackTrace();  
    		    }
    		}
    }
</pre>



上面的java程序里, 使用了android的core.jar里面的类来创建socket.下面我们将会分析这个创建过程是如何调用到linux系统的socket函数的.

##编译java程序
在ubuntu下,执行下面的Makefile,就可以得到android下可以执行的jar包.本jar包将直接通过dalvik虚拟机执行.执行结束后,将会得到hello.jar包.这个包里面会包含有dalvik的执行文件classes.dex.

<pre class="brush: js;">
    android_dir_dx = $(ANDROID_SRC_DIR)/out/host/linux-x86/bin/dx
    all:
    	javac hello.java -cp ./core.jar
    	$(android_dir_dx) --dex --output=hello.jar hello.class
    clean:
    	@rm *.jar *.class    
</pre>

##执行jar
将上面得到的jar包放到android的某个文件夹下面, 然后执行下面的命令,执行命令的路径必须与本jar路径一致.
<pre class="brush: js;">
    dalvikvm -cp hello.jar hello
</pre>

##执行结果
若是执行者没有被授予权限的话,执行socket套接字的操作必将会被拒绝,而抛出perssion denied的错误.如下:
<pre class="brush: js;">
    java.net.SocketException: socket failed: EACCES (Permission denied)
            at libcore.io.IoBridge.socket(IoBridge.java:573)
            at java.net.PlainDatagramSocketImpl.create(PlainDatagramSocketImpl.java:91)
            at java.net.DatagramSocket.createSocket(DatagramSocket.java:131)
            at java.net.DatagramSocket.&lt;init&gt;(DatagramSocket.java:78)
            at hello.main(hello.java:28)
            at dalvik.system.NativeStart.main(Native Method)
    Caused by: libcore.io.ErrnoException: socket failed: EACCES (Permission denied)
            at libcore.io.Posix.socket(Native Method)
            at libcore.io.BlockGuardOs.socket(BlockGuardOs.java:181)
            at libcore.io.IoBridge.socket(IoBridge.java:558)
            ... 5 more
</pre>


#跟踪过程
在这里, 我们看到了exception stack, 因此借助这个stack, 我们可以跟踪到问题的源头.

当然, 也可以选择另外的方式来查看到底是哪里的exception的问题, 这需要来跟踪一下, 这个抛出exception的地方.

从这里, 我们可以看到的是最终, 问题的根源, 是在被调用的jni函数, 也就是
<pre class="brush: js;">
    @Override public FileDescriptor socket(int domain, int type, int protocol) throws ErrnoException {
        return tagSocket(os.socket(domain, type, protocol));
    }
</pre>
仔细, 看这一段的代码, 其中, os 是一个interface, 但是可以肯定的是哪里对这个进行了
好了, 下面, 我们来看看, 这里的jni到底是如何设置的, 这一点挺重要的, 只要分析了, 就应该会很明了.

在 IoBridge.java文件里, 出错的地方代码如下:
<pre class="brush: js;">
    public static FileDescriptor socket(boolean stream) throws SocketException {
        FileDescriptor fd;
        try {
            fd = Libcore.os.socket(AF_INET6, stream ? SOCK_STREAM : SOCK_DGRAM, 0);
</pre>
其中, Libcore.os是静态的, 也就是说不是跟类实例绑定在一起的.
<pre class="brush: js;">
    package libcore.io;
    public final class Libcore {
        private Libcore() { }
        public static Os os = new BlockGuardOs(new Posix());
    }
</pre>
那问题就是出在这个静态成员os的某个方法的执行上面了.那继续来看看, 这个成员到底是什么.

其中, Os这个类, 看了源码, 知道这只是一个接口而已.
那, 接口里面的方法由谁来实现呢, 这就需要来看看具体的定义了.
<pre class="brush: js;">
    public interface Os {
        public FileDescriptor accept(FileDescriptor fd, InetSocketAddress peerAddress) throws ErrnoException;
        public boolean access(String path, int mode) throws ErrnoException;
        public void bind(FileDescriptor fd, InetAddress address, int port) throws ErrnoException;
        public void chmod(String path, int mode) throws ErrnoException;
        public void close(FileDescriptor fd) throws ErrnoException;
</pre>

来看看BlockGuardOs的构造函数
<pre class="brush: js;">
    public class BlockGuardOs extends ForwardingOs {
        public BlockGuardOs(Os os) {
            super(os);
        }
</pre>

这个构造函数, 调用了ForwardingOs的构造函数
<pre class="brush: js;">
    public class ForwardingOs implements Os {
        protected final Os os;
        public ForwardingOs(Os os) {
            this.os = os;
        }
</pre>
这里就可以看到真正的定义了, 也就是说是整体的赋值, 这就需要看传进来的参数os是如何定义的了.
new Posix(), 这个是一个实现了Os接口的类.

好, 来看看这个类的定义
<pre class="brush: js;">
    public final class Posix implements Os {
        Posix() { }
        public native FileDescriptor accept(FileDescriptor fd, InetSocketAddress peerAddress) throws ErrnoException;
        public native boolean access(String path, int mode) throws ErrnoException;
        public native void bind(FileDescriptor fd, InetAddress address, int port) throws ErrnoException;
        public native void chmod(String path, int mode) throws ErrnoException;
        public native void close(FileDescriptor fd) throws ErrnoException;
        public native void connect(FileDescriptor fd, InetAddress address, int port) throws ErrnoException;
        public native FileDescriptor dup(FileDescriptor oldFd) throws ErrnoException;
        public native FileDescriptor dup2(FileDescriptor oldFd, int newFd) throws ErrnoException;
        public native String[] environ();
        public native FileDescriptor socket(int domain, int type, int protocol) throws ErrnoException; 
</pre>  
很明显, 这个类对于接口的实现, 基本上都是通过调用jni来实现的.

那, 我们通过查看android源码, 来看看具体的jni到底是调用了那个函数吧.


上面的jni , 都是调用了用c语言写好的c库里的函数, 而, 实现的地方在android源码里的
libcore_io_posix.cpp文件里面.
具体的请看  

其中, 
<pre class="brush: js;">
    static jobject Posix_socket(JNIEnv* env, jobject, jint domain, jint type, jint protocol) {
        fprintf(stderr, "libcore_io_posix.cpp  Posix_socket  start \n");
        int fd = throwIfMinusOne(env, "socket", TEMP_FAILURE_RETRY(socket(domain, type, protocol)));
        return fd != -1 ? jniCreateFileDescriptor(env, fd) : NULL;
    }
</pre>
就是对上面的
 public native FileDescriptor socket(int domain, int type, int protocol) throws ErrnoException;
的实现
<pre class="brush: js;">
    static JNINativeMethod gMethods[] = {
        NATIVE_METHOD(Posix, accept, "(Ljava/io/FileDescriptor;Ljava/net/InetSocketAddress;)Ljava/io/FileDescriptor;"),
        NATIVE_METHOD(Posix, access, "(Ljava/lang/String;I)Z"),
        NATIVE_METHOD(Posix, bind, "(Ljava/io/FileDescriptor;Ljava/net/InetAddress;I)V"),
        NATIVE_METHOD(Posix, chmod, "(Ljava/lang/String;I)V"),
        NATIVE_METHOD(Posix, close, "(Ljava/io/FileDescriptor;)V"),
        NATIVE_METHOD(Posix, connect, "(Ljava/io/FileDescriptor;Ljava/net/InetAddress;I)V"),
        NATIVE_METHOD(Posix, dup, "(Ljava/io/FileDescriptor;)Ljava/io/FileDescriptor;"),
        NATIVE_METHOD(Posix, dup2, "(Ljava/io/FileDescriptor;I)Ljava/io/FileDescriptor;"),
        NATIVE_METHOD(Posix, environ, "()[Ljava/lang/String;"),
        NATIVE_METHOD(Posix, fcntlVoid, "(Ljava/io/FileDescriptor;I)I"),
        NATIVE_METHOD(Posix, fcntlLong, "(Ljava/io/FileDescriptor;IJ)I"),
        NATIVE_METHOD(Posix, fcntlFlock, "(Ljava/io/FileDescriptor;ILlibcore/io/StructFlock;)I"),
        NATIVE_METHOD(Posix, fdatasync, "(Ljava/io/FileDescriptor;)V"),
        NATIVE_METHOD(Posix, fstat, "(Ljava/io/FileDescriptor;)Llibcore/io/StructStat;"),
        NATIVE_METHOD(Posix, fstatfs, "(Ljava/io/FileDescriptor;)Llibcore/io/StructStatFs;"),
        NATIVE_METHOD(Posix, fsync, "(Ljava/io/FileDescriptor;)V"),
        NATIVE_METHOD(Posix, ftruncate, "(Ljava/io/FileDescriptor;J)V"),
        NATIVE_METHOD(Posix, gai_strerror, "(I)Ljava/lang/String;"),
        NATIVE_METHOD(Posix, getaddrinfo, "(Ljava/lang/String;Llibcore/io/StructAddrinfo;)[Ljava/net/InetAddress;"),
        NATIVE_METHOD(Posix, getegid, "()I"),
        NATIVE_METHOD(Posix, geteuid, "()I"),
        NATIVE_METHOD(Posix, getgid, "()I"),
</pre>
而, 对于jni的注册,  实现方法如下.

这个宏是用来定义

<pre class="brush: js;">
    #define NATIVE_METHOD(className, functionName, signature) \
        { #functionName, signature, reinterpret_cast&lt;void*&gt;(className ## _ ## functionName) }
</pre>
<pre class="brush: js;">
而注册的地方在这里

    // DalvikVM calls this on startup, so we can statically register all our native methods.
    int registerCoreLibrariesJni(JNIEnv* env) {
        ScopedLocalFrame localFrame(env);
        JniConstants::init(env);
        bool result =
                register_java_io_Console(env) != -1 &amp; &amp; 
                register_java_io_File(env) != -1 &amp; &amp; 
                register_java_io_ObjectStreamClass(env) != -1 &amp; &amp; 
                register_java_lang_Character(env) != -1 &amp; &amp; 
                             ......
                             ......
                register_libcore_icu_TimeZones(env) != -1 &amp; &amp; 
                register_libcore_io_AsynchronousCloseMonitor(env) != -1 &amp; &amp; 
                register_libcore_io_Memory(env) != -1 &amp; &amp; 
                register_libcore_io_OsConstants(env) != -1 &amp; &amp; 
                register_libcore_io_Posix(env) != -1 &amp; &amp; 

    int register_libcore_io_Posix(JNIEnv* env) {
        return jniRegisterNativeMethods(env, "libcore/io/Posix", gMethods, NELEM(gMethods));
    }
</pre>

就这样子, android里面的关于socket的操作, 其实最终本质上还是要通过调用linux的socket函数, 
通过jni的方式来调用socket函数.


