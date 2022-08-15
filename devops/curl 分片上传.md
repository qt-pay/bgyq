## curl 分片上传

通过curl调用接口上传一个较大的文件到服务器，因为会经过网关（Nginx）有max_client_body的限制，所以需要用到curl分片上传。

### Content-Type

媒体类型`multipart/form-data`相对于其他媒体类型如`application/x-www-form-urlencoded`等来说，最明显的不同点是：

- 请求头的`Content-Type`属性除了指定为`multipart/form-data`，还需要定义`boundary`参数
- 请求体中的请求行数据是由多部分组成，`boundary`参数的值模式`--${Boundary}`用于分隔每个独立的分部
- 每个部分必须存在请求头`Content-Disposition: form-data; name="${PART_NAME}";`，这里的`${PART_NAME}`需要进行`URL`编码，另外`filename`字段可以使用，用于表示文件的名称，但是其约束性比`name`属性低（因为并不确认本地文件是否可用或者是否有异议）
- 每个部分可以单独定义`Content-Type`和该部分的数据体
- 请求体以`boundary`参数的值模式`--${Boundary}--`作为结束标志



### 接口参数说明

| Header参数名 | 必选 | 类型及范围 | 说明                |
| ------------ | ---- | ---------- | ------------------- |
| Content-Type | true | String     | multipart/form-data |

### 接口请求体(form-data)

| totalChunks | true  | Integer | 应用包总片数       |
| ----------- | ----- | ------- | ------------------ |
| chunkNumber | true  | Integer | 当前分片序号       |
| id          | ture  | String  | 应用包版本ID       |
| pkgType     | true  | String  | 应用包版本类型     |
| file        | true  | File    | 应用包文件         |
| name        | true  | String  | 应用包名称         |
| version     | true  | String  | 应用包版本号       |
| visible     | true  | boolean | 是否上传至公有仓库 |
| label       | False | String  | 应用包的显示名称   |
| tagId       | False | String  | 应用包的标签ID     |
| logo        | False | File    | 应用包logo         |
| description | False | String  | 应用包描述信息     |
| feature     | False | String  | 应用包版本特性     |

 

### 详细脚本

`-F "file=@__FILE_PATH__"` 的请示，传输文件即可。

但是问题是，file size 大于 nginx client_max_body所以需要先将文件切割并使用 `multipart/form-data  `Content-Type。

#### `-F` and `type`

```bash
$ man curl

#  This enables uploading of binary files etc. To force the 'content' part to be a file, prefix the file  name  with an  @  sign.  T
# upload file
$ curl -F profile=@portrait.jpg https://example.com/upload.cgi

# You can also tell curl what Content-Type to use by using 'type='

$ curl -F "web=@index.html;type=text/html" example.com


# map
$ curl -F 'colors="red; green; blue";type=text/x-myapp' example.com
 
# You can add custom headers to the field by setting headers=, like
$ curl -F "submit=OK;headers=\"X-submit-type: OK\"" example.com

```



demo with basic auth

```bash
# -u, --user <user:password>
#  Specify the user name and password to use for server authenticatio
# -H, --header <header/@file>
# curl -H "X-First-Name: Joe" http://example.com/

uploadFile(){
fileID=`uuidgen`
fileName=$1
pkgName=$2
pkgVersion=$3
pkgDescription="uploaded by shell"
pkgType="web"
visible=$4

split -b200m $fileName $fileID 
if [ $? -ne 0 ]; then
echo "split file failed!"
exit 1
fi

totalChunks=`ls \$fileID* | wc -l`
if [ $totalChunks -le 0 ];then
echo "get chunk number failed"
exit 1
fi
     
index=1
for sfile in $fileID*
do
curl -X POST -u "admin:Passw0rd" -F "file=@$sfile;filename=$fileName" -F "totalChunks=$totalChunks" -F "chunkNumber=$index" -F "id=$fileID" -F "name=$pkgName" -F "version=$pkgVersion" -F "description=$pkgDescription" -F "visible=$visible" -F "pkgType=$pkgType" http://$VIP/api/packages 
if [ $? -ne 0 ]; then
echo "upload split failed"
exit 1
fi
let index+=1
done
 
echo "Successfully upload file"
return 0
}

VIP=${Interface_url}
uploadFile cloudos-harbor.zip harbor v1 "true" 

```

end

