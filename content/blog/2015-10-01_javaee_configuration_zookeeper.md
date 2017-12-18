---
author: "Jørgen Ringen"
title: "JavaEE: Externalising configuration with ZooKeeper"
date: 2015-10-01
slug: javaee_configuration_zookeeper
---

An important principle in [continuous delivery](https://continuousdelivery.com/) is to 
build application binaries only _once_ and let the exact same binary flow through each step of the [build pipeline](https://devops.com/continuous-delivery-pipeline/) in order to verify that the binary is ready for production.

A typical build pipeline might look something like this:

1. Commit stage (build, unit test, analysis) 
2. Automated acceptance tests 
3. Capacity testing 
4. User acceptance testing 
5. Production

Many of these steps will often require different application configuration. We might want to stub out some backend services, use different username/password for each environment, etc.

Unfortunately there’s a convention in the Java EE spec to 
package the configuration inside the war or ear file together with the application. 
This means that we have to configure the application build time. 
When we do this we have to create a different application binary for different deployment environments. 
This is a serious risk and an anti pattern in continuous delivery.

---

### Externalising configuration with Apache ZooKeeper

[Apache ZooKeeper](https://zookeeper.apache.org/) is, among many things, a centralised service for maintaining configuration information. 
ZooKeeper maintains a tree-structure with nodes and child-nodes. 
Each node in ZooKeeper can have data as well as children associated with it. 
This is a great data structure for storing configuration. 
ZooKeeper also provides a mechanism called watches, where clients can listen for changes.
This means that we can update configuration at runtime.

There’s an official [Docker image for Apache Zookeeper](https://hub.docker.com/_/zookeeper/) that will help you get Zookeeper up and running. 

We’ll connect to ZooKeeper using the CLI and add some nodes to store our configuration:
```
create /myapp root
create /myapp/config config
create /myapp/config/endpoints endpoints
create /myapp/config/endpoints/twitter my-twitter-test.com
```

---

### Using Curator to connect to ZooKeeper from Java EE

ZooKeeper has a very low-level API and [Apache Curator](https://curator.apache.org/) is a framework client library for ZooKeeper
that has a nice API and is quite easy to work with.

The first thing we need to do is to add the curator client dependencies in our pom:

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>2.9.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-client</artifactId>
    <version>2.9.0</version>
</dependency>
```

Too instantiate curator we will use a @Startup @Singleton bean. The url to ZooKeeper should be provided as an application parameter (avoid build time configuration, remember).

```java
import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.api.CuratorWatcher;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import javax.ejb.Singleton;
import javax.ejb.Startup;
import java.util.HashMap;
import java.util.Map;

@Singleton
@Startup
public class ZooKeeperRegistry {

	private String zookeeperHost = "127.0.0.1:2181"; // use jvm argument
	private CuratorFramework client;
	private Map<String, String> properties = new HashMap<>();

	public String getProperty(String key) {
		return properties.get(key);
	}

	public Map<String, String> getProperties() {
		return properties;
	}

	@PostConstruct
	private void postConstruct() throws Exception {
		// establish connection to ZooKeeper
		RetryPolicy retryPolicy = new ExponentialBackoffRetry(10000, 3);
		client = CuratorFrameworkFactory.newClient(zookeeperHost, retryPolicy);
		client.start();

		// get data for node, add a watcher to retrieve updates
		byte[] endpoint = client.getData().usingWatcher(new MyWatcher()).forPath("/myapp/config/endpoints/twitter");
		properties.put("twitter-endpoint", new String(endpoint));
	}

	@PreDestroy
	private void preDestroy() {
		client.close();
	}

	private class MyWatcher implements CuratorWatcher {

		@Override
		public void process(WatchedEvent event) throws Exception {
			if (event.getType() == Watcher.Event.EventType.NodeDataChanged) {
				// get changed data, re-register the watcher to get further notifications
				byte[] bytes = client.getData().usingWatcher(this).forPath(event.getPath());
				properties.put("twitter", new String(bytes));
			}
		}
	}
}
```

This is just a simple example on how you can store your configuration data outside the war or ear.

There are many management and visualisation tools available for ZooKeeper and there are plugins for both Intellij and Eclipse. ZooKeeper also comes bundled with a gui tool called ZooInspector.

There are also many alternatives to ZooKeeper, like [Spring Config Server](https://spring.io/guides/gs/centralized-configuration/), [Consul](https://www.consul.io/) and [etcd](https://github.com/coreos/etcd).

You can also just use simple jvm-arguments or store configuration in a database, but the most important point is to remove the build-time dependency.