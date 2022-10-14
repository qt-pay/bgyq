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

###　pre-script: js

postman集成了一个强大的，基于nodejs的script引擎，借助它，您可以为requests和collections添加动态的行为。  这样就可以在编写test suite时，构建可以包含动态参数的request，在request之间传递数据等等。您可以在流程中的两个事件中添加要执行的JavaScript代码：  

1. 在发送request之前，编写pre-request script，定制化request。 
2.  收到response之后，用test script，处理返回的数据。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/postman-script.png)

pre-request script就是一段在发送request之前执行的代码。大家可以自己脑补在什么场景可以用到它。比如，随机的URL参数，变化的requst body等。  这里要注意的是，pre-request script和test scripts一样，都是javascript，同时，和angular js一样，可以用两个{ {}}访问环境变量。 

```js
var jsoData = pm.response.json();
var temp = parseInt(jsonData[0].execution_id);
temp +=1;
pm.globals.set("xID")
```

使用变量渲染变化的URL和body

* 127.0.0.1/id/{{xID}}

* Post : 127.0.0.1/_search

  ```json
  {
      "query": {
          "id": "{{xID}}"
      }
  }
  ```

  

### test-script: 断言测试

拿到接口返回信息，根据需求做断言测试、

### 奇怪的测试需求:测试套件collection

接口1：仅创建任务（长时间执行），异步返回任务ID
接口2：持续检测任务（根据ID）运行状态，直到任务执行成功
接口3：任务返回两个数值，计算两数值的成功率，达到阈值就成功

前提：要想实现不同case test之间的变量引用，需要这些test case一起运行就在一个测试套件中，顺序执行。

分析：

* 需要一个for循环，加一个sleep function

  ```js
  function sleep(numberMillis) {
  
  var now = new Date();
  var exitTime = now.getTime() + numberMillis;
  while (true) {
  now = new Date();
  if (now.getTime() > exitTime)
  return;
  }
  }
  sleep(10000)
  ```

* 需要支持除法运算和if语句

伪代码：

可以将case 1 和 cas2 合并

```js
// test case 1 正常执行
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

// URL 
const url = 'http://IP/api/id={{ID}}';
function sleep(n){
    //实现sleep功能
}
// 避免无限循环
for (条件){
    // 发送get请求
pm.sendRequest(url, function (err, res) {
  console.log(err ? err : res.text());  // 控制台打印请求文本
});
    if (pm.response.to.have.status(200)){
        pm.globals.set("var", pm.response.json())
    }
    sleep(10)
}



// test case 2
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});
if (let res = m / n;) {
  // 大于 90% 返回 ok 否则 失败
}
```



### 引用

1. https://blog.csdn.net/wxy45645645/article/details/102697327