# Securing Red Hat JBoss Web Server 3.X
(configured on 3.0.3 but should be good for 3.1 as well)

THIS CONFIGURATION HAS NOT BEEN TESTED!  THIS IS INTENDED AS A STARTING GUIDE ONLY.

## RPM Base Installation

1. Register your machine if you have not done so.

    ```
    # subscription-manager register
    ```

2. List the available subscriptions you have.  Look for the Pool ID of your JBoss Web Server subscription

    ```
    # subscription-manager list --available
    ```

3. Attach the correct subscription.

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
    $ sudo passwd -l apache
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

    ```
    # CIS 8.1 Set ServerToken to 'Prod'
    ServerTokens Prod
    ```
    ```
    # CIS 9.1 
    Timeout 10
    ```
    ```
    # CIS 9.2
    KeepAlive On
    ```
    ```
    # CIS 5.13 Restrict Listen Directive
    Listen 127.0.0.1:8080
    ```
    
    ```
    ServerName localhost:8080
    ```
    
    ```
    DirectoryIndex index.html 
    ```
    
    ```
    <Directory "ANY">
    ...
    Options None
    ...
    </Directory>
    ```
    
    ```
    # CIS 8.2 Set ServerSignature to 'Off'
    ServerSignature Off
    ```

3. Add the following directives:

    Near the beginning of the file, where other server-level configs are placed:


    ```
    # CIS 5.8 Disable HTTP TRACE Method
    TraceEnable Off
    ```
    ```
    # CIS 10.1
    LimitRequestLine 512
    ```
    ```
    # CIS 10.2
    LimitRequestFields 100
    ```
    ```
    # CIS 10.3
    LimitRequestFieldsize 1024
    ```
    ```
    # CIS 10.4
    LimitRequestBody 102400
    ```
    
    Below Includeconf.modules.d/\*.conf line add the following:
    
    ```
    Include conf.modules.d/*.conf
    ...
    # CIS 5.9 Restrict HTTP Protocol Versions 
    # (currently non 1.1, but could be altered to refuse 1.0 but allow 1.1+ in
    # preparation for HTTP 2.0) 
    RewriteEngine On
    RewriteCond %{THE_REQUEST} !HTTP/1\.1$
    RewriteRule .* - [F]
    
    # CIS 5.12 Deny IP Address Based Requests
    RewriteCond %{HTTP_HOST} !^localhost [NC]
    RewriteCond %{REQUEST_URI} !^/error [NC]
    RewriteRule ^.(.*) - [L,F]
    
    # CIS 5.14 Restrict Browser Frames
    Header always append X-Frame-Options SAMEORIGIN
    ```
    
    ```
    <Directory "/opt/jws/jws-3.0/httpd/www/html">
    ...
    # CIS 5.7 Limit HTTP Request Methods
    
        AllowMethods GET POST OPTIONS
    ...
    </Directory>
    ```
    
    ```
    # CIS 5.11 Restrict File Extensions
    <FilesMatch "^.*\..*$">
        Require all denied
    </FilesMatch>
    
    <FilesMatch "^.*\.(css|html?|js|pdf|txt|xml|xsl|gif|ico|jpe?g|png)$">
        Require all granted
    </FilesMatch>
    ```

4. These directives stay the same.  You may want to comment that they are CIS specs and should not be changed.

    ```
    # CIS 9.3
    MaxKeepAliveRequests 100
    ```
    
    ```
    # CIS 9.4
    KeepAliveTimeout 15
    ```
    
    ```
    # CIS 3.1 Run the Apache Web Server as a non-root user
    User apache
    Group apache
    ```
    
    ```
    # CIS 9.5, 9.6
    <IfModule reqtimeout_module>
      RequestReadTimeout header=20-40,MinRate=500 body=20,MinRate=500
    </IfModule>
    ```
    
5. Comment out the following sections:

    - The settings for providing icon images, which won't be needed because auto-indexing will be turned off.
    - The WebDAV settings, which won't be activated anyways because we will turn off the module.
    - The CGI bin settings and ScriptAlias, because we won't be using it.
    - The IndexOptions, which won't be needed because auto-indexing will be turned off.
    - The settings for AddIconByEncoding, AddIconByType, AddIcon, DefaultIcon, ReadmeName, HeaderName, and IndexIgnore, which won't be needed because auto-indexing will be turned off.
    - The Alias for error and the settings for the error page by language, because this feature requires Server Side Includes which will be disabled.
    - IncludeOptional directive, which disables all subconfigs. Reenable if any are needed and be sure to comment out ones you don't want.

