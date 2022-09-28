分享一个工作中使用dmn-js的实践案例。需求背景是在绩效系统中实现量化打分、用户可自定义得分规则。对此我们应用到了DMN，其XML由决策引擎Camunda直接执行，前端方面集成了dmn-js。

考虑到编写规则表和打分功能的易用性，系统管理了规则属性列表，用户在设置规则时，只需选择规则属性，就会自动生成DMN关系图，然后填充相应的规则表达式即可。

- DMN（Decision Model and Notation）决策模型标记
- Camunda 工作流引擎
- dmn-js 实现在浏览器中查看和编辑DMN关系图

**本文主要是分享在vue应用中，使用dmn-js实现在浏览器中选择规则属性自动生成对应的DMN关系图骨架和XML。**

## 演示

[**在线体验**](https://kaxiluo.github.io/dmn-js-vue/dmn.html)

![dmn-js-vue-example](https://kxler.oss-cn-shanghai.aliyuncs.com/2022/09/dmn-js-vue-example.gif)

## 源码

源码GitHub地址 -> [kaxiluo/dmn-js-vue](https://github.com/kaxiluo/dmn-js-vue)
