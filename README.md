# Reproduce _Download Sources_ bug

Example project that reproduces an issue with downloading sources in IDEA IntelliJ.
See [Request #1904460](https://intellij-support.jetbrains.com/hc/en-us/requests/1904460).

_**Note:** I am pushing all IntelliJ files (the whole `.idea` folder and `.iml` file) to help give the full picture,
you might have to remove or amend some of these to open the project correctly, to account for local differences..._ 

## How to reproduce?

Trying to open `org.junit.Test` in this project, I can only see decompiled version of the code.

![download-button](https://raw.githubusercontent.com/snooze92/reproduce-downloadsources-bug/master/screenshots/download-button.png)

Clicking the `Download...` link fails, no matter how many times I try.

![notification](https://raw.githubusercontent.com/snooze92/reproduce-downloadsources-bug/master/screenshots/notification.png)

![eventlog](https://raw.githubusercontent.com/snooze92/reproduce-downloadsources-bug/master/screenshots/eventlog.png)

## Versions affected

I use Mac OS X 10.13.4 and IntelliJ 2018.3. This affects 2018.2 and 2018.1 as well.
Trying a 2017 version works and does not reproduce the issue.
Running any version under Linux works and does not reproduce the issue.

I use Maven 3.3.9, either the version bundled with the IDE, or the maven@3.3 Homebrew formula. 

## Symptoms

Looking in the [`idea.log`](https://raw.githubusercontent.com/snooze92/reproduce-downloadsources-bug/master/idea.log) file,
the exception is clear, but does not provide much detail: 
``` 
2018-12-13 17:19:21,375 [ 94785] INFO - #org.jetbrains.idea.maven - org.eclipse.aether.resolution.ArtifactResolutionException: Could not find artifact <ARTIFACT_DESCRIPTION> 
java.lang.RuntimeException: org.eclipse.aether.resolution.ArtifactResolutionException: Could not find artifact <ARTIFACT_DESCRIPTION> 
at org.eclipse.aether.internal.impl.DefaultArtifactResolver.resolve(DefaultArtifactResolver.java:444) 
at org.eclipse.aether.internal.impl.DefaultArtifactResolver.resolveArtifacts(DefaultArtifactResolver.java:246)
at org.eclipse.aether.internal.impl.DefaultArtifactResolver.resolveArtifact(DefaultArtifactResolver.java:223) 
at org.eclipse.aether.internal.impl.DefaultRepositorySystem.resolveArtifact(DefaultRepositorySystem.java:294)
at org.jetbrains.idea.maven.server.Maven3ServerEmbedderImpl.resolve(Maven3ServerEmbedderImpl.java:1252)
at org.jetbrains.idea.maven.server.Maven3ServerEmbedderImpl.doResolve(Maven3ServerEmbedderImpl.java:1194)
at org.jetbrains.idea.maven.server.Maven3ServerEmbedderImpl.doResolve(Maven3ServerEmbedderImpl.java:1188)
at org.jetbrains.idea.maven.server.Maven3ServerEmbedderImpl.resolve(Maven3ServerEmbedderImpl.java:1057)
at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) 
at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) 
at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) 
at java.lang.reflect.Method.invoke(Method.java:498) 
at sun.rmi.server.UnicastServerRef.dispatch(UnicastServerRef.java:346) 
at sun.rmi.transport.Transport$1.run(Transport.java:200) 
at sun.rmi.transport.Transport$1.run(Transport.java:197) 
at java.security.AccessController.doPrivileged(Native Method) 
at sun.rmi.transport.Transport.serviceCall(Transport.java:196) 
at sun.rmi.transport.tcp.TCPTransport.handleMessages(TCPTransport.java:568) 
at sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run0(TCPTransport.java:826) 
at sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.lambda$run$0(TCPTransport.java:683) 
at java.security.AccessController.doPrivileged(Native Method) 
at sun.rmi.transport.tcp.TCPTransport$ConnectionHandler.run(TCPTransport.java:682) 
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) 
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) 
at java.lang.Thread.run(Thread.java:745) 
Caused by: java.lang.RuntimeException: org.eclipse.aether.transfer.ArtifactNotFoundException: Could not find artifact <ARTIFACT_DESCRIPTION> 
at org.eclipse.aether.internal.impl.DefaultArtifactResolver.resolve(DefaultArtifactResolver.java:434) 
... 24 more 
```

Having enabled DEBUG logging, I could also confirm that it goes to the correct internal Nexus repository: 
``` 
2018-12-13 17:19:21,353 [ 94763] WARN - ution.rmi.RemoteProcessSupport - [RMI TCP Connection(4)-127.0.0.1] DEBUG org.eclipse.aether.internal.impl.DefaultLocalRepositoryProvider - Using manager EnhancedLocalRepositoryManager with priority 10.0 for /Users/<MY_USERNAME>/.m2/repository 
2018-12-13 17:19:21,365 [ 94775] WARN - ution.rmi.RemoteProcessSupport - [RMI TCP Connection(4)-127.0.0.1] DEBUG org.eclipse.aether.internal.impl.DefaultRemoteRepositoryManager - Using mirror nexus (<CORRECT_NEXUS_URL>) for nexus (<CORRECT_NEXUS_URL>). 
```

## What did we try?

I tried:
- click 10 times (in the past, it would work on the 2nd click when trying twice...) but that fails consistently; 
- use `mvn dependency:sources` on the command-line, that works, and then some of the sources do load in the IDE (it's unclear whether the IDE finds them in my local .m2 or if it then finds it on the proxy, which is "lazy" and does not list artifacts until they are requested once); 
- change Maven settings (which version, parallelism...) without success; 
- change Maven importer settings (give it more memory, use different JDKs, automatic sources download...) without success; 
- turn offline mode on or off, without success; 
- invoke the IDE download of sources (see screenshot) without success; 
- remove network-related Custom VM Options without success; 
- try adding `idea.max.intellisense.filesize=4096` to Custom Properties without success; 
- re-indexed the repositories without success.

A colleague of mine also tried: 
- disable all non-default plugins without success; 
- look at network traffic: see proper traffic towards our Nexus instance when using `mvn dependency:sources` but see none when asking IDEA IntelliJ to download sources.
