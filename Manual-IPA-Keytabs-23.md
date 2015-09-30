## IPA Kerberos Configuration with Ambari wizard

- Goal
    - Create and Setup IPA Keytabs for HDP
    
- Assumptions
    - Ambari does NOT currently (2.1.x) support the automatic generation of keytabs against IPA
    - IPA Server is already installed and IPA clients have been installed/configured on all HDP cluster nodes

    - If you are using Accumulo, you need to create the user 'tracer' in IPA. A keytab for this user will be requested for Accumulo Tracer.
    
## Enable kerberos using wizard

- In Ambari, start security wizard by clicking Admin -> Kerberos and click Enable Kerberos. Then select "Manage Kerberos principals and key tabs manually" option
![Image](../master/screenshots/2.3-ipa-kerb-1.png?raw=true)

- Enter your realm
![Image](../master/screenshots/2.3-ipa-kerb-2.png?raw=true)

- Remove clustername from smoke/hdfs principals to remove the `-${cluster_name}` references to look like below
  - smoke user principal: ${cluster-env/smokeuser}@${realm}
  - HDFS user principal: ${hadoop-env/hdfs_user}@${realm}
  - HBase user principal: ${hbase-env/hbase_user}@${realm}
  - Spark user principal: ${spark-env/spark_user}@${realm}
  - Accumulo user principal: ${accumulo-env/accumulo_user}@${realm}
  - Accumulo Tracer User: tracer@${realm}

![Image](../master/screenshots/2.3-ipa-kerb-3.png?raw=true)

- On next page download csv file but **DO NOT** click Next yet
![Image](../master/screenshots/2.3-ipa-kerb-4.png?raw=true)

  - If you are deploying storm, the storm user maybe missing from the storm USER row. If you see something like the below:
```
storm@HORTONWORKS.COM,USER,,/etc
```  
replace the `,,` with `,storm,`
```
storm@HORTONWORKS.COM,USER,storm,/etc
```  

-  Copy above csv to the Ambari Server and place it in the `/var/lib/ambari-server/resources` directory, making sure to remove empty lines at the end.
```
vi kerberos.csv
```

- **On the IPA node** Create principals using csv file

```
## authenticate
kinit admin
```

```
AMBARI_HOST=<ambari_host>
# Get the kerberos.csv file from Ambari
wget http://${AMBARI_HOST}:8080/resources/kerberos.csv -O /tmp/kerberos.csv
# Create IPA service entries.
awk -F"," '/SERVICE/ {print "ipa service-add --force "$3}' /tmp/kerberos.csv | sort -u > ipa-add-spn.sh
sh ipa-add-spn.sh

# Do NOT create users for services if they were create in IPA already.
awk -F"," '/USER/ {print "ipa user-add "$5" --first="$5" --last=Hadoop --shell=/sbin/nologin"}' /tmp/kerberos.csv | sort | uniq > ipa-add-upn.sh
sh ipa-add-upn.sh
```

- **On Ambari node** authenticate and create the keytabs for the headless user accounts

```
## authenticate
sudo echo '<kerberos_password>' | kinit --password-file=STDIN admin
## or (IPA 4)
sudo echo '<kerberos_password>' | kinit -X password-file=STDIN admin
```

```
AMBARI_HOST=<ambari_host>
wget http://${AMBARI_HOST}:8080/resources/kerberos.csv -O /tmp/kerberos.csv
ipa_server=$(grep host /etc/ipa/default.conf |  awk -F= '{print $2}')
sudo mkdir /etc/security/keytabs/
sudo chown root:hadoop /etc/security/keytabs/
grep USER /tmp/kerberos.csv | awk -F"," '{print "ipa-getkeytab -s '${ipa_server}' -p "$3" -k "$6";chown "$7":"$9,$6";chmod "$11,$6}' | sort -u > gen_keytabs_user.sh
sudo bash ./gen_keytabs_user.sh
```

```
# tar up the contents of the headless keytabs (use Ambari's Resources for distribution)
tar cvfz /var/lib/ambari-server/resources/headless.keytabs.tgz -C /etc/security keytabs
```

- **Use pdsh to distribute and expand** headless keytabs.
```
pdsh -g <cluster_host_group> 'wget http://<ambari-server>:8080/resources/headless.keytabs.tgz -O /tmp/headless.keytabs.tgz;cd /etc/security;tar xvfz /tmp/headless.keytabs.tgz'
```

- **Build a distribution script** used to create host specific keytabs.

```
vi build_keytabs.sh
```

