# Apache Loadbalancer Non-SSL

```sh
cd /usr/local
wget https://archive.apache.org/dist/httpd/httpd-2.4.16.tar.gz
wget https://archive.apache.org/dist/apr/apr-1.5.2.tar.gz 
wget https://archive.apache.org/dist/apr/apr-util-1.5.4.tar.gz
tar -xvf httpd-2.4.16.tar.gz
tar -xvf apr-1.5.2.tar.gz 
tar -xvf apr-util-1.5.4.tar.gz
mv apr-1.5.2/ apr
mv apr httpd-2.4.16/srclib/ 
mv apr-util-1.5.4/ apr-util
mv apr-util httpd-2.4.16/srclib/
yum groupinstall "Development Tools" -y
yum install openssl-devel pcre-devel -y 
cd /usr/local/httpd-2.4.16
./configure --enable-so --enable-ssl --with-mpm=prefork --with-included-apr
make && make install
cd /usr/local/apache2/bin
./apachectl start
curl localhost
```

#### Edit the httpd.conf file:

`vi /usr/local/apache2/conf/httpd.conf`
```sh
# add below
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule proxy_ajp_module modules/mod_proxy_ajp.so
LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
LoadModule slotmem_shm_module modules/mod_slotmem_shm.so
LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so
LoadModule lbmethod_bytraffic_module modules/mod_lbmethod_bytraffic.so
LoadModule lbmethod_bybusyness_module modules/mod_lbmethod_bybusyness.so
LoadModule ssl_module modules/mod_ssl.so
```

#### Create a custom conf file for Ranger:
`vi ranger-cluster.conf`

Make the following updates:

Add the following lines, then change the` <VirtualHost *:88>` port to match the default port you set in the `httpd.conf` file in the previous step.

```sh
#Listen 80
<VirtualHost *:80>
        ProxyRequests off
        ProxyPreserveHost on
        #ErrorLog "/var/log/httpd_error_log"
        #CustomLog "/var/log/httpd_access_log" common
	
# Below is configuration for Knox/Ranger, where is SSL enabled, we need have knox cert with us so apache to connect to Knox
#	SSLProxyEngine On
#	SSLVerifyClient optional
#       SSLOptions +ExportCertData
#       SSLProxyVerify none
#       SSLProxyCheckPeerCN off
#       SSLProxyCheckPeerName off
#       SSLProxyCheckPeerExpire off
#       ProxyRequests off
#       ProxyPreserveHost off
	
# Add below bundle of knox cert or any service which SSL enabled in ranger_lb_crt.pem 
        #SSLCACertificateFile /usr/local/apache2/conf/ranger_lb_crt.pem

        Header add Set-Cookie "ROUTEID=.%{BALANCER_WORKER_ROUTE}e; path=/" env=BALANCER_ROUTE_CHANGED

        <Proxy balancer://rangercluster>
                BalancerMember http://<Ranger-Hostname-1>:6080 loadfactor=1 route=1
                BalancerMember http://<Ranger-Hostname-2>:6080 loadfactor=1 route=2

	### Knox Loadbalancing  ###

              # BalancerMember https://<Knox-Hostname-1>:8443 loadfactor=1 route=1
              # BalancerMember https://<Knox-Hostname-2>:8443 loadfactor=1 route=2
               
                Order Deny,Allow
                Deny from none
                Allow from all

                ProxySet lbmethod=byrequests scolonpathdelim=On stickysession=ROUTEID maxattempts=1 failonstatus=500,501,502,503 nofailover=Off
        </Proxy>

        # balancer-manager
        # This tool is built into the mod_proxy_balancer
        # module and will allow you to do some simple
        # modifications to the balanced group via a gui
        # web interface.
        <Location /balancer-manager>
                SetHandler balancer-manager
                Order deny,allow
                Allow from all
        </Location>


       ProxyPass /balancer-manager !
       ProxyPass / balancer://rangercluster/
       ProxyPassReverse / balancer://rangercluster/

</VirtualHost>
```

#### For Knox loadbalancing follow below steps:

```bash
echo -n | openssl s_client -connect <Knox-Hostname-1>:8443 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > /tmp/knoxcert1.crt
echo -n | openssl s_client -connect <Knox-Hostname-2>:8443 | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > /tmp/knoxcert2.crt

cat /tmp/knoxcert1.crt /tmp/knoxcert2.crt > /usr/local/apache2/conf/ranger_lb_crt.pem
```

#### Run the following command to restart the httpd server:
`/usr/local/apache2/bin/apachectl restart`


# For Kerberos setup

```bash
kadmin.local
addprinc -randkey HTTP/<host3>@EXAMPLE.COM
ktadd -norandkey -kt /etc/security/keytabs/spnego.service.keytab HTTP/<host3>@EXAMPLE.COM
ktadd -norandkey -kt /etc/security/keytabs/spnego.service.keytab HTTP/<host2>@EXAMPLE.COM
ktadd -norandkey -kt /etc/security/keytabs/spnego.service.keytab HTTP/<host1>@EXAMPLE.COM
exit
klist -kt /etc/security/keytabs/ranger.ha.keytab
chmod 440 /etc/security/keytabs/ranger.ha.keytab
chown root:hadoop /etc/security/keytabs/ranger.ha.keytab
```
Add config from `Ambari > Ranger > Configs > Advanced > Custom ranger-admin-site`:

`ranger.ha.spnego.kerberos.keytab=/etc/security/keytabs/ranger.ha.keytab`


Ref: https://docs.cloudera.com/HDPDocuments/HDP3/HDP-3.1.4/fault-tolerance/content/configuring_ranger_admin_ha_without_ssl.html


#### curl cmd to test knox
```sh
curl -k -u Username:Password -X GET "https://<LB-Knox Hostname>:lb-port/gateway/default/webhdfs/v1/?op=LISTSTATUS"
```

# ^ Troubleshooting

Login into Ranger node:
```sh
kinit -kt /etc/security/keytabs/rangeradmin.service.keytab $(klist -kt /etc/security/keytabs/rangeradmin.service.keytab |sed -n "4p"|cut -d ' ' -f7)
curl -ik --negotiate -u : "http://c174-node1.squadron.support.hortonworks.com:8888/service/public/v2/api/service"
klist -dfae
```

```
[Wed Nov 27 15:29:40.345322 2019] [proxy:error] [pid 214509] AH00961: HTTPS: failed to enable ssl support for 172.25.40.155:8443 (c174-node4.squadron.support.hortonworks.com)
[Wed Nov 27 15:29:44.109359 2019] [ssl:error] [pid 214510] [remote 172.25.40.155:8443] AH01961: SSL Proxy requested for c174-node1.squadron.support.hortonworks.com:8888 but not enabled [Hint: SSLProxyEngine]
[Wed Nov 27 15:29:44.109396 2019] [proxy:error] [pid 214510] AH00961: HTTPS: failed to enable ssl support for 172.25.40.155:8443 (c174-node4.squadron.support.hortonworks.com)
```
make sure you have `SSLProxyEngine On`
