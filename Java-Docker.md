# Java and Docker - Memory and CPU Limits

## As container deployments become increasingly common, it is important to understand how our applications coordinate with various Linux container technologies at runtime.
Depending on our JVM version, we may need to do a little extra work to ensure your java runtime is aware of the **processor and memory usage limits.**
**Until Java 8u131 and Java 9 the JVM did not recognize memory or cpu limits set by the container.  The first implementation was a experimental feature and had its flaws but in Java 10, memory limits are automatically recognized and enforced. This feature was then backported to Java-8u191.**
For CPU limits, the JVM internally and transparently sets the number of GC threads, and JIT compiler threads.
These can be explicitly set via command line options, **-XX:ParallelGCThreads** and **-XX:CICompilerCount**.
However, in cases where none of the aforementioned JVM command line options are specified, the JVM for a Java application using Java SE 8u121 and earlier, when run in a Docker container,  will use the underlying host configuration of CPUs when transparently determining the number of GC threads, JIT compiler threads and maximum Java heap to use. In short, it will not be aware of the container limits.

**As of Java SE 8u131, and in JDK 9, the JVM is Docker-aware with respect to Docker CPU limits transparently.** That means even if -XX:ParalllelGCThreads, or -XX:CICompilerCount are not specified as command line options, the JVM will apply the Docker CPU limit as the number of CPUs the JVM sees on the system. It will then adjust the number of GC threads and JIT compiler threads accordingly. If -XX:ParallelGCThreads or -XX:CICompilerCount are specified as JVM command line options, and Docker CPU limit are specified, the JVM will use the -XX:ParallelGCThreads and -XX:CICompilerCount values.
However, a few things need to be kept in mind as many libraries and frameworks that rely on thread pools will tune the number of threads based on these numbers.
The Docker **--cpus** flag specifies the percentage of available CPU resources a container can use. 
Specifically it refers to the cpu-shares. It does not adjust the number of physical processors the container has access to, which is what the jdk8 runtime currently uses when setting the number of processors. **This has been fixed with JDK 10.**
Another point to be noted is that cpu shares are enforced only when the CPU cycles are constrained. If there is no contention, then any container is free to use all of the host’s CPU time. 
E.g. in the above example if the second container is idle, then the first container can use all of the CPU cycles, even though it was configured with fewer shares.  
The Docker flag **--cpuset-cpus** is used to limit the cores a container can use. On a 4 core machine we can specify 0-3. We can specify a single core or even multiple cores by comma separating the index of the cores.

Example:  **--cpuset-cpus=0,1** will have the runtime properly seeing 2 cores available instead of 4. 
When we do this the runtime will then tune the number of compiler and garbage collection threads accordingly. 



 Similarly, to **tell the JVM to be aware of Docker memory limits in the absence of setting a maximum Java heap via -Xmx**, there were two JVM command line options required, **-XX:+UnlockExperimentalVMOptions** and **-XX:+UseCGroupMemoryLimitForHeap.**
Otherwise, JVM used to allocate one-fourth of the system’s memory for heap space- these values were extracted directly from the underlying host instead of Docker container. Using the  XX:+UnlockExperimentalVMOptions flag, we can use the -XX:+UseCGroupMemoryLimitForHeap flag to let the JVM check the Control Group memory limit and calculate a maximum heap size.
Here, we can use the **-XX:MaxRAMFraction** flag to help calculate a better heap size. However, MaxRAMFraction is hard to work with since it’s a fraction and must be chosen wisely. 
With Java 10 came better support for Container environment. If we run our Java application in a Linux container the JVM will automatically detect the Control Group memory limit with the UseContainerSupport option. 
We can control the memory with the options: **InitialRAMPercentage, MaxRAMPercentage and MinRAMPercentage.**

To summarise, as for openJDK, **efforts started in Java 8 (update 131) and Java 9. However it was finally solved in Java 10 (A couple of changes have already been back-ported). Now applying CPU and memory limits to our containerized JVMs is straightforward. The JVM will detect hardware capability of the container correctly, tune itself appropriately and make a good representation of the available capacity to the application. As a result, not only CPU Sets but also CPU Shares are now examined by JVM. Furthermore, this becomes the default behaviour**, and can only be disabled via **-XX:-UseContainerSupport** option. To be sure of your particular JVM, please make sure to check the final flags.
References:
- https://blogs.oracle.com/java-platform-group/java-se-support-for-docker-cpu-and-memory-limits
- https://www.docker.com/blog/improved-docker-container-integration-with-java-10/
- https://medium.com/@christopher.batey/cpu-considerations-for-java-applications-running-in-docker-and-kubernetes-7925865235b7
- https://merikan.com/2019/04/jvm-in-a-container/
