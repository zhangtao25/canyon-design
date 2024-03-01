前言

istanbuljs是当下最优秀的JavaScript覆盖率工具，但是它偏低层，只提供了代码探针插装，静态html报告生成的的功能。
2024年前端技术井喷式发展，istanbuljs提供的功能显得抽筋见肘。为此，我们在istanbuljs的基础上开发了一套JavaScript覆盖率解决方案Canyon，拥有处理高并发上报的覆盖率，实时覆盖率聚合，覆盖率报告水合展示的拥有等特性。携程机票、IBU、酒店、商旅等部门都以初具规模的使用，保证了UI自动化的提供了覆盖率数据指标的支持。


## 架构

Canyon的整体技术栈完全基于nodejs（前端、后端、任务、上报器），部署非常简便，仅需要nodejs环境，也适用于云原生环境部署（docker、Kubernetes）。应用整体流程为：代码探针插桩、触发器触发探针、覆盖率数据上报、消息生成覆盖率概览、覆盖率报告呈现。应用的架构设计适用于高频、大体积覆盖率数据的上报，使用分布式部署，消息队列消费。

## 代码覆盖率（已完成）

随着你编写更多的end-to-end测试case，你会发现自己有一些疑问，我需要写更多的测试用例吗？究竟还有哪些代码没测到？用例会不会重复了？这个时候代码覆盖率就派上用场了，它的原理是在代码执行前将代码探针插入到源代码中（其实就是上下文加计数器），这样每当case执行的时候就可以触发其中的计数器.

在代码中插入代码探针的步骤称为**代码插桩**（instrument）。插桩前的代码：

```js
// add.js
function add(a, b) {
  return a + b
}
module.exports = { add }
```

并解析它以查找所有函数、语句和分支，然后将计数器插入代码中。对于上面的代码，它可能看起来像这样：

```js
// 这个对象用于计算每个函数和每个语句被执行的次数
const c = (window.__coverage__ = {
  // "f" 记录每个函数被调用的次数
  f: [0],
  // "s" 记录每个语句被调用的次数
  // 我们有3个语句，它们都从0开始
  s: [0, 0, 0],
})

// 第一个语句定义了函数
c.s[0]++
function add(a, b) {
  // 函数被调用后是第二个语句
  c.f[0]++
  c.s[1]++

  return a + b
}
// 第三个语句即将被调用
c.s[2]++
module.exports = { add }

```

然后再通过与源代码的结合就可以生成漂亮的覆盖率详情报告，就可以明确的知道case执行到了哪些代码。

## 代码插桩（instrumenting-code）

简单介绍一下插桩原理



istanbul是久经沙场的js代码插桩黄金标准，Canyon主要为端到端测试提供解决方案，经过大量的实验验证，现代化前端工程的覆盖率插桩必须要编译时插桩。具体原因是istanbuljs提供的nyc插桩工具只能对原生js进行插桩，然而前端模版语法层出不穷，例如ts、tsx、vue，虽然nyc也可以插桩，但是结构实践证明直接插桩的覆盖率效果不尽人意，无法精确到该插桩到的函数、语句、分支。幸运的是经过调研，我们发现了[babel-plugin-istanbul](https://github.com/istanbuljs/babel-plugin-istanbul)、vite-plugin-istanbul（experimental）、swc-plugin-coverage-instrument(experimental)。等类型工程的插桩解决方案。这些方案无一例外都是在前端工程编译阶段在将代码分析成ast抽象语法树的时候在适当时机进行插桩方法调用，更精确的插桩到的函数、语句、分支。

适用的工程类型：

| 工程类型         | 方案                   |
|--------------|----------------------|
| 原生JavaScript | nyc instrument |
| babel        | babel-plugin-istanbul |
| swc          | swc                  |
| vite         | vite                 |

用户可以根据自己的工程类型选择合适的插桩方案，只需要在工程中安装对应的插件，然后就会在编译时自动插桩。

以babel.config.js为例：

```javascript

module.exports = {
  plugins: [
    [
      'babel-plugin-istanbul',
      {
        exclude: ['**/*.spec.js', '**/*.spec.ts', '**/*.spec.tsx', '**/*.spec.jsx'],
      },
    ],
  ],
};

```

插桩完成后，代码中会插入一些代码探针，这些代码探针会在运行时收集覆盖率数据，然后上报到Canyon服务端。

检查是否插桩成功，可以在编译后的产物中搜索`__coverage__`，如果有则说明插桩成功。


![img_2.png](./test.jpg)

为了紧密关联插桩代码的源代码，我们适配了各种provider，将环境变量发送到Canyon服务端，兑换到reportID，方便覆盖率数据聚合计算完成后的覆盖率源文件的关联展示。

我们还提供了babel-plugin-canyon的babel插件，可以在各种流水线内（aws，gitlab ci）读取环境变量(branch、sha)，以供后续覆盖率数据与对应的gitlab源代码关联。provider，提供商

babel.config.js

```javascript
module.exports = {
  plugins: [
    [
      'babel-plugin-istanbul',
      {
        exclude: ['**/*.spec.js', '**/*.spec.ts', '**/*.spec.tsx', '**/*.spec.jsx'],
      },
    ],
  ],
};
```

支持的提供商：


| 提供商                                                       | 环境变量              |
| ------------------------------------------------------------ | --------------------- |
| [Azure Pipelines](https://azure.microsoft.com/en-us/services/devops/pipelines/) | nyc instrument        |
| [CircleCI](https://circleci.com/)                            | babel-plugin-istanbul |
| [Drone](https://drone.io/)                                   |                       |
| [Github Actions](https://github.com/features/actions)        |                       |
| [GitLab CI](https://about.gitlab.com/gitlab-ci/)             |                       |
| [Jenkins](https://jenkins.io/)                               |                       |
| [Travis CI](https://travis-ci.org/)                          |                       |



需要特别注意的是，代码探针的插桩会在构建产物上下文加上代码探针，会是代码整体产物增大30%，建议不要上生产环境。


## 触发器（trigger）

chrome插件

## 报告



## 社区推广



## 参考链接

