

# CDH KMS 测试

## 0、用户说明


- [x] keyAdminUser用户是key admin user
- [x] hdfs 用 户是 hdfs super user
- [x] user_a 、 user_b 是HDFS普通用户

##  1、创建keytab
按照下面的办法创建keytab
```
addprinc -randkey ourui
xst -norandkey -k ourui.keytab ourui
```
##  2、到key admin 用户创建给user_a的 key


```
kinit -kt keyAdminUser.keytab   keyAdminUser
hadoop key create user_a_key2
```
结果如下：

```
[root@**** ~]# kinit -kt keyAdminUser.keytab   keyAdminUser
[root@**** ~]# hadoop key create user_a_key2
user_a_key2 has been successfully created with options Options{cipher='AES/CTR/NoPadding', bitLength=128, description='null', attributes=null}.
org.apache.hadoop.crypto.key.kms.LoadBalancingKMSClientProvider@6221a451 has been updated.
```

##  3、到hdfs用户给user_a 创建目录并赋权、创建zone


```
kinit  -kt hdfs.keytab  hdfs
hadoop  fs -mkdir /tmp/user_a_kms4test
hadoop  fs -chown user_a:analysis_group /tmp/user_a_kms4test
hdfs crypto -createZone -keyName user_a_key2 -path /tmp/user_a_kms4test
```
结果如下

```
[root@**** ~]# kinit  -kt hdfs.keytab  hdfs
[root@**** ~]# hadoop  fs -mkdir /tmp/user_a_kms4test
[root@**** ~]# hadoop  fs -chown user_a:idc_analysis_group /tmp/user_a_kms4test
[root@**** ~]# hdfs crypto -createZone -keyName user_a_key2 -path /tmp/user_a_kms4test
Added encryption zone /tmp/user_a_kms4test
```


##  4、到user_a用户上传文件、并测试可读性

```
kinit -kt user_a.keytab user_a
echo "Hello World" > /tmp/helloWorld.txt
hadoop fs -put /tmp/helloWorld.txt /tmp/user_a_kms4test
hadoop fs -cat /tmp/user_a_kms4test/helloWorld.txt
rm /tmp/helloWorld.txt
```
结果如下：

```
[root@**** ~]# hadoop fs -put /tmp/helloWorld.txt /tmp/user_a_kms4test
17/04/11 18:18:45 WARN kms.LoadBalancingKMSClientProvider: KMS provider at [http://lpsllfdrcn1.lfidcwanda.cn:16000/kms/v1/] threw an IOException [User [user_a] is not authorized to perform [DECRYPT_EEK] on key with ACL name [user_a_key2]!!]!!
17/04/11 18:18:45 WARN kms.LoadBalancingKMSClientProvider: KMS provider at [http://lpsllfdrcn2.lfidcwanda.cn:16000/kms/v1/] threw an IOException [User [user_a] is not authorized to perform [DECRYPT_EEK] on key with ACL name [user_a_key2]!!]!!
17/04/11 18:18:45 WARN kms.LoadBalancingKMSClientProvider: Aborting since the Request has failed with all KMS providers in the group. !!
put: User [user_a] is not authorized to perform [DECRYPT_EEK] on key with ACL name [user_a_key2]!!
17/04/11 18:18:45 ERROR hdfs.DFSClient: Failed to close inode 1404823
```

从结果看2 user_a对user_a_key2没有 DECRYPT_EEK权限，这时候就设计到可以的白名单设置了。下面我们到kms-acl.xml文件里面配置该key的权限

```
<property>
    <name>key.acl.user_a_key2.DECRYPT_EEK</name>
    <value>user_a</value>
    <description>
      ACL for decryptEncryptedKey operations.
    </description>
  </property>
```
滚动重启KMS server，
我们继续写入数据

```
[root@**** ~]# hadoop fs -put /tmp/helloWorld.txt /tmp/user_a_kms4test
[root@**** ~]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: user_a@a.b.NET

Valid starting       Expires              Service principal
04/11/2017 18:18:18  04/12/2017 18:18:18  krbtgt/a.b.NET@a.b.NET
        renew until 04/18/2017 18:18:18
```

数据写入成功，测试读数据

```
[root@**** ~]# hadoop fs -cat /tmp/user_a_kms4test/helloWorld.txt                    
Hello World
```
读数据成功。


##  5、到user_b用户读取上传数据


```
[root@**** ~]# kinit -kt user_b.keytab user_b
[root@**** ~]# hadoop fs -cat /tmp/user_a_kms4test/helloWorld.txt 
17/04/11 18:40:10 WARN kms.LoadBalancingKMSClientProvider: KMS provider at [http://ipdrcn1.lfidcwan.cn:16000/kms/v1/] threw an IOException [User [user_b] is not authorized to perform [DECRYPT_EEK] on key with ACL name [user_a_key2]!!]!!
17/04/11 18:40:10 WARN kms.LoadBalancingKMSClientProvider: KMS provider at [http://ipdrcn2.lfidcwan.cn:16000/kms/v1/] threw an IOException [User [user_b] is not authorized to perform [DECRYPT_EEK] on key with ACL name [user_a_key2]!!]!!
17/04/11 18:40:10 WARN kms.LoadBalancingKMSClientProvider: Aborting since the Request has failed with all KMS providers in the group. !!
cat: User [user_b] is not authorized to perform [DECRYPT_EEK] on key with ACL name [user_a_key2]!!
```


##  6、到hdfs用户读取上传数据


```
[root@**** ~]# kinit -kt hdfs.keytab hdfs
[root@**** ~]# hadoop fs -cat /tmp/user_a_kms4test/helloWorld.txt 
17/04/11 18:40:31 WARN kms.LoadBalancingKMSClientProvider: KMS provider at [http://ipdrcn1.lfidcwan.cn:16000/kms/v1/] threw an IOException [User:hdfs not allowed to do 'DECRYPT_EEK' on 'user_a_key2']!!
17/04/11 18:40:31 WARN kms.LoadBalancingKMSClientProvider: KMS provider at [http://ipdrcn2.lfidcwan.cn:16000/kms/v1/] threw an IOException [User:hdfs not allowed to do 'DECRYPT_EEK' on 'user_a_key2']!!
17/04/11 18:40:31 WARN kms.LoadBalancingKMSClientProvider: Aborting since the Request has failed with all KMS providers in the group. !!
cat: User:hdfs not allowed to do 'DECRYPT_EEK' on 'user_a_key2'
```

