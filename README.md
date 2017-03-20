# Shibboleth setup with nginx on ubuntu
## Overview
I have spent quite a lot of time and efforts online to find a complete solution for setup **Shibboleth**, one SAML based open source tool for setup identitfy federation infrastructure. However, it is widely used in enterprise federation infrastructure but somehow, not so popular or even rarely heard in web technology communities comparing to **OAuth2**. As a result, except the  [official wiki](https://wiki.shibboleth.net/confluence) that contains comprehensive information but overwhelms developers by plenty of concepts and clueless instructions, I can hardly find updated developer-friendly guide on setup shibboleth, especially when your tech stacks contain nginx on ubuntu. The official wiki provides rich information on how to setup shibboleth on CentOS and RedHat but not mention much about other platform, probably becuase somehow, Ubuntu is considered to be fancy and favored by developers like a toy, while not serious enough for professional system administrators to rely on and setup advanced security functionality, such as enterprise identity federation. But anyway, this guide is for those who have the needs like me but yet figure out a workable solution.

**Noted:** This guide will only cover the topic of setup Shibboleth Server provider with nginx on ubuntu. For id provider and discovery service setup, that is much more complicated and also rarely needed for small organisations and individuals. Leave those to the tech team of large enterprise and we only need to focus how to consume the identity information as a service provider.

## Basic dependencies and versions
1. **Ubuntu -** version: 16.04.1
2. **shibboleth-sp -** this is the service integrated with HTTP server in order to provide Shibboleth authentication method to web applications. We're going to use it together with Nginx. version: 2.5.3
3. **nginx -** version: 1.10.2
4. **nginx-http-shibboleth -** from official wiki [here](https://wiki.shibboleth.net/confluence/display/SHIB2/Integrating+Nginx+and+a+Shibboleth+SP+with+FastCGI), to integrate shibboleth service provider with nginx, we have to use the FastCGI support of nginx and shibboleth to connect them and this is the official package used to do the work. version: 2.0.0(latest)
5. **Supervisor -** utility used to integrate Shibboleth SP with Nginx through fast-cgi 

**Noted:** I have tried to setup on both Ubuntu 16.04.1 and 14.04.1. The problem for 14.04.1 is that some of the dependencies for shiboleth cannot be found from the default apt repositories on 14.04.1 and to make it work, manually updating the source list of apt is required and it causes further dependencies version conflicts if you do not manage it carefully. So my suggestion is to use 16.04.1 for Shibboleth setup purpose. 

## Guide
### Install Shibboleth SP with fast-cgi support

First,  we need to download and install Shibboleth SP packages manually in the following order:

1. [libmemcached11]
2. [libodbc1]
3. [shibboleth-sp2-common]
4. [libshibsp6]
5. [libshibsp-plugins]
6. [shibboleth-sp2-utils]

[libmemcached11]: https://packages.debian.org/sid/libmemcached11
[libodbc1]: https://packages.debian.org/sid/libodbc1
[shibboleth-sp2-common]: https://packages.debian.org/sid/shibboleth-sp2-common
[libshibsp6]: https://packages.debian.org/sid/libshibsp6
[libshibsp-plugins]: https://packages.debian.org/sid/libshibsp-plugins
[shibboleth-sp2-utils]: https://packages.debian.org/sid/shibboleth-sp2-utils

Simply use `sudo apt-get install` to install them in order and you are good to go.

In the end we should have:

a) `/etc/shibboleth/` directory that contains Shibboleth SP configuration files.

b) shibd deamon which can be started using `sudo systemctl restart shibd.service`.

c) `/usr/lib/x86_64-linux-gnu/shibboleth/` directory which contains 'shibauthorizer' and 'shibresponder'. Those are fast-cgi executables required for nginx integration.

If one of the above is missing it means that something went wrong and you may double check if those dependencies have been installed correctly.

### Install and configure Supervisor

Install supervisor:

```
sudo apt-get install supervisor
```

Create a configuration file for supervisor:

```
sudo touch /etc/supervisor/conf.d/shib.conf
```

Edit `/etc/supervisor/conf.d/shib.conf` file:

