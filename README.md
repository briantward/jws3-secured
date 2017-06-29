# Securing Red Hat JBoss Web Server 3.X
(run tests as 3.0.3 but should be good for 3.1 as well)

## RPM Base Installation

1. Register your machine if you have not done so.

```
# subscription-manager register
```

2. List the available subscriptions you have.  Look for the Pool ID of your JBoss Web Server subscription

```
# subscription-manager list --available
```

3. Attache the correct subscription.

```
# subscription-manager attach --pool=$POOL_ID
```

4. Verify it has been attached 

```
# subscription-manager list --consumed
```

5. Enable the repo

```
# subscription-manager repos --enable=jws-3-for-$RHEL_VERSION-server-rpms
```

6. Install

```
# yum groupinstall jws3
```
or
```
# yum groupinstall jws3plus
```

## Zip Base Installation


1. Download zip file from access.redhat.com for your OS.
2. Unzip to secured location of your choice, such as /opt
3. Run post-install script.  This should configure your apache server config files with full
path filenames based on your choice of installation directory.  It should also configure the
apache user with /sbin/nologin shell.

```
$ sudo ./$APACHE_HOME/.postinstall
```

If the script did not correctly install the user, you've probably done something incredibly wrong.  
If you want to do this part yourself and you don't want absolute URLS in your configs:

```
$ sudo groupadd -g 48 -r apache
$ sudo useradd -c "Apache" -u 48 -g apache -s /sbin/nologin -r apache
$ sudo chown -R apache:apache httpd
$ sudo passwd -l apache
```

4. Lock out the apache user.

```
passwd -l apache
```

## Test the server base installation.

1. Create a test file index.html in $WWW_ROOT:

```
<html>
<body>
test
</body>
</html>
```

2. Start the server.  It defaults on port 80, so you must use the root account to start it.

```
$ sudo ./sbin/apachectl start
```

3. Test it.  Be sure this returns the above index.html data.

```
$ curl http://localhost:80
```

4. Stop the server.

```
$ sudo ./sbin/apachectl stop
```

## Start securing it.

1. Backup config files for reference.  Remove them for production use. 

```
$ cp -Ra config config.backup
$ cp -Ra config.d config.d.backup 
$ cp -Ra confg.modules.d config.modules.d.backup
```
2. Change the following existing directives in httpd.conf:

ServerTokens Prod
Timeout 10
KeepAlive On
Listen 127.0.0.1:8080
ServerName localhost:8080
Options None
ServerSignature Off

3. Add the following directives:


Comment out Alias, ScriptAlias, cgi-bin features
Comment out Includes folder - disables all subconfigs, reenable if any are needed and be sure to comment out ones you don't want.

3. In conf.modules.d folder, comment out all lines of the below files.  Do not delete or rename the files, as during an RPM upgrade they would simply be replaced.

    - 00-dav.conf
    - 00-proxy.conf
    - 00-proxyhtml.conf 
    - 00-ssl.conf 
    - 01-cgi.conf 
    - 01-ldap.conf
    - 01-session.conf 
    - 10-auth_kerb.conf

8.  That should leave two more files in conf.modules.d:

    - 00-base.conf
    - 00-mpm.conf


comment out in 00-base.conf


