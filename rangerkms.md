# Rangerkms

### CDP

```
# Create encryption zones

export NAMENODE_PROCESS_DIR=$(ls -1dtr /var/run/cloudera-scm-agent/process/*hdfs-NAMENODE| tail -1)
kinit -kt $NAMENODE_PROCESS_DIR/hdfs.keytab hdfs/`hostname -f`
hadoop key list
hdfs dfs -mkdir /enczone1
hdfs crypto -createZone -keyName key1 -path /enczone1
hdfs crypto -listZones 

echo "Hello TDE" >> myfile.txt
hadoop dfs -put myfile.txt /enczone1
hdfs dfs -chmod 700 /enczone1



$ hdfs dfs -cat /enczone1/myfile.txt

==> access_log2021-06-04.07.log <==
172.27.133.6 - - [04/Jun/2021:07:54:16 +0000] "OPTIONS /kms/v1/keyversion/key1%400/_eek" 200 644
172.27.133.6 - - [04/Jun/2021:07:54:16 +0000] "POST /kms/v1/keyversion/key1%400/_eek" 200 86
172.27.187.13 - - [04/Jun/2021:07:54:17 +0000] "GET /kms/api/status" 200 53


# List a Directory
https://hadoop.apache.org/docs/r1.0.4/webhdfs.html#LISTSTATUS
curl -ik --negotiate -u :  "https://pbhagade-3.pbhagade.root.hwx.site:20102/webhdfs/v1/enczone1?op=LISTSTATUS"

# Open and Read a File
curl -i -L "http://<HOST>:<PORT>/webhdfs/v1/<PATH>?op=OPEN
                    [&offset=<LONG>][&length=<LONG>][&buffersize=<INT>]"

curl -ik --negotiate -u :  "https://pbhagade-3.pbhagade.root.hwx.site:20102/webhdfs/v1/enczone1/myfile.txt?op=OPEN"                    

# Location: https://pbhagade-2.pbhagade.root.hwx.site:9865/webhdfs/v1/enczone1/myfile.txt?op=OPEN&delegation=IAAGcHJhdmluBnByYXZpbgCKAXnWDA4OigF5-hiSDh0UFFJCJRjYyffNeFSLHbJN_9cHVGuEE1NXRUJIREZTIGRlbGVnYXRpb24SMTcyLjI3LjE4Ny4xMzo4MDIw&namenoderpcaddress=nameservice1&offset=0

++++++++++++
WWW-Authenticate: Negotiate YGwGCSqGSIb3EgECAgIAb10wW6ADAgEFoQMCAQ+iTzBNoAMCARCiRgRENLnAJ54II818HzDSehFSlaqiX5UTcVURW8ae2aJa/MG4bqdfmpZ3ezexxVgAtdfrQ0u45TH7ZsPapqB9lXxNcP47w+I=
Set-Cookie: hadoop.auth="u=pravin&p=pravin@ROOT.HWX.SITE&t=kerberos&e=1622829784843&s=cWmqgcx4kqUVLgGZtgK7P6/On0ByINf8KwJ4l+hc1sg="; Path=/; Secure; HttpOnly
Location: https://pbhagade-2.pbhagade.root.hwx.site:9865/webhdfs/v1/enczone1/myfile.txt?op=OPEN&delegation=IAAGcHJhdmluBnByYXZpbgCKAXnWDA4OigF5-hiSDh0UFFJCJRjYyffNeFSLHbJN_9cHVGuEE1NXRUJIREZTIGRlbGVnYXRpb24SMTcyLjI3LjE4Ny4xMzo4MDIw&namenoderpcaddress=nameservice1&offset=0
Content-Type: application/octet-stream
Content-Length: 0

[pravin@pbhagade-2 318-hdfs-NAMENODE]$ curl -ik --negotiate -u :  "https://pbhagade-2.pbhagade.root.hwx.site:9865/webhdfs/v1/enczone1/myfile.txt?op=OPEN&delegation=IAAGcHJhdmluBnByYXZpbgCKAXnWDA4OigF5-hiSDh0UFFJCJRjYyffNeFSLHbJN_9cHVGuEE1NXRUJIREZTIGRlbGVnYXRpb24SMTcyLjI3LjE4Ny4xMzo4MDIw&namenoderpcaddress=nameservice1&offset=0"
HTTP/1.1 403 Forbidden
Content-Type: application/json; charset=utf-8
Content-Length: 200
Connection: close

{"RemoteException":{"exception":"AuthorizationException","javaClassName":"org.apache.hadoop.security.authorize.AuthorizationException","message":"User:hdfs not allowed to do 'DECRYPT_EEK' on 'key1'"}}
++++++++++++++++

```