6. Comment out all the files in conf.d.  Even though we have commented out the IncludeOptional line in httpd.conf, we may want to add some features back, sometime in the future.  Doing this proactively means nothing else gets turned on by accident.

    - manual.conf
    - mod_cluster.conf
    - proxy_ajp.conf
    - ssl.conf
    - userdir.conf
    - welcome.conf

    Note that SSL is disabled here.  Configuring SSL is not in the scope of this document, at this time.  The assumption here is that SSL has been offloaded by a load balancer and this httpd host is in a secured subnet.  Also be aware that this burnt some companies in the past who had private data lines between subnets and thought their traffic was secured.  

7. In conf.modules.d folder, comment out all lines of the below files.  Do not delete or rename the files, as during an RPM upgrade they would simply be replaced.

    - 00-dav.conf
    - 00-proxy.conf
    - 00-proxyhtml.conf 
    - 00-ssl.conf 
    - 01-cgi.conf 
    - 01-ldap.conf
    - 01-session.conf 
    - 10-auth_kerb.conf

7.  That should leave two more files in conf.modules.d:

    - 00-base.conf, comment out...
        
        ```
        # This provides use of Order, Allow, Deny syntax from apache 2.2.  Disable for new servers using 
        # # Require syntax from mod_authz_host
        #LoadModule access_compat_module modules/mod_access_compat.so
        
        # Disable CGI Actions, unless specifically needed
        #LoadModule actions_module modules/mod_actions.so
        
        # After removing any default Alias entries, remove this module unless you specifically need it
        #LoadModule alias_module modules/mod_alias.so
        ```
        ```
        # These are for header based authentication, disable unless specifically needed
        #LoadModule auth_basic_module modules/mod_auth_basic.so
        #LoadModule auth_digest_module modules/mod_auth_digest.so
        #LoadModule authn_anon_module modules/mod_authn_anon.so
        #LoadModule authn_core_module modules/mod_authn_core.so
        #LoadModule authn_dbd_module modules/mod_authn_dbd.so
        #LoadModule authn_dbm_module modules/mod_authn_dbm.so
        #LoadModule authn_file_module modules/mod_authn_file.so
        #LoadModule authn_socache_module modules/mod_authn_socache.so
        ```
        ```
        # These are for database backed auth, disable unless specifically needed
        #LoadModule authz_dbd_module modules/mod_authz_dbd.so
        #LoadModule authz_dbm_module modules/mod_authz_dbm.so
        ```
        ```
        # CIS 2.5 Disable Autoindex module (after removing any specific configurations)
        #LoadModule autoindex_module modules/mod_autoindex.so
        ```
        ```
        # This is used for CGI Scripts
        #LoadModule env_module modules/mod_env.so
        ```
        ```
        # This is used for Server Side Includes
        #LoadModule include_module modules/mod_include.so
        
        # CIS 2.8 Disable Info module (after removing any specific configurations)
        #LoadModule info_module modules/mod_info.so
        ```
        ```
        # This is used mostly for proxy configs
        #LoadModule remoteip_module modules/mod_remoteip.so
        ```
        ```
        # CIS 2.4 Disable Status modules (after removing any specific configuration)
        #LoadModule status_module modules/mod_status.so
        ```
        ```
        # This is used for CGI Scripts
        #LoadModule suexec_module modules/mod_suexec.so
        ```
        ```
        # CIS 2.7 Disable User Directories module
        #LoadModule userdir_module modules/mod_userdir.so
        ```

    - 00-mpm.conf
         Leave this one as-is, or comment out the MPM module you are not using.  

## Notes

- This installation uses port 8080. You may change this to any non-root based port and you will never need to specify root level access on any folders, nor will you need to run apache as root.  The CIS specifications recommend root ownership of certain files, but this is not needed when apache is not run on port 80.  Using an external facing load balancer as a reverse proxy, placed in front of your apache node(s), you will not need to install apache or run apache as root.  This is safer, as the apache admin role can be separate from the system admin role.  In addition, the reverse proxy can offload SSL, speeding up the apache performance.  
- This installation removes a lot of basic features, such as language-based error pages, cgi-bin, and auto-indexing.  Some of these features do not need to be turned off to meet CIS specifications, but they are done here to simplify the server as much as possible.  Add back in what you may need and be sure to review any CIS specifications for the file permissions where needed (e.g. cgi-bin location and files).  We assume that most companies would want to customize these pages anyways, and do it in some proprietary way, probably without the use of Server Side Includes.  If you want that behavior just turn it back on and note that SSI is old technology.
- CGI-BIN was disabled.  Again, we assume that most web-sites will not be implementing this older technology.
- Authenitcation modules were disabled under the assumption that some other mechanism would be responsible for handling that, where needed.  Enable them if you need them.
- Apache works great as a reverse proxy too.  That is not covered here, and all proxy modules have been disabled.

