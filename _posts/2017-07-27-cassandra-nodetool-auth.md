---
title:  "Cassandra nodetool authentication using the internal authorizer"
date:   2017-07-27 00:00:00 -0600
categories: cassandra nodetool
---
I learned about a new feature in Cassandra 3.6+ today, the ability to enable nodetool authentication by using Cassandra's [built-in authorizer.](http://docs.datastax.com/en/cassandra/3.0/cassandra/configuration/secureConfigNativeAuth.html) This turned out to be very useful, as I no longer need to manage a username/password in CQL in addition to a username/password in JMX. I'll quickly go through the steps I took to set this up. I did it in Docker, but the steps should be the same in a VM as well.

First, Cassandra's internal authentication/authorization needs enabled. There are two lines in `cassandra.yaml` that need changed:

```diff
- authenticator: AllowAllAuthenticator
- authorizer: AllowAllAuthorizer
+ authenticator: PasswordAuthenticator
+ authorizer: CassandraAuthorizer
```

This will enable authentication when you try to log in with cql, or when an app tries to connect with cql. The default username and password is `cassandra:cassandra`. Now, we want to use that same username/password (or any unique username/password we create) for nodetool. Let's set up nodetool auth using Cassandra's internal database.

The nodetool auth config is set up using Java flags. We will need to add and remove a few flags in the `cassandra-env.sh` file. Let's start with adding flags. Add the following to your `cassandra-env.sh`:

* `JVM_OPTS="$JVM_OPTS -Djava.security.auth.login.config=$CASSANDRA_HOME/cassandra-jaas.config"`
* `JVM_OPTS="$JVM_OPTS -Dcassandra.jmx.authorizer=org.apache.cassandra.auth.jmx.AuthorizationProxy"`

This flag tells Cassandra to use the `cassandra-jaas.config` file to set up its auth. The content of this file is important. If you downloaded Cassandra as a tar, this file may already exist. If not, it needs created with the following content, in the `$CASSANDRA_HOME` directory:
```
// Delegates authentication to Cassandra's configured IAuthenticator
CassandraLogin {
  org.apache.cassandra.auth.CassandraLoginModule REQUIRED;
};
```

We also need to remove (or comment) a few flags (if they exist) from `cassandra-env.sh`. These are the flags that set up the legacy jmx auth. The flags that need removed are:
* `-Dcom.sun.management.jmxremote.password.file=/etc/cassandra/jmxremote.password`
* `-Dcom.sun.management.jmxremote.access.file=/etc/cassandra/jmxremote.access`

Last, start Cassandra and ensure you get an error running nodetool with no username/password:
```
Exception in thread "main" java.lang.SecurityException: Authentication failed! Credentials required
```

You should now be able to run nodetool with a username/password and get success:
`nodetool -u cassandra -pw cassandra status`

Next steps: create a unique username/password with cql, and try using it to authenticate to nodetool.

---
### Further reading:
Here is an [example](https://github.com/dcparker88/cassandra-kubernetes/blob/f9d1c769c34111354e6973f2133430e12a24a26c/image/Dockerfile#L22-L24) of me configuring this type of auth in a Docker container, that I am going to use in Kubernetes. This will be useful because I will only need to have one set of credentials that I manage through cql. I won't need to manage a separate set, and a separate jmx.password file with those credentials.