##### using knox
```

[root@pbhagade-2 ~]# curl -i -k -u pravin:Welcome -L -X GET "https://pbhagade-1.pbhagade.root.hwx.site:8443/gateway/cdp-proxy-api/webhdfs/v1/enczone1/myfile.txt?op=OPEN"
HTTP/1.1 307 Temporary Redirect
Date: Fri, 04 Jun 2021 08:58:23 GMT
Set-Cookie: KNOXSESSIONID=node01xjcgkvgrcnj41phhtrhopvijv6.node0; Path=/gateway/cdp-proxy-api; Secure; HttpOnly
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Set-Cookie: rememberMe=deleteMe; Path=/gateway/cdp-proxy-api; Max-Age=0; Expires=Thu, 03-Jun-2021 08:58:26 GMT; SameSite=lax
Date: Fri, 04 Jun 2021 08:58:26 GMT
Cache-Control: no-cache
Expires: Fri, 04 Jun 2021 08:58:26 GMT
Date: Fri, 04 Jun 2021 08:58:26 GMT
Pragma: no-cache
X-Content-Type-Options: nosniff
X-FRAME-OPTIONS: SAMEORIGIN
X-XSS-Protection: 1; mode=block
Location: https://pbhagade-1.pbhagade.root.hwx.site:8443/gateway/cdp-proxy-api/webhdfs/data/v1/webhdfs/v1/enczone1/myfile.txt?_=AAAACAAAABAAAAEAbF5ZpS5ynU0G49zfU4Iqg2yA7qy_dVIMi0C7tLYzLBxJ89uVL8_m5iLYuJyoJRS9kThMXoH_kdWI8yengOXGBLfBtVnUn2lHFsVNrpqOZBUNKcexiDSXFza6RG1xXdtpfFU0lBJb2OrOAdbltymyGUHqQQKKn5hvIgq7LCrrHoOrNKK3DS4ojsHRdDFUQaLEENSxFBoceDNeG3D9wU9wDTZndHu71FFpKmCZaWEnPbvnZfJufW2b5aAbVcKG_EabS-L1HY6_T6EQQu4_GVKfZ8bv57L2rZWapxOpLZLlD0IGtPBxJBXx2khFKhD_o9qUHqw2afoGYc1d6C_wv5iLuRvANFSKv0Pb3qMkp0a1QHXOPxL2uf-GbA
Content-Type: application/octet-stream
Content-Length: 0

HTTP/1.1 200 OK
Date: Fri, 04 Jun 2021 08:58:26 GMT
Set-Cookie: KNOXSESSIONID=node0smxh8z2pvdut1osd0gwpftudo7.node0; Path=/gateway/cdp-proxy-api; Secure; HttpOnly
Expires: Thu, 01 Jan 1970 00:00:00 GMT
Set-Cookie: rememberMe=deleteMe; Path=/gateway/cdp-proxy-api; Max-Age=0; Expires=Thu, 03-Jun-2021 08:58:28 GMT; SameSite=lax
Access-Control-Allow-Methods: GET
Access-Control-Allow-Origin: *
Content-Type: application/octet-stream
Connection: close
Content-Length: 10

Hello TDE
[root@pbhagade-2 ~]#
```
