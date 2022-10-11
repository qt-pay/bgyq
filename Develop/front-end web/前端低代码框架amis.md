## 前端低代码框架amis

前端低代码框架，通过 JSON 配置就能生成各种后台页面，极大减少开发成本，甚至可以不需要了解前端。

百度开源的好像有点意思

### 使用JSON配置页面

为了实现用最简单方式来生成大部分页面，amis 的解决方案是基于 [JSON](https://baike.baidu.com/item/JSON) 来配置，它的独特好处是：

- **不需要懂前端**：在百度内部，大部分 amis 用户之前从来没写过前端页面，也不会 `JavaScript`，却能做出专业且复杂的后台界面，这是所有其他前端 UI 库都无法做到的；
- **不受前端技术更新的影响**：百度内部最老的 amis 页面是 6 年多前创建的，至今还在使用，而当年的 `Angular/Vue/React` 版本现在都废弃了，当年流行的 `Gulp` 也被 `Webpack` 取代了，如果这些页面不是用 amis，现在的维护成本会很高；
- **享受 amis 的不断升级**：amis 一直在提升细节交互体验，比如表格首行冻结、下拉框大数据下不卡顿等，之前的 JSON 配置完全不需要修改；
- 可以 **完全** 使用 [可视化页面编辑器](https://aisuda.github.io/amis-editor-demo/) 来制作页面：一般前端可视化编辑器只能用来做静态原型，而 amis 可视化编辑器做出的页面是可以直接上线的。

### demo

`Service` 有个重要的功能就是支持配置 `schemaApi`，通过它可以实现动态渲染页面内容。

```json
                      {
                        "label": "预配置检查wow",
                        "className": "text-sm",
                        "icon": "fa fa-check",
                        "visible": true,
                        "-reload":"body",
                        "level": "success",
                        "type": "action",
                        "disabled":false,
                        "actionType": "dialog",
                        "dialog": {
                            "size": "xl",
                            "title": "检查中...",
                            "actions": [
                                { "type": "button", "actionType": "confirm", "label": "关闭", "primary": true }
                            ],
                            "body": {
                                // 动态渲染
                                "type": "service",
                                // 后端接口地址
                                "schemaApi": "/api/websshLoader?height=680&env=cvk&task=precheck"
                            }
                        }
                      },

```

