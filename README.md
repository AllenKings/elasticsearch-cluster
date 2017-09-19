# elasticsearch-cluster
create elasticsearch cluster use docker-compose.

包含的插件:
* [elasticsearch-analysis-ik](https://github.com/medcl/elasticsearch-analysis-ik)
* [search guard 2](https://floragunn.com/)

## ik 分词
提供中文分词

:smiley:如果不需要search guard插件功能，注释掉`config/elasticsearch.yml`文件中的相关配置，并删除`plugins/search-guard-2`
和`plugins/search-guard-ssl`目录即可

### 配置远程热更新词库
编辑配置文件`plugins/ik/config/IKAnalyzer.cfg.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
	<comment>IK Analyzer 扩展配置</comment>
	<!--用户可以在这里配置自己的扩展字典 -->
	<entry key="ext_dict">custom/mydict.dic;custom/single_word_low_freq.dic</entry>
	 <!--用户可以在这里配置自己的扩展停止词字典-->
	<entry key="ext_stopwords">custom/ext_stopword.dic</entry>
	<!--用户可以在这里配置远程扩展字典 -->
	<entry key="remote_ext_dict">http://somehost/dict.txt</entry>
	<!--用户可以在这里配置远程扩展停止词字典-->
	<!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>

```

## search guard 安全插件
elasticsearch 自身并不带任何安全功能，这部分是由插件提供。有[elastic](https://www.elastic.co)官方提供的**shield**，
但是需要购买对应授权才能使用；还有一个就是**search guard**了，
功能同样强大，支持 elasticsearch 各个版本。

详细的配置请参考官方文档：(http://floragunncom.github.io/search-guard-docs/)

:smiley:如果不需要ik插件，注释掉`config/elasticsearch.yml`文件中的相关配置，并删除`plugins/search-guard-2`目录即可

### 生成相关证书
使用官方提供的[search-guard-ssl](https://github.com/floragunncom/search-guard-ssl)项目来生成所需要的证书。

clone下来后，使用`example-pki-scripts`目录内的相关命令来生成自签名证书。

_step 1: CA_

```shell
./gen_root_ca.sh CA_PASS 
```

这个`CA_PASS`和`TS_PASS`十分重要，需要记住。后面将使用`CA_PASS`来给其他证书签名，`TS_PASS`在**search guard**配置中多处使用。

_step 2: 节点证书_

```shell
./gen_node_cert.sh 0 NODE_KS_PASS CA_PASS 
```

`NODE_KS_PASS`会在后面的配置中用到


_step 3: 管理员客户端证书_

```shell
./gen_client_node_cert.sh sga ADMIN_CLIENT_KS_PASS CA_PASS
```

_step 4: 请求客户端证书_

```shell
./gen_client_node_cert.sh java JAVA_CLIENT_KS_PASS CA_PASS
```

我们使用`jks`格式的证书，上面三步生成了：
* `truststore.jks` 包含公钥的证书，用于验证其他证书是否合法
* `node-0-keystore.jks` 节点证书，颁发给节点表明其身份
* `sga-keystore.jks` 管理员客户端证书，用于初始化`search guard`时判定权限
* `java-keystore.jks` JAVA客户端证书，在使用`TransportClient`进行连接时需要提供认证

### 将证书COPY至配置目录

将 `truststore.jks` `node-0-keystore.jks` 拷贝至`config`目录，
在容器运行时会挂载到每个节点的`/usr/share/elasticsearch/config`

将 `sga-keystore.jks` 拷贝至`config`目录，这个证书是在`elasticsearch`启动后，初始化`search guard`时使用，
也可以在启动后使用`docker cp`命令拷贝至目标节点，然后进行初始化操作

### 添加用户
添加`search guard`用户，在访问`elasticsearch`时做验证，保护数据安全。

编辑`plugins/search-guard-2/sgconfig/sg_internal_users.yml`文件

```yaml
esadmin:
  hash: *hash.sh -p <password>*
  
java_client:
  username: CN=java,OU=SSL,O=Test,L=Test,C=CN
  hash: "the password is not really necessary"
```

这里添加了两个用户：
* `esadmin` 设定为管理员，可以管理整个集群
* `CN=java,OU=SSL,O=Test,L=Test,C=CN` 设定为java连接客户端，有限的访问权限（使用下面的角色控制）
 这种配置方式的用户名为`DN`，从证书中获取的信息只要匹配就认证成功，所以这里密码是非必要的

:imp:这里最好将不使用的用户注释，配置语法在文件的注释中有详细说明

### 自定义角色

编辑`plugins/search-guard-2/sgconfig/sg_roles.yml`文件，在下面新增两个角色：

```yaml
# 限定 articles 这个索引，并且只读
idx_ro_articles:
  indices:
    'articles':
      '*':
        - READ

# 限定 articles 这个索引，可以进行 CRUD 操作
idx_crud_articles:
  indices:
    'articles':
      '*':
        - CRUD
```

该配置中也有许多已经创建好的角色可供选择。

### 映射角色和用户的关系

编辑`plugins/search-guard-2/sgconfig/sg_roles_mapping.yml`文件，新增一组映射：

```yaml
# 只允许 zhang 进行文章查询
idx_ro_articles:
  users:
    - zhang

# 只允许 java 和 li 进行文章的CRUD操作
idx_crud_articles:
  users:
    - 'CN=java,OU=SSL,O=Test,L=Test,C=CN'
    - li
```

## 启动集群

在使用`search guard`和`ik`并配置好的情况下，可以使用`docker-compose`命令启动集群

### 修改docker-compose配置

编辑 `docker-compose.yml` 配置文件

:imp:`elasticsearch`的版本不要随意改动，因为插件的版本和`elasticsearch`的版本严格对应。
如果修改版本，请重新安装对应版本的插件。

`./es-data` 用于存储`elasticsearch`的日志及数据，为了避免数据丢失，请制定好响应的备份策略

### 修改节点配置

编辑 `config/elasticsearch.yml` 配置文件，该文件将配置每个节点的行为

TCP连接时对应的配置：
* `searchguard.ssl.transport.keystore_password` 节点证书密码
* `searchguard.ssl.transport.truststore_password` 信任库密码
* `searchguard.ssl.transport.keystore_filepath` 节点证书路径，相对于`/usr/share/elasticsearch/config`的路径，
 也就是本地的`config`目录，因为启动时我们会将`config`目录挂载至每个节点
* `searchguard.ssl.transport.truststore_filepath` 信任库路径

HTTP连接时的配置，跟TCP对应相同：
* `searchguard.http.transport.keystore_password` 节点证书密码
* `searchguard.http.transport.truststore_password` 信任库密码
* `searchguard.http.transport.keystore_filepath` 节点证书路径
* `searchguard.http.transport.truststore_filepath` 信任库路径

允许操作search guard配置的管理员配置（执行初始化命令）：
* `searchguard.authcz.admin_dn` 在前面我们为一个名为`sga`的client生成了一个证书

### 运行集群

在`docker-compose.yml`所在目录执行启动命令
```shell
docker-compose up -d
```

### 初始化Search Guard

在使用`docker-compose`启动后，还无法正常访问，因为`search guard`还没有初始化。

使用 `docker ps` 列出集群的各个节点

使用 `docker exec -it <容器id> /bin/bash` 进入容器内部

切换至`search guard`配置命令所在目录:
```shell
cd /usr/share/elasticsearch/plugins/search-guard-2/tools/
```

执行初始化命令（确保`sgadmin.sh`文件拥有可执行权限）：
```shell
sgadmin.sh \
-cd /usr/share/elasticsearch/plugins/search-guard-2/sgconfig \
-cn elasticsearch \
-ks /usr/share/elasticsearch/config/sga-keystore.jks \
-ts /usr/share/elasticsearch/config/truststore.jks \
-nhnv \
-kspass sga_psw \
-tspass ts_psw
```

需要注意的是，`-cn`指定的集群名称必须和`elasticsearch.yml`配置文件中配置的一致，否则无法完成初始化

`-ks`指定管理员证书的路径，如果前面没有COPY至`config`目录，那么这里需要上传至节点容器内

`-kspass`和`-tspass`需要指定正确，否则无法正常初始化

现在访问集群，提示输入用户名密码！配置完成。
