# Introduction: 			
This documentation was developed and tested using EJBCA Branch 7_4 _3 _x on Wildfly 14.0.0 with MariaDB 2.2.6 as the database engine . Everything in this document is current as of August 4, 2021.


## Wildfly Installation:
###### Download the wildlfy verison 14:

```
wget https://download.jboss.org/wildfly/14.0.0.Final/wildfly-14.0.0.Final.zip 
```

Configure Wildfly by following the steps on this [site:](https://linuxtechlab.com/wildfly-10-10-1-0-installation/)

Since, we have already downloaded and unzipped the file so skip step 1. 
Starting from step 2, we will replace the IP Addresses with the one’s IP in the file and add the paths:
```
cd wildfly/standalone/configuration/
nano standalone.xml
```

In these 3 places, replace the IP Addresses with the system’s IP:

```
<subsystem xmlns="urn:jboss:domain:webservices:2.0">
<wsdl-host>${jboss.bind.address:192.168.1.100}</wsdl-host>
<endpoint-config name="Standard-Endpoint-Config"/>

<interface name="management">
<inet-address value="${jboss.bind.address.management:192.168.1.100}"/>
</interface>

<interface name="public">
<inet-address value="${jboss.bind.address:192.168.1.100}"/>
</interface>
```

After changing IPs, open Standalone.conf file by:

```
cd wildfly/bin/
nano standalone.conf
```

In this file, provide the path of wildfly and jdk:
```
JAVA_HOME="/home/jdk1.8.0_73" 
JBOSS_HOME="/home/wildfly"
```
Save the both files and exit. 
To check is wildfly is running or not we will run the script named ‘standalone.sh’, write:
```
cd wildfly/bin/  
./standalone.sh -b 0.0.0.0
```
## MariaDB:
Download MariaDB by:
```
wget https://downloads.mariadb.com/Connectors/java/connector-java-2.2.6/mariadb-java-client-2.2.6.jar 
unzip <filename>
```
## EJBCA Installation: 

Open the terminal (ctrl+alt+T) and go to root by using:
```
$ sudo su 
```
Download EJBCA zip [file](https://github.com/primekeydevs/ejbca-ce/tree/Branch_7_4_3_x)

or through terminal by using:
```
wget https://github.com/primekeydevs/ejbca-ce/tree/Branch_7_4_3_x
unzip filename
```

Now using the installation guide [link](https://github.com/dilucide/ejbca-install/blob/master/EJBCA%20Install%20Ubuntu.md) we will install the ejbca. We will run the following commands in sequence: 
```
sudo apt update
sudo apt upgrade    
```

###### Install Java Framework through:
```
sudo apt install openjdk-8-jdk openjdk-8-demo openjdk-8-doc openjdk-8-jre-headless openjdk-8-source 
```

###### Install ant to build JAVA applications:
```
sudo apt install ant
```

###### Install MariaDB:
```
sudo apt install mariadb-server
```

To set the password on server run:
```
sudo mariadb -u root
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' IDENTIFIED BY '<writeYourPasswordHere >';
FLUSH PRIVILEGES;
```

Now, we will create a database:
```
mysql -u root –p
```

Delete database (if any) by and create new one:
```
drop database <ejbca(db name)>;
CREATE DATABASE <dbname- ejbca> CHARACTER SET utf8 COLLATE utf8_general_ci;
GRANT ALL PRIVILEGES ON ejbca.* TO 'ejbca'@'localhost' IDENTIFIED BY 'ejbca';
exit
```
###### Configuring EJBCA:

We will configure it by making changes to the files: first we will copy these files:

```
cd /conf/
cp database.properties.sample database.properties
cp install.properties.sample install.properties
cp web.properties.sample web.properties
cp ejbca.properties.sample ejbca.properties
```

We will make changes to them by using the installation guide link which is provided above.


Using the given [link](https://doc.primekey.com/ejbca740/ejbca-installation/application-servers/wildfly-14-jboss-eap-7-2#WildFly14/JBossEAP7.2-AddDatabaseDriver), we will configure the application server by performing following steps: 
First we will deploy the java file to wildfly:

```
cp mariadb-java-client-2.2.6.jar /home/local/wildfly/standalone/deployments/mariadb-java-client.jar
```
**make sure to run wildfly before deploying to file, otherwise ‘cannot start’ error will be arises**
In new terminal login as a root and write:

```
wildfly/bin/jboss_cli.sh –c
/socket-binding-group=standard-sockets/socket-binding=remoting:add(port="4447")
/subsystem=undertow/server=default-server/http-listener=remoting:add(socket-binding=remoting)
:reload
```

Now we will add the data source: 
(***in user-name and password write your Database username and password***)

```
data-source add --name=ejbcads --driver-name="mariadb-java-client.jar" --connection-url="jdbc:mysql://127.0.0.1:3306/ejbca" --jndi-name="java:/EjbcaDS" --use-ccm=true --driver-class="org.mariadb.jdbc.Driver" --user-name="ejbca" --password="ejbca" --validate-on-match=true --background-validation=false --prepared-statements-cache-size=50 --share-prepared-statements=true --min-pool-size=5 --max-pool-size=150 --pool-prefill=true --transaction-isolation=TRANSACTION_READ_COMMITTED --check-valid-connection-sql="select 1;"
:reload
```
###### Configuring Wildfly:

```
wildfly/bin/jboss_cli.sh –c
/subsystem=remoting/http-connector=http-remoting-connector:write-attribute(name=connector-ref,value=remoting)
/socket-binding-group=standard-sockets/socket-binding=remoting:add(port=4447,interface=management)
/subsystem=undertow/server=default-server/http-listener=remoting:add(socket-binding=remoting,enable-http2=true)
/subsystem=infinispan/cache-container=ejb:remove()
/subsystem=infinispan/cache-container=server:remove()
/subsystem=infinispan/cache-container=web:remove()
/subsystem=ejb3/cache=distributable:remove()
/subsystem=ejb3/passivation-store=infinispan:remove()	
:reload
```

###### Configure Logging:

```
/subsystem=logging/logger=org.ejbca:add(level=INFO)
/subsystem=logging/logger=org.cesecore:add(level=INFO)
```

```
/subsystem=logging/logger=org.ejbca:write-attribute(name=level, value=DEBUG)
/subsystem=logging/logger=org.cesecore:write-attribute(name=level, value=DEBUG)
```

###### Remove Existing TLS and HTTP Configuration

```
$ wildfly/bin/jboss-cli.sh --connect
/subsystem=undertow/server=default-server/http-listener=default:remove()
/subsystem=undertow/server=default-server/https-listener=https:remove()
/socket-binding-group=standard-sockets/socket-binding=http:remove()
/socket-binding-group=standard-sockets/socket-binding=https:remove()
:reload
```

###### Add New Interfaces and Sockets
In ‘inet-address’ paste your System IP

```
/interface=http:add(inet-address="0.0.0.0")    
/interface=httpspub:add(inet-address="0.0.0.0")
/interface=httpspriv:add(inet-address="0.0.0.0")
/socket-binding-group=standard-sockets/socket-binding=http:add(port="8080",interface="http")
/socket-binding-group=standard-sockets/socket-binding=httpspub:add(port="8442",interface="httpspub")
/socket-binding-group=standard-sockets/socket-binding=httpspriv:add(port="8443",interface="httpspriv")
```

###### Configure TLS:

```
/subsystem=elytron/key-store=httpsKS:add(path="keystore/keystore.jks",relative-to=jboss.server.config.dir,credential-reference={clear-text="serverpwd"},type=JKS)
/subsystem=elytron/key-store=httpsTS:add(path="keystore/truststore.jks",relative-to=jboss.server.config.dir,credential-reference={clear-text="changeit"},type=JKS)
/subsystem=elytron/key-manager=httpsKM:add(key-store=httpsKS,algorithm="SunX509",credential-reference={clear-text="serverpwd"})
/subsystem=elytron/trust-manager=httpsTM:add(key-store=httpsTS)
/subsystem=elytron/server-ssl-context=httpspub:add(key-manager=httpsKM,protocols=["TLSv1.2"])
/subsystem=elytron/server-ssl-context=httpspriv:add(key-manager=httpsKM,protocols=["TLSv1.2"],trust-manager=httpsTM,need-client-auth=true,authentication-optional=false,want-client-auth=true)
```

###### Add HTTP(S) Listeners

```
/subsystem=undertow/server=default-server/http-listener=http:add(socket-binding="http", redirect-socket="httpspriv")
/subsystem=undertow/server=default-server/https-listener=httpspub:add(socket-binding="httpspub", ssl-context="httpspub", max-parameters=2048)
/subsystem=undertow/server=default-server/https-listener=httpspriv:add(socket-binding="httpspriv", ssl-context="httpspriv", max-parameters=2048)
:reload
```

###### HTTP Protocol Behavior Configuration 

```
/system-property=org.apache.catalina.connector.URI_ENCODING:add(value="UTF-8")
/system-property=org.apache.catalina.connector.USE_BODY_ENCODING_FOR_QUERY_STRING:add(value=true)
/system-property=org.apache.tomcat.util.buf.UDecoder.ALLOW_ENCODED_SLASH:add(value=true)
/system-property=org.apache.tomcat.util.http.Parameters.MAX_COUNT:add(value=2048)
/system-property=org.apache.catalina.connector.CoyoteAdapter.ALLOW_BACKSLASH:add(value=true)
/subsystem=webservices:write-attribute(name=wsdl-host, value=jbossws.undefined.host)
/subsystem=webservices:write-attribute(name=modify-wsdl-address, value=true)
:reload
```

Now go to the EJBCA folder through terminal and run the following commands:

```
ant -q clean deployear
```

```
ant runinstall
```

```
ant deploy-keystore
```
***keep the wildfly running at the back***

Open a browser and goto settings:
•	search certificate
•	select view certificate
•	if any certificate is already imported delete it, and click on import
•	goto folder ejbca> p12> select the certificate name ‘SuperAdmin’.
•	Click OK

Now open a new tab and write:

```
https://<SystemIP>:8443/ejbca/adminweb  
```
