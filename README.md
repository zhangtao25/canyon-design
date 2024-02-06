# 开源 | Canyon: JavaScript代码覆盖率解决方案

istanbuljs是当下最优秀的JavaScript覆盖率工具，但是它偏低层，只提供了代码探针插装，静态html报告生成的的功能。24年前端技术井喷式发展，istanbuljs提供的功能显得抽筋见肘。为此，我们在istanbuljs的基础上开发了一套JavaScript覆盖率解决方案Canyon，拥有处理高并发上报的覆盖率，实时覆盖率聚合，覆盖率报告水合展示的拥有等特性。携程机票、IBU、酒店、商旅等部门都以初具规模的使用，保证了UI自动化的提供了覆盖率数据指标的支持。

## 架构

下图说明了 Canyon 的架构：

![](./architecture.png)

## 工程代码插桩

经过我们大量的实验验证，前端工程的覆盖率插桩必须要编译时插桩，具体原因是istanbuljs提供的nyc插桩工具只能对原生js进行插桩，然而前端模版语言层出不穷，例如ts、tsx，我们需要在工程构建时进行探针插桩。进过调研，我们发现了[babel-plugin-istanbul](https://github.com/istanbuljs/babel-plugin-istanbul)、vite-plugin-istanbul（experimental）、swc-plugin-coverage-instrument(experimental)。等类型工程的插桩解决方案。

我们还提供了babel-plugin- canyon的babel插件，可以在各种流水线内（aws，gitlab ci）读取环境变量(branch、sha)，以供后续覆盖率数据与对应的gitlab源代码关联。

需要特别注意的是，代码探针的插桩会在构建产物上下文加上代码探针，会是代码整体产物增大30%，建议不要上生产环境。

以下是经过代码插桩的代码

```js
//插桩前
function add(a,b) {
    if (a<b){
        return a;
    }
    return a + b;
}
//插桩后
function cov_qxoryl6wa() {
    var path = "/Users/zhangtao/Desktop/farm-project/add.js";
    var hash = "9a265619447962c8591309785f0f73485412db47";
    var global = new Function("return this")();
    var gcv = "__coverage__";
    var coverageData = {
        path: "/Users/zhangtao/Desktop/farm-project/add.js",
        statementMap: {
            "0": {start: {line: 2, column: 4}, end: {line: 4, column: 5}},
            "1": {start: {line: 3, column: 8}, end: {line: 3, column: 17}},
            "2": {start: {line: 5, column: 4}, end: {line: 5, column: 17}}
        },
        fnMap: {
            "0": {
                name: "add",
                decl: {start: {line: 1, column: 9}, end: {line: 1, column: 12}},
                loc: {start: {line: 1, column: 18}, end: {line: 6, column: 1}},
                line: 1
            }
        },
        branchMap: {
            "0": {
                loc: {start: {line: 2, column: 4}, end: {line: 4, column: 5}},
                type: "if",
                locations: [{start: {line: 2, column: 4}, end: {line: 4, column: 5}}, {
                    start: {line: 2, column: 4},
                    end: {line: 4, column: 5}
                }],
                line: 2
            }
        },
        s: {"0": 0, "1": 0, "2": 0},
        f: {"0": 0},
        b: {"0": [0, 0]},
        _coverageSchema: "1a1c01bbd47fc00a2c39e90264f33305004495a9",
        hash: "9a265619447962c8591309785f0f73485412db47"
    };
    var coverage = global[gcv] || (global[gcv] = {});
    if (!coverage[path] || coverage[path].hash !== hash) {
        coverage[path] = coverageData;
    }
    var actualCoverage = coverage[path];
    {// @ts-ignore
        cov_qxoryl6wa = function () {
            return actualCoverage;
        };
    }
    return actualCoverage;
}

cov_qxoryl6wa();

function add(a, b) {
    cov_qxoryl6wa().f[0]++;
    cov_qxoryl6wa().s[0]++;
    if (a < b) {
        cov_qxoryl6wa().b[0][0]++;
        cov_qxoryl6wa().s[1]++;
        return a;
    } else {
        cov_qxoryl6wa().b[0][1]++;
    }
    cov_qxoryl6wa().s[2]++;
    return a + b;
}
```

## 二、触发器

当代码插桩完成以后其实已经完成了一大步，第二步即触发器（case）的选择，我们的canyon解决方案主要面对的是端到端的测试，所以可以选择 cypress.io 之类的端到端测试工具来完成覆盖率代码探针的触发。携程拥有自己的UI自动化平台，在此过程中充当触发器的角色，代码覆盖率也是用来反映UI自动化case的重要指标，我们通过录制生产环境的case，然后在flytest上进行case的调试，最后再在测试环境回放对应UI自动化。

```js
const playwright = require('playwright');

(async () => {
  const browser = await playwright.chromium.launch();
  const context = await browser.newContext();
  const page = await context.newPage();
  await page.goto('http://example.com');

  // Access window variable
  const windowVariable = await page.evaluate(() => window.__coverage__);

  console.log(windowVariable);

  await browser.close();
})();
```





## 三、上报

值得一提的是，进过babel-plugin-istanbul插桩的项目的覆盖率数据全部存在了window对象上，上报就是将其提取出来。我们UI自动化使用的工具是playright，可以在使用脚本轻松的将其提取出来上报，当然与之一起上报的还有流水线中获取到的sha变量

对于单页面应用我们其实比较好进行上报，因为他的对象不会销毁，但是对于一些非单页面应用，我们需要在其visiblechange的时候，监听这个事件，在这个钩子里进行覆盖率数据的上报。

```js
document.addEventListener('visibilitychange', function() {
  if (document.visibilityState === 'visible') {
    console.log('Page is now visible');
  } else if (document.visibilityState === 'hidden') {
    console.log('Page is now hidden');
  }
});
```



## 四、聚合

携程的UI自动化case对于一些复杂页面有非常大量的case，就Trip.com的case来说，每次基本上都有2000次的覆盖率上报，我们需要对大量的case进行分类聚合，我们还有crn的覆盖率数据，那就更大了，基本是项目体积的1/3，而且crn是单页面类型的应用，每个覆盖率都有 30m那么大，由于本人才疏学浅，无法做到实时的大批量的大数据聚合计算，于是我想到了使用消息队列的形式来慢慢消费这些数据，确保数据能在1分钟以内生成。

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
```



## 五、报告

对于istanbul的报告模块研究发现，他只提供了静态html文件的版本，我们需要的水合版本，于是我们研究istanbul- report模块，照着他的样子开发了水合版本，他可以结合标准的istanbul覆盖率数据水合生成对应的report详情页。

## 变更代码

通过compareTarget，然后通过gitlab提供的源码来回溯。

## react native

携程的主要技术栈是react native，我们与携程的平台合作，提供了标准化的模版工程修改脚本，在打包阶段修改源代码，安装好babel插件，这样就可以打出带有代码探针的包，这样，拥有UI自动化的项目就可以轻松的获取到UI自动化的覆盖率数据，一些手工测试也可以大致的统计处覆盖率数据，对于测试的有一定意义的指导



## 生产环境插桩

未来如果经过大量验证需要探寻出一套插桩代码上生产的方案。上生产的意义在于覆盖率数据触发者变成了真实用户，采集到的覆盖率数据更有意义，对分析代码有重大意义，例如做一些代码优化，减少无用代码，梳理代码逻辑等。