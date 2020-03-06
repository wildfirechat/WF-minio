# 野火Minio

野火Minio是基于最流行的开源对象存储服务[Minio](https://github.com/minio/minio)进行二次开发的，二次开发的内容仅涉及到与野火IM的对接。因此野火Minio可以按照原生Minio的使用和运维。但也有一些细微细节不同。首次部署需要进行配置，以后可以正常启动就行。***野火Minio依赖于专业版，适合对安全性能要求很高的私有化部署，社区版不支持。***

## 运行方式
1. 上传：客户端需要上传时，先调用IM服务获取上传的token，IM服务根据配置里的信息计算出一个token（不需要与OSS进行调用）返回给客户端，客户端用自己的私钥把数据加密后用IM服务返回的token上传。OSS收到请求后验证token，验证通过后，在去IM服务的数据库获取用户的私钥进行解密保存文件。
2. 下载：客户端直接请求OSS服务。

综上OSS服务与IM系统没有直接的交互，唯一需要就是能访问IM数据库。

## 配置方法

#### 1. 环境配置
OSS服务需要独立的公网IP。客户端上传下载都是与OSS服务进行直接连接，不经过IM服务器，如果想要好的体验，需要保证一定的带宽。OSS服务与IM系统没有直接的联系，OSS服务需要访问IM数据库，所以防火墙可以禁止除了IM数据库以外的所有主动出访，限制除80以外的所有入访。

#### 2.启动野火IM服务
启动野火IM服务，创建数据库，后面Minio需要用到野火IM的数据库

#### 3.启动野火Minio服务
```sh
minio  server /minio-data
```
运行成功后会有如下的提示
```
Endpoint:  http://47.52.118.96  http://127.0.0.1            
AccessKey: 0M7YVO70QPKBPWBZW5FW
SecretKey: ZrBsSST++1Qjap+Nfs3P2BujHCHDuqrsrYi0zNn8

Browser Access:
   http://47.52.118.96  http://127.0.0.1            

Command-line Access: https://docs.min.io/docs/minio-client-quickstart-guide
   $ mc config host add myminio http://47.52.118.96 0M7YVO70QPKBPWBZW5FW ZrBsSST++1Qjap+Nfs3P2BujHCHDuqrsrYi0zNn8

Object API (Amazon S3 compatible):
   Go:         https://docs.min.io/docs/golang-client-quickstart-guide
   Java:       https://docs.min.io/docs/java-client-quickstart-guide
   Python:     https://docs.min.io/docs/python-client-quickstart-guide
   JavaScript: https://docs.min.io/docs/javascript-client-quickstart-guide
   .NET:       https://docs.min.io/docs/dotnet-client-quickstart-guide

```
> 如果没有可执行权限，使用```chmod a+x minio```来添加可执行权限。由于需要用到80端口，在linux机器上使用root权限，使用```root```用户或者```sudo```命令来运行.

#### 4. 解压mc目录下的```mc```工具。增加可执行权限，然后执行下面语句为Minio服务设置别名
```shell script
./mc config host add myminio http://47.52.118.96 0M7YVO70QPKBPWBZW5FW ZrBsSST++1Qjap+Nfs3P2BujHCHDuqrsrYi0zNn8
```
> 不需要在Minio服务所在的机器上运行，可以远程。另外```myminio```是服务的别名，可以任意起名，后面需要用到，如果在一台电脑操作多个minio服务，注意别名不要重复

> mc工具只支持mac和linux

#### 5. 获取Minio的配置
```shell script
mc admin config get myminio/ > myconfig
```
> 如果出现错误，请确认当前客户端是否跟Minio能够连通，Minio服务是否正常运行。

#### 6. 更改Minio的配置如下
```text
 "WFChat": {
  "DefaultSecret": "00,11,22,33,44,55,66,77,78,79,7A,7B,7C,7D,7E,7F",
  "MySQLAddr": "192.168.1.199:3306",
  "MySQLDB": "wfchat",
  "MySQLPassword": "123456",
  "MySQLUserName": "root"
 },
```
> ```DefaultSecret```需要配置IM服务参数```client.proto.secret_key```相同的值，可以保存默认不变，如果改动这里需要使用16进制，并把```0X```去掉。
> MySQL的地址正确配置就行。注意与野火IM MySQL配置的格式不通，保持当前这种格式。

执行下面语句，更新配置
```text
mc admin config set myminio/ < myconfig
```
#### 7. 重启野火Minio服务
重启服务，好让设置生效，必须重启才行。

#### 8. 新建bucket
用浏览器打开```http://47.52.118.96```（这里作为示例，实际使用时请换成客户服务的外网IP），然后点右下角的```+```，选择创建bucket。然后分别创建3个bucket，如下图所示：
![bucket list](./asset/bucket_list.png)

设置权限,点击bucket右侧的菜单按钮，选择```Edit policy```，弹出如下图界面，选择```Add```添加
![edit_policy](./asset/bucket_policy.png)


#### 9. 配置野火IM
野火IM的配置请参考专业版野火IM部署说明，更改完配置后重启。
```
##存储使用类型，0使用内置文件服务器（仅供用于研发测试），1使用七牛云存储，2使用阿里云对象存储，3野火私有对象存储
##除了内置文件服务器外，其他对象存储服务需要设置上传需要鉴权，下载不需要鉴权模式。下载的安全性在于生成对象的key为uuid，无法被穷举。
media.server.media_type 3 ##这里改成3

## minio服务地址，要求是公网地址
media.server_url  http://47.52.118.96
media.access_key 0M7YVO70QPKBPWBZW5FW
media.secret_key ZrBsSST++1Qjap+Nfs3P2BujHCHDuqrsrYi0zNn8

## bucket名字
media.bucket_general_name media
## domain为minio服务器地址/bucket名字。不是服务器地址/minio/bucket名字，如下所示
media.bucket_general_domain http://47.52.118.96/media
media.bucket_image_name media
media.bucket_image_domain http://47.52.118.96/media
media.bucket_voice_name media
media.bucket_voice_domain http://47.52.118.96/media
media.bucket_video_name media
media.bucket_video_domain http://47.52.118.96/media
media.bucket_file_name media
media.bucket_file_domain http://47.52.118.96/media
media.bucket_portrait_name storage
media.bucket_portrait_domain http://47.52.118.96/media
media.bucket_favorite_name storage
media.bucket_favorite_domain http://47.52.118.96/media
```
> 上述参数为示例参数，请替换为客户对应的参数。
> bucket media/storage为示例，客户实际使用时可以使用不同的名称。
> 上传必须支持http上传（我们已经加密过了），因此```media.server_url```必须是http的，media.bucket_XXXX_domain可以增加https的支持。

#### 10. 验证上传下载是否正常。
验证上传下载是否正常。

## 进阶配置
#### 1. 使用域名
配置域名解析到minio服务器，然后配置文件中的ip替换成域名即可。

#### 2. 配置https
上传必须是http，http method是```put```。可以下载配置为https。首先用如下命令启动，更换minio服务的端口为9000
```
./minio server --address 47.52.118.96:9000 /minio-data
```
然后配置nginx，反向代理到80端口，配置证书，使其能够同时支持HTTP/HTTPS双栈，注意中转时要把```http header```都带上。

最后就是修改配置文件```media.server_url```保持不变，```media.bucket_XXXX_domain```改为https对应地址。

#### 3. 配置CDN加速
如果客户比较多，且全国甚至全世界分别比较广，使用dns能够提高下载速度，提高用户体验。可以对下载进行dns加速，然后正确配置```media.bucket_XXXX_domain```。

## 鸣谢
感谢[Minio](https://github.com/minio/minio)提供如此棒的开源产品
