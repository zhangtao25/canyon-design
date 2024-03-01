# 开源 | Canyon: JavaScript代码覆盖率解决方案

istanbuljs是当下最优秀的JavaScript覆盖率工具，但是它偏低层，只提供了代码探针插装，静态html报告生成的的功能。2024年前端技术井喷式发展，istanbuljs提供的功能显得抽筋见肘。为此，我们在istanbuljs的基础上开发了一套JavaScript覆盖率解决方案Canyon，拥有处理高并发上报的覆盖率，实时覆盖率聚合，覆盖率报告水合展示的拥有等特性。携程机票、IBU、酒店、商旅等部门都以初具规模的使用，保证了UI自动化的提供了覆盖率数据指标的支持。

## 架构

下图说明了 Canyon 的架构：

Canyon的整体技术栈完全基于nodejs(前端、后端、任务)，部署仅需要nodejs环境，也适用于云原生环境部署(docker、Kubernetes)。应用整体流程为：代码探针插桩、触发器触发探针、覆盖率数据上报、消息生成覆盖率概览、覆盖率报告呈现。应用的架构设计适用于高频、大体积覆盖率数据的上报，使用分布式部署，消息队列消费。

## 一、代码插桩



参考 https://github.com/cypress-io/code-coverage ！！！！从头开始写



ts-loader，必须打开sourceMap

sourceMap上传上去，未来开启生产插桩，减少代码探针数量





经过我们大量的实验验证，前端工程的覆盖率插桩必须要编译时插桩，具体原因是istanbuljs提供的nyc插桩工具只能对原生js进行插桩，然而前端模版语言层出不穷，例如ts、tsx、vue我们需要在工程构建时进行探针插桩。进过调研，我们发现了[babel-plugin-istanbul](https://github.com/istanbuljs/babel-plugin-istanbul)、vite-plugin-istanbul（experimental）、swc-plugin-coverage-instrument(experimental)。等类型工程的插桩解决方案。

我们还提供了babel-plugin-canyon的babel插件，可以在各种流水线内（aws，gitlab ci）读取环境变量(branch、sha)，以供后续覆盖率数据与对应的gitlab源代码关联。provider，提供商

需要特别注意的是，代码探针的插桩会在构建产物上下文加上代码探针，会是代码整体产物增大30%，建议不要上生产环境。

以下是经过代码插桩的代码

说明适配不同provider提供商

## 二、触发器

当代码插桩完成以后其实已经完成了一大步，第二步即触发器（case）的选择，我们的canyon解决方案主要面对的是端到端的测试，所以可以选择 cypress.io之类的端到端测试工具来完成覆盖率代码探针的触发。携程拥有自己的UI自动化平台，在此过程中充当触发器的角色，代码覆盖率也是用来反映UI自动化case的重要指标，我们通过录制生产环境的case，然后在flytest上进行case的调试，最后再在测试环境回放对应UI自动化。

为了区分覆盖率的触发来源，我们引入了report id的概念。每次上报覆盖率数据时，都会带上一个 report id，这个 report id 是由系统 生成的，我们设计的本意是想让您使用它来区分自己的 case，如果您不是使用它来区分 case，我们会以 commitsha 作为report id，所有 此次 commitsha 的覆盖率数据都会往这个 report id 上累加。



## 三、上报

值得一提的是，进过babel-plugin-istanbul插桩的项目的覆盖率数据全部存在了window对象上，上报就是将其提取出来。我们UI自动化使用的工具是playright，可以在使用脚本轻松的将其提取出来上报，当然与之一起上报的还有流水线中获取到的sha变量

对于单页面应用我们其实比较好进行上报，因为他的对象不会销毁，但是对于一些非单页面应用，我们需要在其visiblechange的时候，监听这个事件，在这个钩子里进行覆盖率数据的上报。



## 四、聚合

携程的UI自动化case对于一些复杂页面有非常大量的case，就Trip.com的case来说，每次基本上都有2000次的覆盖率上报，我们需要对大量的case进行分类聚合，我们还有crn的覆盖率数据，那就更大了，基本是项目体积的1/3，而且crn是单页面类型的应用，每个覆盖率都有 30m那么大，由于本人才疏学浅，无法做到实时的大批量的大数据聚合计算，于是我想到了使用消息队列的形式来慢慢消费这些数据，确保数据能在1分钟以内生成。



## 五、报告

对于istanbul的报告模块研究发现，他只提供了静态html文件的版本，我们需要的水合版本，于是我们研究istanbul- report模块，照着他的样子开发了水合版本，他可以结合标准的istanbul覆盖率数据水合生成对应的report详情页。

## 变更代码

通过compareTarget，然后通过gitlab提供的源码来回溯。

## react native

携程的主要技术栈是react native，我们与携程的平台合作，提供了标准化的模版工程修改脚本，在打包阶段修改源代码，安装好babel插件，这样就可以打出带有代码探针的包，这样，拥有UI自动化的项目就可以轻松的获取到UI自动化的覆盖率数据，一些手工测试也可以大致的统计处覆盖率数据，对于测试的有一定意义的指导



## 生产环境插桩

未来如果经过大量验证需要探寻出一套插桩代码上生产的方案。上生产的意义在于覆盖率数据触发者变成了真实用户，采集到的覆盖率数据更有意义，对分析代码有重大意义，例如做一些代码优化，减少无用代码，梳理代码逻辑等。



## 六、社区推广

从这篇文章发表时起，我们将正式开源Canyon项目。JavaScript是时下最流行的编程语言，但是端到端测试覆盖率收集领域一直空白，我们的代码开发基于了Istanbuljs，shiki等优秀开源项目，我们有信心推出canyon开源可以赢得社区的反响，并且可以有大量js开发者参与进来。

我们基于的时istanbuljs的代码插桩，对于javascript语言领域而言，istanbuljs可谓是久经沙场，如果可以投入生产实践插桩收集，将会在代码质量优化方面提供非常重要的指标参考。

Canyon在未来还有很大发展空间，例如生产环境插桩收集还未验证，与playwright puppeteer cypress等自动化测试的工具还没有深度链接，这些都已经规划到了未来的开发计划中。希望在未来Canyon可以在携程及社区有更多人参与建设。



## 七、参考链接

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