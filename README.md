# demo-vt-issues
How to demonstrate virtual threads performance issues 


# Reproducing the virtual threads issues using the AcmeAir-authservice application in Open Liberty

## Hardware and operating system configurations

It is preferable, from a performance analysis perspective, to have two separate servers:

- the System Under Test (SUT)
- the load driver

Doing this isolates the two JVMs to avoid interactions.

If you cannot get a separate load driver system, you can run the load driver on the same server as the SUT. If you do this, the SUT and load driver must be pinned to separate CPU sets, so that the behavior of the SUT is minimally affected by the CPU and memory consumption of the load driver.

Your choice of operating system for the SUT depends on which virtual threads issue you want to reproduce:

- we see the virtual threads "low CPU, low throughput" issue most clearly on bare metal Linux/Intel systems with kernel 4.x (e.g. RHEL 8 )
- we see a mix of "low CPU, low tps" and "high CPU, high tps" behavior, with kernel 5.x (e.g. RHEL 9)
- we do not see "low CPU, low tps" on kernel 6.x, or on virtualized systems; they show only "high CPU, low tps"

(See the "How to reproduce the "unexpected virtual threads performance findings" section for more details.)

As normal with performance work, the SUT should be running only the test workload; if other processes are running on the system, the test workload behavior might be affected.

## Building and configure Liberty on the SUT

Because we concluded from our investigations that we would not replace Liberty's existing autonomic threadpool with virtual threads, there is no release build of Liberty using virtual threads to handle HTTP traffic. We have made a Liberty virtual threads enablement patch PR available for test/demo purposes. You can either build Liberty in its entirety with the patch PR included, or use an Open Liberty build from the download site and patch that build by swapping in the channel framework jar file from the PR, which contains the patch. 

1. Configure Java on the SUT:

    1. On the SUT, [download Java 21 or later](https://developer.ibm.com/languages/java/semeru-runtimes/downloads/).
    2. In a terminal, set `JAVA_HOME` to point to the Java 21+ executable:

      ```
      # export JAVA_HOME=<path-to-java-dir>
      ```

2. [Build Open Liberty](https://github.com/OpenLiberty/open-liberty?tab=readme-ov-file#running-a-build) with the virtual threads patch from GitHub:

   - [Open Liberty source code](https://github.com/OpenLiberty/open-liberty)
   - [virtual threads enablement patch PR](https://github.com/OpenLiberty/open-liberty/pull/28317)

   The virtual threads enablement patch provides a new channel framework project (a new channelfw JAR file) in the Open Liberty build.

2. Configure Open Liberty on the SUT:

   1. On the SUT, extract the built Open Liberty package.

   3. [Build the AcmeAir auth service app](https://github.com/blueperf/acmeair-authservice-java/blob/main/Build_Instructions.md), then download the WAR file and place it on the SUT.

   4. Create a Liberty server to deploy the app:

         ```
         # cd <Liberty-install>/wlp
         # ./bin/server acmeAirAuth create
         ```

   6. Edit the `server.xml` file in the Liberty server:

      ```xml
      # vim usr/server/acmeAirAuth/server.xml
      ```

   7. In the `server.xml` file, replace the `featureManager` and `httpEndpoint` content  (these are generic defaults when a new server is created) with the following lines:

      ```xml
      <featureManager>
         <feature>microProfile-6.1</feature>
      </featureManager>

      <logging consoleFormat="simple" maxFiles="10" />

      <monitor  filter="REST"/>

      <httpEndpoint id="defaultHttpEndpoint" host="*" httpPort="9080" httpsPort="9443">
         <tcpOptions soReuseAddr="true" />
         <httpOptions maxKeepAliveRequests="-1" />
      </httpEndpoint>

      <webApplication type="war" location="<path-to>/acmeair-authservice-java-6.1.war" context-root="/">
         <classloader apiTypeVisibility="+third-party" />
      </webApplication>
      ```

   8. In the `server.xml` file, replace `<path-to>` in the `webApplication` element with the location where the AcmeAir auth service WAR was placed.
   9. Start the server:

      ```
      # ./bin/server acmeAirAuth start
      ```

   10. Check the server logs after start completes to verify there are no errors, and that the server and application started successfully. The `messages.log` file should contain entries like this:

         ```
         CWWKF0011I: The acmeAirAuth server is ready to run a smarter planet. The acmeAirAuth  server started in 6.676 seconds.
         ....
         SRVE0242I: [acmeAirAuth ] [/] [com.acmeair.web.AuthServiceApp]: Initialization successful.
         ```

   11. Create a file to hold the Java options for the server:

         ```
         # vim usr/server/acmeAirAuth/jvm.options
         ```

   12. Add the following lines in the `jvm.options` file:

         ```
         -Xms2g
         -Xmx2g
         -Dotel.sdk.disabled=true
         #-DwqmUseVirtualThreads=true
         ```

      The last item in the `jvm.options` file is the system property that controls the "Liberty virtual threads demo patch". The property defaults to `false`:  by default, HTTP requests are served by threads from the Liberty default thread pool (a JUC ExecutorService).
      
      When the property is set to `true` (by uncommenting the line in the `jvm.options` file) and restarting the acmeAirAuth server, HTTP requests submitted to the server are served using virtual threads.

   You should now have a working Liberty server configured with an app that responds to REST endpoint with an "OK" message.

   You can check that the app responds correctly by using curl:

      ```
      # curl -s -w ''%{http_code}'' http://<ip-addr-or-hostname>:9080/status ; echo ""
      OK200
      ```

4. Prepare the load driver on the load driver system:

   1. Download the current version of [Apache JMeter](https://jmeter.apache.org/download_jmeter.cgi).
   2. Extract the downloaded ZIP fle on to the load driver system.
   3. Extract the attached `acmeair-demo-jmx.zip` and place the enclosed file `microprofile_primitives.jmx` in the `bin` directory of the JMeter installation.
   [LAURA: where is this attached?]
   4. In a terminal on the load driver system, in the JMeter `bin` directory, run the following command (with the SUT acmeAirAuth server started and ready to receive HTTP requests):

      ```
      taskset -c 7-10 ./jmeter.sh  -n -t microprofile_primitives.jmx -JHOST=<SUT-host/ip> -JPORT=9080 -JTHREAD=10 -JDURATION=30 -JRAMP=10 -JURL=/status ; sleep 5 ;taskset -c 7-10 ./jmeter.sh  -n -t microprofile_primitives.jmx -JHOST=<SUT-host/ip> -JPORT=9080 -JTHREAD=25 -JDURATION=90 -JRAMP=10 -JURL=/status
      ```

      - Replace <SUT-host/ip> with the hostname or IP address of the SUT where the acmeAirAuth server is running.
      - The example pins the JMeter process to CPUs 7-10 on the load driver; adjust the CPU selection to suit the system that your load driver is running on.
      
   You should see output from JMeter like this:

      ```
      Creating summariser <summary>
      Created the tree successfully using microprofile_primitives.jmx
      Starting standalone test @ Tue May 07 16:35:20 EDT 2024 (1715114120197)
      Waiting for possible Shutdown/StopTestNow/HeapDump/ThreadDump message on port 4445
      summary +      1 in 00:00:00 =    7.0/s Avg:    30 Min:    30 Max:    30 Err:     0 (0.00%) Active: 1 Started: 1 Finished: 0
      summary +  94357 in 00:00:09 = 10060.5/s Avg:     0 Min:     0 Max:    47 Err:     0 (0.00%) Active: 10 Started: 10 Finished: 0
      summary =  94358 in 00:00:10 = 9909.5/s Avg:     0 Min:     0 Max:    47 Err:     0 (0.00%)
      summary + 205738 in 00:00:10 = 20573.8/s Avg:     0 Min:     0 Max:    75 Err:     0 (0.00%) Active: 10 Started: 10 Finished: 0
      summary = 300096 in 00:00:20 = 15372.2/s Avg:     0 Min:     0 Max:    75 Err:     0 (0.00%)
      summary + 205017 in 00:00:10 = 20501.7/s Avg:     0 Min:     0 Max:   111 Err:     0 (0.00%) Active: 10 Started: 10 Finished: 0
      summary = 505113 in 00:00:30 = 17109.7/s Avg:     0 Min:     0 Max:   111 Err:     0 (0.00%)
      summary +  11167 in 00:00:01 = 21030.1/s Avg:     0 Min:     0 Max:    13 Err:     0 (0.00%) Active: 0 Started: 10 Finished: 10
      summary = 516280 in 00:00:30 = 17179.0/s Avg:     0 Min:     0 Max:   111 Err:     0 (0.00%)
      Tidying up ...    @ Tue May 07 16:35:50 EDT 2024 (1715114150532)
      ```

   Check to make sure that the `Err: count` is zero. This confirms that requests are being served as expected.

## How to reproduce the "unexpected virtual threads performance findings"

The article describes two main categories of unexpected virtual thread performance:

- low CPU, low tps - this was seen consistently on Linux kernel 4.x (RHEL 8 )
- high CPU, low tps - this was seen consistently on Linux kernel 6.2 (Ubuntu 22.04)
  - Linux kernel 5.x (RHEL 9) showed intermittent "low CPU, low tps" and more often showed "high CPU, low tps"

In this context, a few definitions:

- low CPU: the SUT is not using all the available CPU, although it has enough offered load to do so.
- high CPU: the SUT is using all the available CPU.
- low tps: the SUT is handling significantly less traffic than a baseline established by running the same test with the SUT configured to use the Liberty thread pool to serve the offered load.

So if you want to see "low CPU, low tps", set up a test system with Linux kernel 4.x ... if you want to see "high CPU, low tps" use a system with Linux kernel 5.x (which might show some "low CPU, low tps" on occasion) or Linux kernel 6.x.

Or, stating it the other way around, if you can only get a test system with Linux kernel 4.x, then you should be able to reproduce "low CPU, low tps" but probably not "high CPU, low tps", and vice-versa if you can only get a test system with Linux kernel 5.x or 6.x.

The "low CPU, low tps" behavior is most prominent when running the SUT (Liberty) on two CPUs, and "high CPU, low tps" is very evident with the SUT on two CPUs, so for simplicity we recommend running the SUT on two CPUs. This can be done on a many-CPU LinTel system with the taskset -c N,M command as a prefix to the Liberty start command, where N,M are the two CPUs to run on. For example, this will launch Liberty acmeAirAuth server on CPUs 2 and 3:

```
taskset -c 2,3 <wlp-install>/bin/server start acmeAirAuth
```

If the JMeter load driver process will run on a different system than the Liberty SUT, then it can run on any CPUs on that system. The only concern is to make sure that the JMeter process has enough cpu and memory so that it can fully load the SUT; the load driver should not be the bottleneck.

If you only have a single system to work with, but it has enough CPUs to host both the SUT and the load driver, then make sure to pin the JMeter process to a different set of CPUs than the SUT is using, preferably on a different chipset if available. For example, if  acmeAirAuth is launched using `taskset -c 2,3` as in the example above, then JMeter may be launched with `taskset -c 10-15`.

In the preceding example, JMeter is given more CPUs than the acmeAirAuth SUT. This is intentional, so that JMeter is able to fully load the SUT. When you run a test with acmeAirAuth configured to use the Liberty thread pool, the 2-CPU SUT should show near 100% CPU utilization, while the JMeter load driver CPUs should show plenty of idle CPU.

Speaking of observing the CPU utilization: when working in LinTel, NMON is a great tool for observing utilization of various system resources, including CPU. When I run these experiments, I always have an NMON GUI open on a terminal into each system (SUT host and load driver host).

With the SUT and load driver CPU configuration sorted out, you are now ready to do some A-B testing, comparing performance using Liberty's thread pool against performance using virtual threads. The A-B control is this system property in the jvm.options file:

```
#-DwqmUseVirtualThreads=true
```

With that property commented out (or set to `false`), Liberty will serve the offered load with the Liberty default thread pool. With the property set to `true`, virtual threads will be used to serve the offered load.


# Any questions?

Contact us on developers@openliberty.io
