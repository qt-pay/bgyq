## Postman使用

### k-v格式

先F12找到目标url，然后`Copy as cURL(bash)`， 得到下面的curl 信息

```bash
curl 'http://100.5.15.111:9091/api/v1/cluster/deployment?t=1629286947976' \
-H 'Connection: keep-alive' \
-H 'Pragma: no-cache' \
-H 'Cache-Control: no-cache' \
-H 'Accept: application/json, text/plain, */*' \
-H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhZG1pbiI6dHJ1ZSwiZXhwIjoxNjI5MzEzNDIwLCJuYW1lIjoieTE0OTg4In0.mOb01uCpwwkzuVZiPIlB3HOjCcywJaiOucaP_3TL2Y4' \
-H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36' \
-H 'Referer: http://100.5.15.111:9091/' \
-H 'Accept-Language: zh,en;q=0.9,en-US;q=0.8,zh-CN;q=0.7' \
-H 'Cookie: matrix-token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhZG1pbiI6dHJ1ZSwiZXhwIjoxNjI5MzEzNDIwLCJuYW1lIjoieTE0OTg4In0.mOb01uCpwwkzuVZiPIlB3HOjCcywJaiOucaP_3TL2Y4' \
--compressed \
--insecure
```

然后Postman执行Import --> `Paste Raw Text` ，将curl信息Paste后，报错了:`Invalid format for curl`。

百思不得其解...

直到看到了`--compressed not supported remove it from the end`.

我瞬间明白了，Postman只支持`key=value`形式的... 这样`--Parameter`不支持，所以报错。

去掉结尾的`--compressed and --insecure`即可。

### newman

　　Newman 是 Postman 推出的一个 nodejs 库，直接来说就是 Postman 的json文件可以在命令行执行的插件。
　　Newman 可以方便地运行和测试集合，并用之构造接口自动化测试和持续集成。

Newman is a command-line collection runner for Postman. It allows you to effortlessly run and test a Postman collection directly from the command-line. It is built with extensibility in mind so that you can easily integrate it with your continuous integration servers and build systems.

```bash
$ newman run --reporters cli,json  -e execute-environment.json --insecure --timeout-request 30000 --reporter-json-export execute-result.json  execute-collections.json > execute-log.log


```

作用：

前端页面将测试用例写入到json文件中，后端调用newman