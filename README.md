# 开源 | Canyon: 提升JavaScript代码质量的全面覆盖率分析工具

## 背景

[istanbuljs](https://github.com/istanbuljs/istanbuljs) 是一款优秀的JavaScript代码覆盖率工具，主要用于单元测试的代码覆盖率检测和生成本地覆盖率报告。然而，随着现代前端技术和UI自动化测试的发展，对端到端测试的代码覆盖率检测需求逐渐增加，istanbuljs提供的功能显得捉襟见肘。  

在携程内部JavaScript代码覆盖率使用的是gitlab内置的coverage上报，也是只支持单元测试的覆盖率收集和概览数据展示。随着携程的前端技术日益精进，我们有了自己的前端流量录制平台，并且部署了相当大规模的模拟器集群进行UI自动化[(flybirds)](https://github.com/ctripcorp/flybirds)回放。这种场景下，需要对端到端测试的代码覆盖率进行收集和展示，以便开发同学更好的了解到自己的代码质量。

传统的istanbuljs提供的功能已经无法满足我们的需求。我们需要处理UI自动化过程中来自前端高并发的覆盖率上报，实时的覆盖率聚合，以及覆盖率数据的水合展示。因此，我们在Istanbuljs的基础上开发了Canyon的来解决端到端测试覆盖率难收集的问题。

目前，携程的多个部门已经开始使用Canyon，并在持续集成流水线构建阶段插入探针代码，在UI自动化测试阶段收集和上报覆盖率数据。服务端实时生成详尽的覆盖率报告，为UI自动化测试用例提供全面的覆盖率数据指标。

## 介绍

Canyon 通过简单的 Babel 插件配置即可实现代码插装、覆盖率上报和实时报告生成。其技术栈完全基于 JavaScript，只需 Node.js 环境即可运行，部署方便，适用于云原生环境的部署（如 Docker、Kubernetes）。

应用的架构设计适用于处理高频、大规模的覆盖率数据上报，能够应对 UI 自动化测试中的各种场景。同时，Canyon 与现有的 CI/CD 工具（如 GitLab CI、Jenkins）无缝集成，使用户能够轻松地在持续集成流水线中使用。

架构图如下：

![img_1.png](./architecture.png)

下面我会根据以下几个部分来介绍 Canyon 的主要功能：

1. 代码覆盖率
2. 代码插桩
3. 测试与上报
5. 覆盖率聚合
5. 覆盖率报告
6. 变更代码覆盖率
7. react native 覆盖率收集方案
8. 覆盖率提升优先级列表

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
  f: [1],
  s: [1, 1, 1]
}
```

这个测试用例覆盖率达到了100%，每个函数和每个语句都至少执行了一次。但是在实际应用中，要达到100%的代码覆盖率需要多次测试。

这是覆盖率的基本介绍，有了这个前置知识，方便大家理解下面的内容。

## 代码插桩（instrumenting-code）

__代码覆盖率最重要的一环就是代码插桩__

istanbuljs 是久经沙场的js代码插桩黄金标准。Canyon主要为端到端测试提供解决方案，经过大量的实验验证，现代化前端工程的覆盖率插桩必须要编译时插桩。具体原因是istanbuljs提供的nyc插桩工具只能对原生js进行插桩，然而前端模版语法层出不穷，例如ts、tsx、vue，虽然nyc也可以插桩，但是结构实践证明直接插桩的覆盖率效果不尽人意，无法精确到该插桩到的函数、语句、分支。幸运的是经过调研，我们发现了[babel-plugin-istanbul](https://github.com/istanbuljs/babel-plugin-istanbul)、vite-plugin-istanbul（experimental）、swc-plugin-coverage-instrument(experimental)。等类型工程的插桩解决方案。这些方案无一例外都是在前端工程编译阶段在将代码分析成ast抽象语法树的时候在适当时机进行插桩方法调用，更精确的插桩到的函数、语句、分支。

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


![img_2.png](./img_3.png)

为了紧密关联插桩代码的源代码，我们适配了各种provider，将环境变量发送到Canyon服务端，兑换到reportID，方便覆盖率数据聚合计算完成后的覆盖率源文件的关联展示。

我们还提供了babel-plugin-canyon的babel插件，可以在各种流水线内（aws，gitlab ci）读取环境变量(branch、sha)，以供后续覆盖率数据与对应的gitlab源代码关联。

babel.config.js

```javascript
module.exports = {
  plugins: [
    [
        'babel-plugin-canyon',
        {
          provider: 'gitlab',
          branch: process.env.CI_COMMIT_REF_NAME,
          sha: process.env.CI_COMMIT_SHA,
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


## 测试与上报

当插桩完成发布到测试环境后，我们就可以进行测试了。拿playwright举例，对于插桩成功的前端应用站点，window对象上面都会挂载__coverage__和__canyon__对象，我们需要在playwright测试过程中收集并上报这些数据到canyon的服务端。

[playwright](https://playwright.dev/)示例：

```js
const {chromium} = require('playwright');
const main = async () => {
  const browser = await chromium.launch()
  const page = await browser.newPage();
  // 进入被测页面
  await page.goto('http://test.com')
  // 执行测试用例
  // 用例1
  await page.click('button')
  // 用例2
  await page.fill('input', 'test')
  // 用例3
  await page.click('text=submit')
  const coverage = await page.evaluate(`window.__coverage__`)
  // 收集上报覆盖率
  upload(coverage)
  browser.close()
}

main()

```

携程内部有自己的UI自动化平台 [flybirds](https://github.com/ctripcorp/flybirds)，我们在flybirds内部集成了Canyon覆盖率数据的收集和上报。真实的浏览器UI自动化测试的覆盖率收集场景较为复杂，主要体现在多页面（MPA）的覆盖率收集时机不确定性。

### 单页面（SPA）与多页面（MPA）

当测试用例执行完成后，对于单页面应用(SPA)或者多页面应用而言，上报步骤是将页面window对象上的__coverage__对象上报到Canyon服务端，对于单页面应用来说，相对来说比较简单，在所有测试内容都在单页面应用内，覆盖率数据会常驻在window对象中，对于多页面应用而言，路由的跳转会导致window对象的重制，丢失coverage对象。所以这个时机是至关重要的，经过大量实践验证，我们找到了浏览器的onvisiblechange方法

- visibilitychange

在浏览器可见性改变的时候上报覆盖率数据，值得一提的是，对于visibilitychange这种可能会导致重复数据上报，但是对于覆盖率统计来说，未执行到的代码多次合并来说不会影响覆盖率的具体指标数据统计。

- fetchLater

Chrome 浏览器正在积极引入一个革命性的 JavaScript API——[fetchLater()](https://developer.chrome.com/blog/fetch-later-api-origin-trial)。这个全新的 API 旨在彻底简化关闭页面时的数据发送过程，确保即使在页面关闭后或用户离开的情况下，请求也能在未来某个时刻被安全、可靠地发出。

这个API的推出时令人振奋的，可以很好的解决多页面（MPA）收集难的问题，只需要在浏览器关闭时收集。

> 注：fetchLater() 已在 Chrome 中提供，用于在版本 121（2024 年 1 月发布）开始的原始试验中供真实用户测试，该试验将持续到 Chrome 126（2024 年 7 月）。

## 聚合

覆盖率数据的来源是同一版本的代码，覆盖率数据是可以聚合的，Canyon内部使用reportID来关联测试用例和细分聚合维度。这样做可以让海量的覆盖率数据聚合成有限个，即Case的数量。

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
```

端到端测试的覆盖率数据特点之一是单体数据体积大，在项目整体插桩的情况下相当于整体源代码体积的30%。携程Trip.com flight站点的预定页UI自动化case上报次数每次可达2000次，每次10M数据，这样的数据量对于Canyon服务端来说是一个巨大的挑战。

对于单条数据大且高频次的数据上报场景，很难做到实时数据聚合计算。 Canyon采用消息队列的形式来消费数据，并且设计成无状态服务，适用于云原生时代的容器化部署，可通过HPA弹性伸缩容来应用不同场景下的测试覆盖率上报。


## 报告

对于覆盖率报告展示，我们沿用了istanbul-report的界面风格，但是由于istanbul-report只提供了静态html文件的生成，不适合现代化前端水合数据生成html的模式，为此我们参考了它的源码，使用了monaco-editor标记源代码覆盖率。

```js
  const decorations = useMemo(() => {
    if (data) {
        const annotateFunctionsList = annotateFunctions(data.coverage, data.sourcecode);
        const annotateStatementsList = annotateStatements(data.coverage);
        return [...annotateStatementsList, ...annotateFunctionsList].map((i) => {
            return {
                inlineClassName: 'content-class-found',
                startLine: i.startLine,
                startCol: i.startCol,
                endLine: i.endLine,
                endCol: i.endCol,
            };
        });
    } else {
        return [];
    }
}, [data]);
```

经过着色后的效果：

![img_2.png](./img_4.png)


## 变更代码覆盖率

对于变更代码覆盖率，我们统计的公式是 覆盖到的新增代码行/所有新增代码行。

我们通过配置compareTarget来指定对比目标，再联合gitlab的git diff接口获取变更代码行结合覆盖率数据计算。
```js
/**
 * returns computed line coverage from statement coverage.
 * This is a map of hits keyed by line number in the source.
 */
function getLineCoverage(statementMap:{ [key: string]: Range },s:{ [key: string]: number }) {
  const statements = s;
  const lineMap = Object.create(null);

  Object.entries(statements).forEach(([st, count]) => {
    if (!statementMap[st]) {
      return;
    }
    const { line } = statementMap[st].start;
    const prevVal = lineMap[line];
    if (prevVal === undefined || prevVal < count) {
      lineMap[line] = count;
    }
  });
  return lineMap;
}

```

## react native 覆盖率收集方案

携程的移动端技术栈主要是react native，好消息是对于我们的插桩方案一样适用，因为都是基于babel编译。并且得力于得力于公司内部的react native项目结构统一，我们将编译时插桩做到了流水线中，在流水线中分别打包“正常包”和”插桩包“，这样搭配UI自动化可以形成一套完整的录制回放覆盖率指标收集的测试体系。

利用websocket暴露模拟器内覆盖率数据：

```js
// 创建WebSocket连接
const socket = new WebSocket('ws://localhost:8080');

// 当WebSocket连接打开时触发
socket.onopen = () => {
    console.log('Connected to coverage WebSocket server');
};

// 当收到WebSocket消息时触发
socket.onmessage = event => {
    try {
      if (JSON.parse(event.data).type === 'getcoverage') {
        // 发送覆盖率数据
        socket.send(JSON.stringify(payload));
      }
    } catch (e) {
      console.log(e);
    }
};

// 当WebSocket连接关闭时触发
socket.onclose = () => {
    console.log('Disconnected from coverage WebSocket server');
};
```

目前携程机票部门的APP模块均已接入Canyon，经过实践istanbuljs可以很好的对其进行插桩及覆盖率数据收集，测试团队在每次生产发布前会以Canyon的覆盖率数据指标来衡量此次发布的质量情况。

## 覆盖率提升优先级列表

刚开始接入Canyon的时候，对于大型应用的覆盖率数据，如果缺乏大量UI自动化case覆盖率数据是非常低的。对于整体的覆盖率把控，一开始直接出个“istanbul”覆盖率报告大家很容易迷茫，不知道何从入手。

后来经过调研，发现公司有专门的生产覆盖率函数收集系统。然后我们结合自己的覆盖率数据，决定做一个《覆盖率提升优先级列表》，以此来提示开发同学提升覆盖率的方向。

总而言之，我们定义了一个覆盖率权重公式：生产覆盖率*100*0.3+(1-测试覆盖率)*100*0.3+文件函数数量*0.2。优先推荐提升文件体积大，生产覆盖率高且测试覆盖率低的文件覆盖率。


## 社区推广

从这篇文章发表时起，我们将正式开源Canyon。JavaScript是时下最流行的编程语言，但是端到端测试覆盖率收集领域一直空白，我们的代码开发基于了istanbuljs，monaco editor等优秀开源项目，我们有信心推出Canyon开源可以赢得社区的反响，并且可以有大量JavaScript开发者参与进来。

Canyon在未来还有很大发展空间，例如生产环境插桩收集还未有待验证尝试，与playwright、puppeteer、cypress等自动化测试的工具还没有深度链接，这些都已经规划到了未来的开发计划中。希望在未来Canyon可以在携程及社区里有更多人参与建设。

## 参考链接

**开源项目 Canyon：**

https://github.com/canyon-project/canyon

**JavaScript覆盖率工具：**

https://github.com/istanbuljs/istanbuljs

**基于浏览器的代码编辑器：**

https://github.com/microsoft/monaco-editor

**JavaScript文本差异：**

https://github.com/kpdecker/jsdiff

**"An O(ND) Difference Algorithm and its Variations" (Myers, 1986).**

http://www.xmailserver.org/diff2.pdf
