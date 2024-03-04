# 开源 | Canyon: JavaScript代码覆盖率解决方案

## 前言

istanbuljs是当下最优秀的JavaScript覆盖率工具，但是它偏低层，只提供了代码探针插装，静态html报告生成的的功能。
2024年前端技术井喷式发展，istanbuljs提供的功能显得抽筋见肘。为此，我们在istanbuljs的基础上开发了一套JavaScript覆盖率解决方案Canyon，拥有处理高并发上报的覆盖率，实时覆盖率聚合，覆盖率报告水合展示的拥有等特性。携程机票、IBU、酒店、商旅等部门都以初具规模的使用，保证了UI自动化的提供了覆盖率数据指标的支持。


## 简述

Canyon的整体技术栈完全基于nodejs（前端、后端、任务、上报器），部署非常简便，仅需要nodejs环境，也适用于云原生环境部署（docker、Kubernetes）。应用整体流程为：代码探针插桩、触发器触发探针、覆盖率数据上报、消息生成覆盖率概览、覆盖率报告呈现。应用的架构设计适用于高频、大体积覆盖率数据的上报，使用分布式部署，消息队列消费。

## 代码覆盖率

随着你编写更多的end-to-end测试case，你会发现自己有一些疑问，我需要写更多的测试用例吗？究竟还有哪些代码没测到？用例会不会重复了？这个时候代码覆盖率就派上用场了，它的原理是在代码执行前将代码探针插入到源代码中（其实就是上下文加计数器），这样每当case执行的时候就可以触发其中的计数器.

在代码中插入代码探针的步骤称为**代码插桩**（instrument）。插桩前的代码：

```js
// add.js
function add(a, b) {
  return a + b
}
module.exports = { add }
```

插桩过程是对代码解析以查找所有函数、语句和分支，然后将计数器插入代码中。对于上面的代码，插桩完成后：

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

我们希望确保文件中的每个语句和函数`add.js`都已被我们的测试至少执行一次。因此我们编写一个测试：

```js
// add.cy.js
const { add } = require('./add')

it('adds numbers', () => {
  expect(add(2, 3)).to.equal(5)
})
```

当测试调用时`add(2, 3)`，执行“add”函数内的计数器递增，覆盖范围对象变为：

```js
{
  // "f" keeps count of times each function was called
  // we only have a single function in the source code
  // thus it starts with [0]
  f: [1],
  // "s" keeps count of times each statement was called
  // we have 3 statements, and they all start with 0
  s: [1, 1, 1]
}
```



这个单一测试已经实现了 100% 的代码覆盖率——每个函数和每个语句都至少执行了一次。但是，在现实应用程序中，实现 100% 的代码覆盖率需要多次测试。

测试完成后，可以将覆盖对象序列化并保存到磁盘，以便生成人性化的报告。收集到的覆盖范围信息还可以发送到外部服务，并在拉取请求审查期间提供帮助。



然后再通过与源代码的结合就可以生成漂亮的覆盖率详情报告，就可以明确的知道case执行到了哪些代码。

## 代码插桩（instrumenting-code）

简单介绍一下插桩原理



