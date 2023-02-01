## Container Memory Challenges

### Background
Recently we saw  microservice being killed and container restarted in production. It was a service used by customers to setup their autorecharges, and didn't have much load. The behaviour was strange, as it had not be observed earlier. Most of our microservices are springboot applications which are containerized and deployed into AWS ECS. Their memory footprint is small, and we didn't have much trouble running them over the past few years, until we started to observe this strange behaviour for this particular microservice.

The production team, had been monitoring the service and suggested there is a memory leak. The application's memory size keeps increasing till the container runs out of memory and it gets restarted.  The support team increased the heap size to see if it helps. It did for sometime, but after a week, the same problem resurfaced.


There were somethings to note about this application. It was running on Amazon's Corretto Java 8. 


Base docker file being used is

```docker

FROM amazoncorretto:8

# Install JDK
RUN yum install -y aws-cli \
  && yum install -y which \
  && rm -f /etc/localtime && ln -s /usr/share/zoneinfo/Australia/Sydney /etc/localtime

# Install gcredstash into the container so we can use it to retrieve certs and other secrets.
RUN yum install -y  https://github.com/winebarrel/gcredstash/releases/download/v0.3.5/gcredstash-0.3.5-1.el6.x86_64.rpm
# Add The following command to your startup.sh script to configure credstash for the account the container is starting in.
# AWS_REGION=ap-southeast-2 gcredstash setup

# Set one or more individual labels
LABEL moa.ms.base.version="0.0.1" \
      moa.ms.base.release-date="2020.02.03" \
      moa.ms.base.version.is-production="myoptus"

```

This would install the latest image of Amazon's corretto Java 8, but didn't specify the exact version. To find out the full version we quickly hopped on to the container and checked

```shell

$ java -version

openjdk version "1.8.0_302"
OpenJDK Runtime Environment Corretto-8.302.08.1 (build 1.8.0_302-b08)
OpenJDK 64-Bit Server VM Corretto-8.302.08.1 (build 25.302-b08, mixed mode)


```
As we know, when java application runs it 'sees' the whole memory and all CPU cores available on the system and aligns its resources to it. When run without the -Xms and -Xmx flags, it would set be default as 25% of the system memory. When running under a container, this can be a problem. 

Lets check what the default memory allocation is for different JDK versions.
For Java 11 and above
The Xmx value is 25% of the available memory with a maximum of 25 GB. However, where there is 2 GB or less of physical memory, the value set is 50% of available memory with a minimum value of 16 MB and a maximum value of 512 MB.

For Java 8
The Xmx value is half the available memory with a minimum of 16 MB and a maximum of 512 MB.


The good news, is this version supports containers, and when started without initial and max heap sizes, it would honour container's memory restrictions and accordingly set max heap size. The settings being used are

![Alt text](memory-settings.png?raw=true "Container Memory Limits")


The container memory has been fixed at 10% of Xmx value (1792m), which it seems can be a problem as its not taking into account memory used by JVM for its internal functioning. The JVM's off-heap is created at the JVM startup and stores per-class structures such as runtime constant pool, field and method data, and the code for methods and constructors, as well as interned Strings. 


Java Process Memory = Heap memory + Metaspace + CodeCache + (ThreadStackSize * Number of Threads) + DirectByteBuffers + Jvm-native

Unfortunately, the only information JVM provides on non-heap memory is its overall size. No detailed information on non-heap memory content is available. Though we can estimate the values of components which make up the non heap. 


Paramters affecting the memory available to the JVM include:

    -Xms: Sets the minimum and initial size of the heap.
    -Xmx: Sets the maximum size of the heap.
    -XX:PermSize: Sets the initial size of the Permanent Generation (perm) memory area. This option was available prior to JDK 8 but is no longer supported.
    -XX:MaxPermSize: Sets the maximum size of the perm memory area. This option was available prior to JDK 8 but is no longer supported.
    -XX:MetaspaceSize: Sets the initial size of Metaspace. This option is available starting in JDK 8.
    -XX:MaxMetaspaceSize: Sets the maximum size of Metaspace. This option is available starting in JDK 8.


### Metspace
What are the default Metaspace and MaxMetaspace size values