```
[fcgi-program:shibauthorizer]
command=/usr/lib/x86_64-linux-gnu/shibboleth/shibauthorizer
socket=unix:///etc/shibboleth/shibauthorizer.sock
socket_owner=_shibd:_shibd
socket_mode=0666
user=_shibd
stdout_logfile=/var/log/supervisor/shibauthorizer.log
stderr_logfile=/var/log/supervisor/shibauthorizer.error.log

[fcgi-program:shibresponder]
command=/usr/lib/x86_64-linux-gnu/shibboleth/shibresponder
socket=unix:///etc/shibboleth/shibresponder.sock
socket_owner=_shibd:_shibd
socket_mode=0666
user=_shibd
stdout_logfile=/var/log/supervisor/shibresponder.log
stderr_logfile=/var/log/supervisor/shibresponder.error.log
```

**Noted:** The socket locations `unix:///etc/shibboleth/shibauthorizer.sock` and `unix:///etc/shibboleth/shibresponder.sock` are for Ubuntu specifically. For Debain system, you can replace `etc` with `opt`. Moreover, in some guides online for configuring supervisor for shibboleth, they use `socket_owner=shibd:shibd` for authorizing shibd deamon. However, it does not work in my case and `socket_owner=_shibd:_shibd` is the correct one.

Then Restart Supervisor:

```
sudo service supervisor restart
```
After this, you should find two sockets created at:

```
unix:///etc/shibboleth/shibauthorizer.sock
unix:///etc/shibboleth/shibresponder.sock
```
with owner as **_shibd**.

Optionally, you can check the log files located by the configurations above and there should not no error log if everything works well.

### Build Nginx from sources with fast-cgi

Nginx is required to be built(or rebuilt if you have installed it already) from source with modules `nginx-http-shibboleth` and `headers-more`. The latest version of nginx seems to support dynamic module loading already. But in this guide, I will also show the process of statically building nginx with additional modules.

Download 'nginx-http-shibboleth' external module:

```
git clone https://github.com/nginx-shib/nginx-http-shibboleth
```

Download and unzip 'headers-more' external module(latest version is 0.32 by the time this guide created):

```
wget https://github.com/openresty/headers-more-nginx-module/archive/v0.32.zip
unzip v0.32.zip
```

Download and build Nginx:

```
wget http://nginx.org/download/nginx-1.10.2.tar.gz
tar -xzvf nginx-1.10.2.tar.gz
```

Build nginx with additional modules:

```
cd nginx-1.10.2
```
```
./configure --sbin-path=/usr/sbin/nginx \
 --conf-path=/etc/nginx/nginx.conf \
 --pid-path=/run/nginx.pid \
 --error-log-path=/var/log/nginx/error.log \
 --http-log-path=/var/log/nginx/access.log \
 --with-http_ssl_module \
 --with-ipv6 \
 --add-module=/{modules location}/nginx-http-shibboleth \
 --add-module=/{modules location}/headers-more-nginx-module-0.32
```
``` 
make
```
```
sudo make install
```
**Remember** to replace {modules location} above to the real locations of the modules. This process will take a few minutes and you can do something else at the time :)

###Configure Nginx

After finishing above steps, you may create a default site conf at ` /etc/nginx/sites-available/` or you can use your own nginx conf file name. The configuration content will be something as below:

```
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    
    ...

	# Shibboleth
	
	location = /shibauthorizer {
	  internal;
	  include fastcgi_params;
	  fastcgi_pass unix:/opt/shibboleth/shibauthorizer.sock;
	}
	
	location /Shibboleth.sso {
	  include fastcgi_params;
	  fastcgi_pass unix:/opt/shibboleth/shibresponder.sock;
	}
	
	location /shibboleth-sp {
	  alias /usr/share/shibboleth/;
	}
		
	# Login location configuration
		
	location /login {
	  include {nginx-http-shibboleth module location}/includes/shib_clear_headers;
	  more_clear_input_headers 'Variable-*' 'Shib-*' 'Remote-User' 'REMOTE_USER' 'Auth-Type' 'AUTH_TYPE';
	  more_clear_input_headers 'displayName' 'mail' 'persistent-id';
	  shib_request /shibauthorizer;
	  proxy_pass http://127.0.0.1:8888;
	}
}
```

The part under ```# Shibboleth``` is required by shibboleth itself and you can just keep it there without any change. Other part is up to your nginx configuration and this is out of the scope of the guide, so I will not discuss it any more. Remember to add:

```
include {nginx-http-shibboleth module location}/includes/shib_clear_headers;
more_clear_input_headers 'Variable-*' 'Shib-*' 'Remote-User' 'REMOTE_USER' 'Auth-Type' 'AUTH_TYPE';
more_clear_input_headers 'displayName' 'mail' 'persistent-id';
shib_request /shibauthorizer;
```
for each location block you want to protect by shibboleth.

### Configure Shibboleth SP
This can be a big topic that has been discussed in the official wiki already. Here I will list down some practices that may help you save several times of trial and error and scratch your head.
**Noted:** the configuration below only cover the scenario that there is only one application working as the service provider and only one id provider to talk to. Complicated multiple id providers and virtual hosts setup on one server as multiple service providers is out of the scope of this guide. Please refer to the official wiki.

#### Shibboleth2.xml
This is the core configuration file for shibboleth. Despite from the plenty of complicated configurable points, following are a few you need to care about:

1.```ApplicationDefaults```	

This XML block contains most of the configurations related to your application(service provider). A few critical attributes of it you may need to modify based on your need:

* **entityID** this works as the identity of your service provider on throughout the identity federation environment. Usually, we follow the convention `{full_domain}/shibboleth` for it. Note that this looks like a url end point but it is not. It's just an ID to identify the service provider.
* **homeURL** this is the home url shibd deamon will redirect to after user access has passed shibboleth validation.
* **signing** this is an attribute you probably need to modify from the default "**false**" value. Official wiki provides complete explanation about how to configure [here](https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPSigningEncryption).

2.```Sessions```

This is one xml block included inside `<ApplicationDefaults>`, which controls the session. Three attributes you need to have a look:

* **handlerURL** Usually we use `Shibboleth.sso` for this value, which matches the one we set in one of the nginx location configuration block. You can modify it but just make sure it matches with those in nginx configurations.
* **handlerSSL** Most of the case, we will setup shibboleth with https to enhance the security and in this case you should set this value to be **true**. However, chances exist that people want to just use http, which is not recommended. In this case, you may need to disable this by setting it to **false**.
* **cookieProps** This one controls the allowed connections for cookie. In previous version of shibboleth, there are several value you need to provide for this configuration, but now you can just use set configurations described as "**https**" or "**http**" and shibboleth will do the remaining work. Same as above, "**https**" will restrict cookie to be only used on https secured connection and "**http**" will also allow cookie to be used in any http connection.

3.```SSO```

This is one block inside `<Sessions>`, which states the entityID of id provider as well as allowed SAML protocol version. Following is an example:
	
	```
    <SSO entityID="IDProvider">
      SAML2 SAML1
    </SSO>
    ```
   SAML2 SAML1 correspond to **SAML2.0** and **SAML1.1** protocols. Here the configuration states that both **SAML2.0** and **SAML1.1** are allowed. One thing you need to be careful here is the entityID. It **must match** with the entityID in the one inside **id provider metadata** you get from your id provider.
   
4.```MetadataProvider```

This block specifies the solution to obtain ip provider metadata, either through secured https request or locally. Easy way to get thing done is to use the local way, which you just need to put the id provider metadata file on your server and update `file` attribute to point to the metadata location.
	
5.```RequestMapper```

This is a necessary configuration in order to map the protection target location to shibboleth so that it know where to protect. This is originally not needed but because we use fastCGI, it is required. Following is an example matching the nginx configuration above:
	
	```
	<RequestMapper type="XML">
	    <RequestMap>
	        <Host name="{host domain}"
	              authType="shibboleth"
	              requireSession="true"
	              redirectToSSL="443">
	            <Path name="/login" />
	        </Host>
	    </RequestMap>
	</RequestMapper>
	```
Note that `redirectToSSL="443"` is a good practice for https only setup. It helps redirect all http access for the target path to https.
	
#### metadata.xml
Not much to be discussed here as the content of metadata highly depends on the specific application requirements. But just note a few points:

1. the certificate used in metadata for securely communication with id provider can be self-signed certificate and it is actually recommended to do so. Shibboleth also provides program to auto generate the certificate and private key and put it at developer desired place.
2. entityID in metadata should match with that in `<ApplicationDefaults>` in **shibboleth2.xml**.
3. For AssertionConsumerService, there are quite a few available options for shibboleth on SAML2.0 and usually the common choice is `urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST`.

