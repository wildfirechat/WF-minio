# 野火Minio

野火Minio是基于最流行的开源对象存储服务[Minio](https://github.com/minio/minio)进行二次开发的，二次开发的内容仅涉及到与野火IM的对接。因此野火Minio可以按照原生Minio的使用和运维，仅有部分关于IM的配置在首次启动后需要配置。***野火Minio依赖于专业版，适合对安全性能要求很高的私有化部署，社区版不支持。***

## 运行方式
1. 上传：客户端需要上传时，先调用IM服务获取上传的token，IM服务根据配置里的信息计算出一个token（不需要与OSS进行调用）返回给客户端，客户端用自己的私钥把数据加密后用IM服务返回的token直接上传到OSS服务。OSS收到请求后验证token，验证通过后，再去通过IM服务的API获取用户的私钥进行解密保存文件。
2. 下载：客户端直接请求OSS服务。如果bucket禁止非授权访问，客户端可以调用getAuthorizedMediaUrl方法来获取经过授权的URI进行访问，这个如上传一样，由服务器端计算出授权。一般建议头像和动态表情运行非授权访问，其它重要文件可以配置成授权访问。

综上OSS服务需要能够获取到用户的密钥，用来解密客户端加密的信息。获取用户密钥是通过IM服务的server api进行。

## 环境要求
OSS服务需要独立的公网IP。客户端上传下载都是与OSS服务进行直接连接，不经过IM服务器，如果想要好的体验，需要保证一定的带宽。OSS服务如果通过server api与im服务交互，需要确保服务的连通性，可以限制除了访问IM服务管理端口以外的所有出访。OSS服务需要限制除80/443以外的所有入访。OSS服务还有一个管理端口，仅用于Minio管理后台登陆管理，这个需要限制外网访问，仅对特定IP允许访问或者配置完成后禁止外网访问。

## 升级方法
如果已经部署2023.3.24之前的版本，部署最新版本时需要按照[minio官方文档](https://min.io/docs/minio/linux/operations/install-deploy-manage/migrate-fs-gateway.html)进行升级。旧版本保留在```legacy_20230324```分支。

## 首次部署
#### 1. 下载Minio服务版本
需要使用野火提供的Minio服务，不能用Minio官方的版本。请从本项目[minio_release](./minio_release)目录下下载对应版本的Minio服务。

#### 2. 启动野火Minio服务
```sh
## minio-data 为数据目录，需要运行前创建好，且具有读写权限。API端口（对外服务使用的）是9000，管理端口（Minio官方称为console端口）是9002，这两个端口都可以改成别的端口。
## 如果目录/minio-data已经被其他版本的Minio服务启动过，可能需要进行数据迁移或者清空内容，防止有兼容问题产生。
./minio server --address :9000  --console-address :9002 /minio-data
```
运行成功后会有如下的提示
```
Endpoint:  http://10.255.20.59:9000  http://127.0.0.1:9000

Browser Access:
   http://10.255.20.59:9000  http://127.0.0.1:9000

Object API (Amazon S3 compatible):
   Go:         https://docs.min.io/docs/golang-client-quickstart-guide
   Java:       https://docs.min.io/docs/java-client-quickstart-guide
   Python:     https://docs.min.io/docs/python-client-quickstart-guide
   JavaScript: https://docs.min.io/docs/javascript-client-quickstart-guide
   .NET:       https://docs.min.io/docs/dotnet-client-quickstart-guide

```
> 如果没有可执行权限，使用```chmod a+x minio```来添加可执行权限。如果端口小于1000，比如80端口，在linux机器上使用root权限，使用```root```用户或者```sudo```命令来运行。默认AK/SK是比较简单的，正式上线前需要改为复杂字符串

#### 3. 解压mc目录下的```mc```工具。增加可执行权限，然后执行下面语句为Minio服务设置别名。
```shell script
./mc alias set myminio http://47.52.118.96:9000 minioadmin minioadmin
```
> 可以在Minio服务所在的机器上本地运行也可以远程执行。另外```myminio```是服务的别名，可以任意起名，后面需要用到，如果在一台电脑操作多个minio服务，注意别名不要重复

> 最后两个参数为AK/SK，需要使用正确的值，第一步启动的控制台日志中会有。

> 注意端口是API端口9000，不是管理端口，不要写错了。

#### 4. 更新Minio的野火IM配置
```
./mc admin config set myminio WFChat IMAdminUrl=http://${im_server_address}:18080/admin/minio/sk IMAdminSecret=${im_server_admin_secret}
```
> IMAdminUrl是IM服务的server api地址，建议用内网地址，需要确保minio服务与IM服务管理端口的连通性。18080是默认的管理端口，如果修改过这里也需要对应修改。IMAdminSecret为IM服务的server api的密钥。

如果IM服务使用国密加密，需要执行下面操作开启国密加密。注意只有特殊客户才配置此项，普通客户请忽略此配置。
```
./mc admin config set myminio WFChat SM4Encrypt=on
```
> 如果要关闭，on改成off就可以了

#### 5. 重启野火Minio服务
重启服务，好让设置生效，必须重启才行。使用如下命令重启
```
./mc admin service restart myminio
```

#### 6. 新建bucket
用浏览器打开```http://47.52.118.96:9002```（这里作为示例，实际使用时请替换IP和管理端口），如果是升级部署可以看见之前存在的bucket，内部数据都存在，不用再创建bucket，如果是首次部署则为空。点左上角的```Create Bucket```按钮，创建bucket。创建至少2个bucket（建议每个桶建一个，桶的列表见下面配置），如下图所示：
![bucket list](./asset/bucket_list.png)

设置权限,点击bucket进入bucket详情也没，按照图上步骤说明为匿名用户添加桶的读权限。
![edit_policy](./asset/bucket_policy.png)

> 建议桶的策略是私有写公开读的，这样消息中的资源就可以被客户端直接访问。如果想要增加安全性，可以设置成私有读写，客户端需要先获取认证后再访问，详情请参考野火文档安全说明。

配置完bucket以后，就可以防火墙关掉控制台端口的入访权限，保证数据安全。

#### 7. 配置野火IM
野火IM的配置请参考专业版野火IM部署说明，***更改完配置后重启IM服务***。
```
##存储使用类型，0使用内置文件服务器（仅供用于研发测试），1使用七牛云存储，2使用阿里云对象存储，3野火私有对象存储
##除了内置文件服务器外，其他对象存储服务需要设置上传需要鉴权，下载不需要鉴权模式。
##这里改成3
media.server.media_type 3

## minio服务地址，要求是域名或者公网IP，不用加http头
media.server_host  47.52.118.96
## 下面配置API端口，不能写到server_url中去。如果API端口非80，下面media.bucket_XXXX_domain的地址也要加上API端口。
media.server_port 9000
## web客户端，小程序客户端和大文件上传需要用到https，请配置nginx支持https，关于https后面有说明
media.server_ssl_port 9443
## AK/SK minio服务启动时会打印出来AK/SK，默认是minioadmin，需求修改复杂字符串。
media.access_key minioadmin
media.secret_key minioadmin

## bucket名字
media.bucket_general_name media
## domain为minio服务器地址/bucket名字,不是服务器地址/minio/bucket名字。如果API端口非80还要加上端口号。
media.bucket_general_domain http://47.52.118.96:9000/media
media.bucket_image_name media
media.bucket_image_domain http://47.52.118.96:9000/media
media.bucket_voice_name media
media.bucket_voice_domain http://47.52.118.96:9000/media
media.bucket_video_name media
media.bucket_video_domain http://47.52.118.96:9000/media
media.bucket_file_name media
media.bucket_file_domain http://47.52.118.96:9000/media
media.bucket_sticker_name media
media.bucket_sticker_domain http://47.52.118.96:9000/media
media.bucket_moments_name media
media.bucket_moments_domain http://47.52.118.96:9000/media
media.bucket_portrait_name storage
media.bucket_portrait_domain http://47.52.118.96:9000/storage
media.bucket_favorite_name storage
media.bucket_favorite_domain http://47.52.118.96:9000/storage
```
> 上述参数为示例参数，请替换为客户对应的参数。

> bucket media/storage为示例，客户实际使用时可以使用不同的名称。但至少要创建2个用来存储长期保存和短期保存的媒体文件，建议为上述9类每类创建一个bucket，这样才能更精细化地处理媒体文件。

> 上传必须支持http上传（因为移动端和PC端只能支持http上传，不用担心安全问题，因为数据是经过加密过的），因此不能屏蔽掉http访问，media.bucket_XXXX_domain建议增加https的支持（后面有HTTPS支持的说明)。


#### 8. 验证上传下载是否正常。
验证发送图片/语音/视频/文件等媒体类消息，验证修改用户头像，验证大文件上传。

#### 9. 配置HTTPS
野火客户端在处理文件上传时有两种情况，一种是小文件，通过协议栈加密后使用http上传；另外一种是大文件，在客户端的client层上传，在client层上传时没有经过加密。另外文件下载时也是没有加密的。所以为了安全需要配置HTTPS，通过HTTPS进行大文件上传和所有文件下载。因此需要配置minio服务同时支持http和https（http必须保留，因为移动端和pc端协议栈还需要使用）。配置见下面说明。

## 进阶配置
#### 1. 使用域名
配置域名解析到minio服务器，然后IM服务配置文件中的IP替换成域名。这样如果以后迁移存储服务也不会有问题。

#### 2. 集群部署
只有部署集群后，才可以提供高可用，当少于一半的存储磁盘损坏也不会丢失数据。具体部署方法请自行查找Minio官方文档，按照minio官方部署集群后还需要按照上面说明对接野火IM。

#### 3. 配置https
客户端协议栈上传必须是http，http method是```put```，大文件上传没有加密建议通过https上传，还有下载文件时也可以使用https来增加安全性。首先用如下命令启动，minio服务的端口为9000
```
./minio server --address :9000 --console-address :9002 /minio-data
```
然后配置nginx，同时监听 80 和 443 端口(这两个端口也可以更换为其他端口，需要和 im-server 配置文件里面的`media.server_port`和`media.server_ssl_port`保持一致)，其中443 端口需要配置ssl证书，这样能够同时支持HTTP/HTTPS双栈，注意转发时要把```http header```都带上。

最后就是修改配置文件```media.server_host```Nginx的公网host，```media.bucket_XXXX_domain```改为https对应地址，```media.server_port```为NG的http端口，```media.server_ssl_port```为NG的ssl的端口。

因为客户端是直连minio进行上传下载的，不是通过IM服务中转的，所以```media.server_host```必须是客户端可以访问的。另外NG转发需要把所有的内容都转到minio服务去，可以参考野火提供的[示例](./nginx/minio.conf)。

#### 4. 配置CDN加速
如果客户比较多，且全国甚至全世界分别比较广，使用dns能够提高下载速度，提高用户体验。可以对下载进行dns加速，然后正确配置```media.bucket_XXXX_domain```。

## 常见问题
#### 1. 使用nginx反向代理后，不能发送大文件
这是因为nginx默认不能发送大文件，请把最大文件限制改为4G或者更大，最大超时时间改为60分钟以上。如下所示：
```
#上传文件大小限制
client_max_body_size 4096M;

#设置为on表示启动高效传输文件的模式
sendfile on;

#保持连接的时间，保持60分钟以上
keepalive_timeout 3600;
```
#### 2. 上传文件失败，minio服务抛出异常
检查minio控制台日志异常调用栈中是否有```get user secret```字段。如果有则说明是minio调用im服务获取用户密钥失败，检查minio服务是否可以访问im服务的管理端口（默认是18080），管理密钥是否正确等。也常见于修改了IM服务的管理密钥，忘记更新minio服务配置中的IM服务的管理密钥。

#### 3. 使用nginx反向代理没有转发header
header中带有用户id信息，这样minio可以查找到对应用户id的密钥，用来解密上传的内容。***请确保转发时带上所有header***，下面为参考配置 ：
```
location / {
        proxy_set_header  Host  $http_host;
        proxy_set_header  X-real-ip $remote_addr;
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass  http://minio_cluster;
}
```

#### 4. 不要修改minio的region
不要修改minio的region，不能修改为其他值。这个值实际使用上没有其他的意义。

#### 5. 启动失败
有可能使用minio的CPU架构不对，比如在X86架构的服务器上使用了Arm64架构的minio，需要用Amd64架构的minio；也有可能是数据目录```/minio-data```被其他版本的Minio创建过，数据不兼容导致启动失败。

#### 6. 国密配置错误
国密一般是不开启的，请保持国密为关。如果要开启，需要跟IM服务和客户端确认，需要三方同时开启才可以，如果有一个不一致，会导致无法连接或者无法上传。

#### 7. 其它问题
如果还是无法解决问题，请把客户端协议栈日志、minio控制台日志和tcpdump日志一起发给我们。抓去tcpdump日志的命令如下：
```
sudo tcpdump -w minio.pcap
```
客户端操作，等待失败后，ctrl+c结束命令，日志在当前目录的minio.pcap文件。

## 鸣谢
感谢[Minio](https://github.com/minio/minio)提供如此棒的开源产品