```sh
➜  ~ java -XX:+PrintFlagsFinal -version | grep MetaspaceSize
    uintx InitialBootClassLoaderMetaspaceSize       = 4194304                             {product}
    uintx MaxMetaspaceSize                          = 18446744073709547520                {product}
    uintx MetaspaceSize                             = 21807104                            {pd product}
java version "1.8.0_341"
```

JVM parameter values are shown in bytes, the default MetaspaceSize is about 20MB, however MaxMetaspaceSize is 18 Exabytes!! 18446744073709547520 = 0xFFFFFFFFFFFFF000 basically the maximum possible 64-bit integer. Its a parameter to indicate JVM can use as much as possible, if required.


### DirectByteBuffer
DirectByteBuffer is an intersting one. The amount of DirectByteBuffer used, if not specified by the option XX:MaxDirectMemorySize would use the Runtime.getRuntime.maxMemory(), which equates to Heap size (Xmx). 
It was introduced with java.nio (JDK 1.4). This package uses ByteBuffers to read and write data. java.nio.ByteBuffer is an abstract class; its concrete subclasses are HeapByteBuffer, which simply wraps a byte[] array and the DirectByteBuffer that allocates off-heap memory. They are created by ByteBuffer.allocate() and ByteBuffer.allocateDirect() calls, respectively.
Each buffer kind has its pros and cons, but what’s important is that the OS can read and write bytes only from or to native memory. Thus, if your code (or some I/O library, such as Netty) uses a HeapByteBuffer for I/O, its contents are always copied to or from a temporary DirectByteBuffer that's created by the JDK under the hood.

Furthermore, the JDK caches one or more DirectByteBuffers per thread, and by default, there is no limit on the number or size of these buffers. As a result, if a Java app creates many threads that perform I/O using HeapByteBuffers, and/or these buffers are big, the JVM process may end up using a lot of additional native memory that looks like a leak. Native memory regions used by the given thread are released only when the thread terminates, and subsequently, the GC reaches that thread’s DirectByteBuffer instance(s).  A system property to control the memory used by buffer cache was introduced in JDK 9. 

<b>From the release notes of JDK 9.</b>

<i>
The system property jdk.nio.maxCachedBufferSize has been introduced in JDK 9 to limit the memory used by the "temporary buffer cache". The temporary buffer cache is a per-thread cache of direct memory used by the NIO implementation to support applications that do I/O with buffers backed by arrays in the Java heap. The value of the property is the maximum capacity of a direct buffer that can be cached. If the property is not set, then no limit is put on the size of buffers that are cached. Applications with certain patterns of I/O usage may benefit from using this property. In particular, an application may see a benefit to using this property if it does I/O with large multi-megabyte buffers at startup but thereafter does I/O with small buffers. Applications that do I/O using direct buffers will not see any benefit to using this system property.

https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8147468
</i>



#### Thread stack

Each JVM thread has a private native stack to store call stack information, local variables, and partial results. Therefore, the stack plays a crucial part in method invocations. This is part of the JVM specification, and consequently, every JVM implementation out there is using stacks. If we don't specify a size for the stacks, the JVM will create one with a default size. Usually, this default size depends on the operating system and computer architecture.
To change the stack size, we can use the -Xss tuning flag. For example, the -Xss1m sets the stack size equal to 1 MB. The more threads we have, more would be memory consumption.


## Conclusions

Having the option to use heap memory vs. direct memory proved very useful in identifying a “leak” (in this case, it wasn’t really a leak, just too aggressive of an allocation strategy) in a deployment environment that doesn’t permit direct access to the Box/VM/container.

While configuring the container's memory, its important to keep in mind not only the dot release version of java, but also its minor versions as some features/bug-fixes are backported to these minor versions.

The use of following in docker file is probably not the best way to know which minor version of java would be used. 
from amazoncorretto:8

Its always better to use SHA hash of image. https://rockbag.medium.com/why-you-should-pin-your-docker-images-with-sha-instead-of-tags-fd132443b8a6

For Java 8 based containers, use the following memory approximation
-Xms128m
-Xmx512m
-XX:MaxMetaspaceSize=64m
-XX:MaxDirectMemorySize=512m

The size of container memory should be 512m + 64m + 512m + 256m(buffer). Though these values don't guarantee container's won't be killed, but it does provide a close enough estimation of actual memory consumption.

Though it was possible for us to determine the reasons for large memory use, and find a way to control it, however this is not the last of memory issues we'll see.