### Switch from In-Memory Session Storage to Database (PostgreSQL, MySQL, etc)
Sometimes, due to the increasing incoming load for the application, single server architecture is not enough to carry the load and also not suitable for scaling further. Then you will need to migrate the system into a load balanced server cluster. Specifically for clustering with shibboleth based federation, you will need a centralized end point, like database, to synchronize and persist your session data among servers. This section will guide you through a typical setup using PostgreSQL database (instructions for other database can be found [here](https://wiki.shibboleth.net/confluence/display/SHIB2/NativeSPODBCStorageService)).

#### Database Setup
1. create a new database on your server postgresql scope called `shibboleth-sp`(or any name suitable for your purpose).
2. create three tables with following schema:

```
CREATE TABLE version (
    major int NOT NULL,
    minor int NOT NULL
    );
    
INSERT INTO version VALUES (1,0);

CREATE TABLE strings (
    context varchar(255) NOT NULL,
    id      varchar(255) NOT NULL,
    expires timestamp    NOT NULL,
    version smallint     NOT NULL,
    value   varchar(255) NOT NULL,
    PRIMARY KEY (context, id)
    );
    
CREATE TABLE texts (
    context varchar(255) NOT NULL,
    id      varchar(255) NOT NULL,
    expires timestamp    NOT NULL,
    version smallint     NOT NULL,
    value   text         NOT NULL,
    PRIMARY KEY (context, id)
    ); 
```

3. **NOTE:** be sure that your pg_hba.conf file is setup to allow IPv4 MD5 authentication from the network location of your Shibboleth-SP host:

```
local   all             postgres                      peer
host    all             all              127.0.0.1/32 md5
host    all             all              10.0.0.0/8   md5
```

4. **IMPORTATN STEP:** on ubuntu, you need to install one package:

[odbc-postgresql]

[odbc-postgresql]: https://odbc.postgresql.org

**odbc-postgresql** package is the odbc connector for PostgreSQL on ubuntu, packaged with PostgreSQL driver. You will need this to enable shibboleth to connect to the database. 
After installing, you can find two configurations files created: `/etc/odbcinst.ini` and `/etc/odbc.ini`. Run command `odbcinst -q -d` and you will get the available driver configurations.
Specifically, there are two types of drivers available by default: **PostgreSQL ANSI** and **PostgreSQL Unicode**. The difference is on the encoding format the database will use to store and parse the data. You can further explore the details online but most of the cases we will stick to **PostgreSQL Unicode**.

#### Shibboleth Configuration
After configuring the database, you will need to update shibboleth.xml also.

1. first update this part of the file:
	
```
<OutOfProcess logger="shibd.logger">
	<Extensions>
	  <Library path="odbc-store.so" fatal="true"/>
	</Extensions>
</OutOfProcess>
```
The library odbc-store.so is the one used by shibboleth to connect to database. For some system, this library will be available under your `/etc/shibboleth` folder. However, someone doesn't have this lib available after they install the shibboleth resource. It may locate in a different place (`/usr/lib/...` for mine). In this case, you may have to create a symbolic link to link the odbc-
store.so into folder `/etc/shibboleth` for the ease of your shibboleth configuration.

2. Then added the follow configurations after the `<OutOfProcess>` block:

```
<StorageService type="ODBC" id="db" cleanupInterval="900">
    <ConnectionString><![CDATA[
	Driver=PostgreSQL;Server=127.0.0.1;Port=5432;Database=shibboleth-sp;Uid=shibboleth-sp;Password=shibboleth-sp-password
]]></ConnectionString>
  </StorageService>
  <SessionCache type="StorageService" StorageService="db" cacheAllowance="3600" inprocTimeout="900" cleanupInterval="900" />
  <ReplayCache StorageService="db" />
  <ArtifactMap StorageService="db" artifactTTL="180" />
```

This part provides necessary database information for shibboleth to connect. The **CDATA[...** string is the database connection string that is common setup. We will skip the detailed discussion about it.

3. Last but not least, change the `relayState` from **ss:mem** to **ss:db** in `<Sessions>` block. This will switch the session persistence strategy from memory to database.

After all the configurations are done, ideally you can expect the shibboleth service provider is using database to save the session information now. While if there is any trouble, feel free to check the shibboleth error log and that will provide you detailed enough reasons.

## Conclusion
After all the steps above, you should be able to setup a workable server protected by shibboleth. There are also quite a lot of way to verify the sucess, which you can easily find and I will not discussed here.