```
AMBARI_HOST=<ambari_host>
wget http://${AMBARI_HOST}:8080/resources/kerberos.csv -O /tmp/kerberos.csv
ipa_server=$(grep server /etc/ipa/default.conf |  awk -F= '{print $2}')
if [ "$ipa_serverX" == "X" ]; then
    ipa_server=$(grep host /etc/ipa/default.conf |  awk -F= '{print $2}')
fi
if [ ! -d /etc/security/keytabs ]; then
    mkdir -p /etc/security/keytabs
fi
chown root:hadoop /etc/security/keytabs/
grep SERVICE /tmp/kerberos.csv | awk -F"," '/'$(hostname -f)'/ {print "ipa-getkeytab -s '$(echo $ipa_server)' -p "$3" -k "$6";chown "$7":"$9,$6";chmod "$11,$6}' | sort -u > gen_keytabs_service.sh
bash ./gen_keytabs_service.sh
```

Copy file to the Ambari Servers resource directory for distribution.

```
scp build_keytabs.sh root@<ambari_host>:/var/lib/ambari-server/resources
```

- **On the Each HDP node via 'pdsh'** authenticate and create the keytabs

```
# Should be logging in as 'root'
pdsh -g <host_group> -l root

> ## authenticate
> echo '<kerberos_password>' | kinit --password-file=STDIN admin
> wget http://<ambari_host>:8080/resources/build_keytabs.sh -O /tmp/build_keytabs.sh
> bash /tmp/build_keytabs.sh
> ## Verify kinit works before proceeding (should not give errors)
> # Service Account Check
> sudo -u hdfs kinit -kt /etc/security/keytabs/nn.service.keytab nn/$(hostname -f)@HORTONWORKS.COM
> # Headless Check (check on multiple hosts)
> sudo -u ambari-qa kinit -kt /etc/security/keytabs/smokeuser.headless.keytab ambari-qa@HORTONWORKS.COM
> sudo -u hdfs kinit -kt /etc/security/keytabs/hdfs.headless.keytab hdfs@HORTONWORKS.COM
```

# WARNING: IPA UI is Broken after above procedure

When the process to build keytabs for services is run on the same host that IPA lives on, it will invalidate the keytab used by Apache HTTPD to authenticate.

Replace /etc/httpd/conf/ipa.keytab with /etc/security/keytabs/spnego.service.keytab

```
cd /etc/httpd/conf
mv ipa.keytab ipa.keytab.orig
cp /etc/security/keytabs/spnego.service.keytab ipa.keytab
chown apache:apache ipa.keytab
service httpd restart
```

### Remove the headless.keytabs.tgz file from /var/lib/ambari-server/resources on the Ambari-Server.

- Press next on security wizard and proceed to stop services
![Image](../master/screenshots/2.3-ipa-kerb-stop.png?raw=true)

- Proceed with the next steps of wizard by clicking Next
![Image](../master/screenshots/Ambari-kerborize-cluster.png?raw=true)
![Image](../master/screenshots/Ambari-start-services.png?raw=true)

- Once completed, click Complete and now the cluster is kerborized
![Image](../master/screenshots/Ambari-wizard-completed.png?raw=true)

-------

## Using your Kerberized cluster

0. Try to run commands without authenticating to kerberos.
  ```
$ hadoop fs -ls /
15/07/15 14:32:05 WARN ipc.Client: Exception encountered while connecting to the server : javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
  ```

  ```
$ curl -u someuser -skL "http://$(hostname -f):50070/webhdfs/v1/user/?op=LISTSTATUS"
<title>Error 401 Authentication required</title>
  ```


1. Get a token
  ```
## for the current user
sudo su - gooduser
kinit

## for any other user
kinit someuser
  ```

2. Use the cluster

* Hadoop Commands
  ```
$ hadoop fs -ls /
Found 8 items
[...]
  ```
  
* WebHDFS
  ```
## note the addition of `--negotiate -u : `
curl -skL --negotiate -u : "http://$(hostname -f):50070/webhdfs/v1/user/?op=LISTSTATUS"
  ```

* Hive *(using Beeline or another Hive JDBC client)*

  * Hive in Binary mode *(the default)*
   ```
beeline -u "jdbc:hive2://localhost:10000/default;principal=hive/$(hostname -f)@HORTONWORKS.COM"
   ```

  * Hive in HTTP mode
  ```
## note the update to use HTTP and the need to provide the kerberos principal.
beeline -u "jdbc:hive2://localhost:10001/default;transportMode=http;httpPath=cliservice;principal=HTTP/$(hostname -f)@HORTONWORKS.COM"
  ```