istanbul是久经沙场的js代码插桩黄金标准，Canyon主要为端到端测试提供解决方案，经过大量的实验验证，现代化前端工程的覆盖率插桩必须要编译时插桩。具体原因是istanbuljs提供的nyc插桩工具只能对原生js进行插桩，然而前端模版语法层出不穷，例如ts、tsx、vue，虽然nyc也可以插桩，但是结构实践证明直接插桩的覆盖率效果不尽人意，无法精确到该插桩到的函数、语句、分支。幸运的是经过调研，我们发现了[babel-plugin-istanbul](https://github.com/istanbuljs/babel-plugin-istanbul)、vite-plugin-istanbul（experimental）、swc-plugin-coverage-instrument(experimental)。等类型工程的插桩解决方案。这些方案无一例外都是在前端工程编译阶段在将代码分析成ast抽象语法树的时候在适当时机进行插桩方法调用，更精确的插桩到的函数、语句、分支。

适用的工程类型：

| 工程类型         | 方案                   |
|--------------|----------------------|
| vanilla javascript | [nyc](https://github.com/istanbuljs/nyc) |
| babel        | [babel-plugin-istanbul](https://github.com/istanbuljs/babel-plugin-istanbul) |
| vite      | [vite-plugin-istanbul](https://github.com/ifaxity/vite-plugin-istanbul) (experimental) |
| swc      | [swc-plugin-coverage-instrument](https://github.com/kwonoj/swc-plugin-coverage-instrument) (experimental) |

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


| 提供商                                                       |
| ------------------------------------------------------------ |
| [Azure Pipelines](https://azure.microsoft.com/en-us/services/devops/pipelines/) |
| [CircleCI](https://circleci.com/)                            |
| [Drone](https://drone.io/)                                   |
| [Github Actions](https://github.com/features/actions)        |
| [GitLab CI](https://about.gitlab.com/gitlab-ci/)             |
| [Jenkins](https://jenkins.io/)                               |
| [Travis CI](https://travis-ci.org/)                          |



需要特别注意的是，代码探针的插桩会在构建产物上下文加上代码探针，会是代码整体产物增大30%，建议不要上生产环境。


## 测试用例

当插桩完成发布到测试环境后，我们就可以开始测试阶段了。端到端测试我们以[playwright](https://playwright.dev/)为例。

```js
import { test, expect } from '@playwright/test';

test('测试累加器', async ({ page }) => {
  await page.goto('https://canyon.demo/');

  await page.getByTestId('add-btn').click();
  
  await expect(page.getByTestId('add-result')).toHaveText('3');
});
```



## 上报

当测试用例执行完成后，对于单页面应用(SPA)或者多页面应用而言，上报步骤是将页面window对象上的__coverage__对象上报到Canyon服务端，对于单页面应用来说，相对来说比较简单，在所有测试内容都在单页面应用内，覆盖率数据会常驻在window对象中，对于多页面应用而言，路由的跳转会导致window对象的重制，丢失coverage对象。所以这个时机是至关重要的，经过大量实践验证，我们找到了浏览器的on visible change方法


以下是关于 `visibilitychange` 事件的浏览器兼容性报告，已移除兼容性列：

| 浏览器              | 版本  |
| ------------------- | ----- |
| Chrome              | 62+   |
| Edge                | 18+   |
| Firefox             | 56+   |
| Opera               | 49+   |
| Safari              | 14.1+ |
| Chrome Android      | 62+   |
| Firefox for Android | 56+   |
| Opera Android       | 46+   |
| Safari on iOS       | 14.5+ |
| Samsung Internet    | 8.0+  |
| WebView Android     | 62+   |

请注意，`visibilitychange` 事件在大多数现代浏览器中得到支持，但在一些较旧版本中可能存

在浏览器可见性改变的时候上报覆盖率数据，值得一提的是，对于visibilitychange这种可能会导致重复数据上报，但是对于覆盖率统计来说，未执行到的代码多次合并来说不回影响覆盖率的具体指标数据统计。

对于高并发的测试场景，建议使用case本地聚合后上报覆盖率数据的方法，这样可以极大减少Canyon服务端数据处理压力。

```js
/**
 * 合并两个相同文件的文件覆盖对象实例，确保执行计数正确。
 *
 * @method mergeFileCoverage
 * @static
 * @param {Object} first 给定文件的第一个文件覆盖对象
 * @param {Object} second 相同文件的第二个文件覆盖对象
 * @return {Object} 合并后的结果对象。请注意，输入对象不会被修改。
 */
function mergeFileCoverage(first, second) {
  const ret = JSON.parse(JSON.stringify(first));

  delete ret.l; // 移除派生信息

  Object.keys(second.s).forEach(function (k) {
    ret.s[k] += second.s[k];
  });

  Object.keys(second.f).forEach(function (k) {
    ret.f[k] += second.f[k];
  });

  Object.keys(second.b).forEach(function (k) {
    const retArray = ret.b[k];
    const secondArray = second.b[k];
    for (let i = 0; i < retArray.length; i += 1) {
      retArray[i] += secondArray[i];
    }
  });

  return ret;
}

/**
 * 合并两个覆盖对象，确保执行计数正确。
 *
 * @method mergeCoverage
 * @static
 * @param {Object} first 第一个覆盖对象
 * @param {Object} second 第二个覆盖对象
 * @return {Object} 合并后的结果对象。请注意，输入对象不会被修改。
 */
function mergeCoverage(first, second) {
  if (!second) {
    return first;
  }

  const mergedCoverage = JSON.parse(JSON.stringify(first)); // 深拷贝 coverage，这样修改出来的是两个的合集
  Object.keys(second).forEach(function (filePath) {
    const original = first[filePath];
    const added = second[filePath];
    let result;

    if (original) {
      result = mergeFileCoverage(original, added);
    } else {
      result = added;
    }

    mergedCoverage[filePath] = result;
  });

  return mergedCoverage;
}

const fs=require('fs');
const first=fs.readFileSync('./data/first.json','utf-8');
const second=fs.readFileSync('./data/second.json','utf-8');
console.log(JSON.parse(first)["/builds/canyon/canyon-demo/src/pages/Home.tsx"]['s']);
console.log(mergeCoverage(JSON.parse(first),JSON.parse(second))["/builds/canyon/canyon-demo/src/pages/Home.tsx"]['s']);
```



## 聚合

端到端测试的覆盖率数据特点之一是单体数据体积大，在项目整体插桩的情况下相当于整体源代码体积的30%，并且携程Trip.com flight站点的预定页UI自动化case触发visibilitychange的次数可达到近千次，对于单条数据大且高频次的数据上报场景，很难做到实时数据聚合计算。Canyon采用消息队列的形式来消费聚合数据。并且设计成无状态服务，适用于云原生时代的容器化部署，可通过HPA弹性伸缩容来应用不同场景下的测试覆盖率上报。



## 报告

对于覆盖率报告展示，我们沿用了istanbul-report的界面风格，但是由于istanbul-report只提供了静态html文件的生成，不适合现代化前端水合数据生成html的模式，为此我们参考了他的源码，使用了shiki（语法高亮器）标记源代码覆盖率，根据istanbul- report的标记方法。

## 变更代码覆盖率

对于变更代码覆盖率，我们统计的公式是 覆盖到的新增代码行/所有新增代码行，通过配置compareTarget来指定对比目标，再通过变更代码行的获取我们通过gitlab的接口，和jsdiff可以计算得出sha和compareTarget对比出来每个文件的变更行的行号，再通过 覆盖到的新增代码行/所有新增代码行 的公式计算。

```js
//计算代码
```



## 社区推广

从这篇文章发表时起，我们将正式开源Canyon项目。JavaScript是时下最流行的编程语言，但是端到端测试覆盖率收集领域一直空白，我们的代码开发基于了Istanbuljs，shiki等优秀开源项目，我们有信心推出canyon开源可以赢得社区的反响，并且可以有大量js开发者参与进来。

我们基于的时istanbuljs的代码插桩，对于javascript语言领域而言，istanbuljs可谓是久经沙场，如果可以投入生产实践插桩收集，将会在代码质量优化方面提供非常重要的指标参考。

Canyon在未来还有很大发展空间，例如生产环境插桩收集还未验证，与playwright puppeteer cypress等自动化测试的工具还没有深度链接，这些都已经规划到了未来的开发计划中。希望在未来Canyon可以在携程及社区有更多人参与建设。

## 参考链接

**开源项目 Canyon：**

https://github.com/canyon-project/canyon

**JavaScript覆盖率工具：**

https://github.com/istanbuljs/istanbuljs

**Shiki美观而强大的语法高亮器：**

https://github.com/shikijs/shiki

**JavaScript 文本差异：**

https://github.com/kpdecker/jsdiff

**"An O(ND) Difference Algorithm and its Variations" (Myers, 1986).**

http://www.xmailserver.org/diff2.pdf
