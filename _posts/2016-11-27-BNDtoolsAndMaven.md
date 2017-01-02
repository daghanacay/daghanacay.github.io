---
published: true
layout: post
title:  "BND tools, MVN, and Gradle"
categories: AWS, streaming
---

* place holder for TOC
{:toc}

In this blog I will talk about the bnd tools plugin for eclipse, mvn and gradle plugins that can be used to aid CI/CD processes.

# Bnd tools repositories

BND tool requires a set of repositories where resolution process can find the dependencies. Developers are responsible to construct this repositories at development time. There are multiple ways to construct this repositories as described below.

## BND Pom repository

This is the repository that automatically exposes mvn repositories. Details can be found at [bnd pom repository]. The benefit of this repository is to download transient dependencies automatically. You can add this repository to your bnd configuration as follows:

```
-plugin.2.easyiot.device = \
	aQute.bnd.repository.maven.pom.provider.BndPomRepository; \
		snapshotUrls=https://oss.sonatype.org/content/groups/osgi; \
		releaseUrls=https://repo1.maven.org/maven2/; \
		pom=${build}/EasyDeviceMaven.xml; \
		name=EasyDeviceMaven; \
		location=${build}/cache/EasyDeviceMaven.xml
```

The index file identified by pom parameter e.g. *EasyDeviceMaven.xml* has a very familiar format with the maven file.

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>local</groupId>
	<artifactId>EasyIotMaven</artifactId>
	<version>0.0.0</version>

	<packaging>pom</packaging>

	<dependencies>
		<dependency>
			<groupId>com.easyiot</groupId>
			<artifactId>com.easyiot.color3led.device.api</artifactId>
			<version>1.0.0-SNAPSHOT</version>
		</dependency>
    </dependencies>
</project>
```

## Fixed index repository

Uses an xml based index file as bundle references. The benefit of this kind of index is to bypass artefactories like maven. You can upload jar files of the bundles and attach this index for the bnd resolver to find and download and use the files. For example, you can upload your build artefacts to github along with the code and refer to the index file using raw.github URL to distribute your artefacts. You can add it to your bnd configuration as follows:

```
-plugin.2.easyiot.device = \
	aQute.bnd.deployer.repository.FixedIndexedRepo; \
	        name		=       EasyIot-Device; \
	        locations	=       https://raw.githubusercontent.com/daghanacay/com.easyiot.device/master/cnf/release/index.xml
```

The index file is one of the most compicated content and very hard to read but you can create these indexes automatically through relasing from BND tools or using mvn plugin as will be discussed later in this blog.

```
<repository increment="1480839251000" name="Release" xmlns="http://www.osgi.org/xmlns/repository/v1.0.0">
  <resource>
    <capability namespace="osgi.identity">
      <attribute name="osgi.identity" value="com.easyiot.LT100H.device.api"/>
      <attribute name="type" value="osgi.bundle"/>
      <attribute name="version" type="Version" value="1.0.0.201612040814"/>
    </capability>
    <requirement namespace="osgi.wiring.package">
      <directive name="filter" value="(&amp;(osgi.wiring.package=org.osgi.dto)(version&gt;=1.0.0)(!(version&gt;=2.0.0)))"/>
    </requirement>
  </resource>
</repository>
```

## JPM repository (deprecated)

JPM repository is a wrapper around maven central repository with additional metadata for OSGi bundles. It is important to know that JPM repository does not save transient dependencies. That is if *bundle A* dependencies on *bundle B* and you only store *bundle A's* reference in your JPM repository, your build will break. The reason is that JPM repository will not automatically look for transient dependencies, you have to manually add reference to *bundle B* in your repository. JPM repository is attached to your build process using the jpm.Repository plugin as follows.

```
-plugin.enroute.centralrepo = \
    aQute.bnd.jpm.Repository; \
            includeStage    =       true; \
            name            =       Central; \
            location        =       "${build}/cache/jpm"; \
            index           =       ${build}/repository.json
```

This plugin refers to *repository.json* as the repository file. The file contents looks as below:

```
{
 "revisionRefs":[{
  "artifactId":"aQute.angular-ui","baseline":"0.6.0","bsn":"aQute.angular-ui","created":1381308874426,"description":"Bootstrap components written in pure AngularJS by the AngularUI Team","groupId":"osgi","md5":"AD58D9F4870CC5362486D054CDBB6CBC","name":"aQute.angular-ui","phase":"MASTER","qualifier":"201310090854","revision":"7D0FD4A839B65352D7B27AB479D38708397A8A0C","size":27440,"urls":["http://repo.jpm4j.org/rest/bundle/51C83986E4B06EF1574B84F7/7d0fd4a839b65352d7b27ab479d38708397a8a0c"],"version":"0.6.0.201310090854"
 }]
}
```

As one can see the .json file has quite a lot information about the artifact called *aQute.angular-ui* including version, where to download etc. It is laborous to create this file but eclipse BNDTools plugin aid you to construct this files.

## Maven BND repository

This repsoitory uses more user friendly xml fiels as their index details can be found at [mvn bnd repository]. Maven BND repositry is similar to jpm repository is similar to JPM in that it does not brings in the transient dependencies automatically. However, Maven BND tool has a more familiar index file format than jpm repository. For those who are familiar with maven will find it easier to read and manipulate manually. You can add a Maven BND repository to your project as follows:

```
-plugin.central = \
	aQute.bnd.repository.maven.provider.MavenBndRepository; \
		releaseUrl=https://repo.maven.apache.org/maven2/; \
		index=${.}/central.maven; \
		name="Central"
```

The the contents of the index file *central.maven* that stores the repository information is shown below

```
activespace:jcache:1.0-dev-3
com.google.guava:guava:19.0
```

There are tools to create index file including the BNDtools eclipse plugin.

# Maven plugins

You can do most of the actions the bndtool does using MVN plugins [mvn plugins]. This requires you adding an additional pom file in your project to company the bnd configuration. Here are some of the important ones:

* bnd-maven-plugin: creates OSGi metadata for your project.
* bnd-indexer-plugin: A plugin used to generate an OSGi repository index from a set of Maven dependencies. these indexes can be used by aQute.bnd.deployer.repository.FixedIndexedRepo during the resolution process
* bnd-baseline-maven-plugin: Checks the semantic verisioning rules against a baseline. It searches all the files and compares the code with the baseline code to determine the semantic version change on the exisiting code and suggest version updates if any changes is detected.
* bnd-export-maven-plugin: Exports the project with all the depenedencies into an executable jar file. See also Gradle export function.

# Gradle

## Building

Builds all the projects in a workspace and runs the integration test projects name ending with test *.*.*.test.

## Exporting executable jar

You can export all the executable files for .bnd files defined in project with gradle using as single command:

```
$ ./gradlew export
```

You can also export an individual executable by adding "export." infront of your bnd file.

```
$ ./gradlew export.debug.bnd
```
[mvn bnd repository]:http://bnd.bndtools.org/plugins/maven.html
[bnd pom repository]:http://bnd.bndtools.org/plugins/pomrepo.html
[mvn plugins]:https://github.com/bndtools/bnd/tree/master/maven
