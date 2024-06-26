# 开源 | Canyon: JavaScript代码覆盖率解决方案

istanbuljs是当下最优秀的JavaScript覆盖率工具，但是它偏低层，只提供了代码探针插装，静态html报告生成。24年前端技术井喷式发展，istanbuljs提供的功能显得抽筋见肘。为此，我们在istanbuljs的基础上开发了一套javascript覆盖率解决方案Canyon，拥有xx，xx等特性。已在携程内部稳定运行。

调研

istanbuljs在单元测试方面已经做到非常完善，也内置在大多数的单元测试框架中。我所处的机票部门，基础设施完善，有独立的UI自动化测试平台，和前端用户操作的录制回放平台，为我们提供了大量的UI自动化case，所以需求就应运而生了，覆盖率的触发源于触发器，那么UI自动化case就充当了这个角色，录制的大量UI自动化case我们给它们分门别类，做好标记，在运行case自动化case的时候对运行的代码进行覆盖率插装，那么就可以产生覆盖率收集，再构建一个平台来收集上报，聚合，就可以统计的到对应UI自动化的case覆盖率，同时需要对每个case上报的覆盖率数据打上标签，这样可以回溯对应的覆盖率数据



## 架构

下图说明了 Canyon 的架构：







## 工程代码插桩

进过我们进行了大量的实验，前端工程的覆盖率插桩必须要编译时插桩，具体原因是istanbuljs提供的nyc插桩工具只能对原生js进行插桩，然而前端模版语言层出不穷，例如ts、tsx，我们需要在工程构建时进行探针插桩。进过调研，我们发现了[babel-plugin-istanbul](https://github.com/istanbuljs/babel-plugin-istanbul)、vite-plugin-istanbul（experimental）、swc-plugin-coverage-instrument(experimental)。等类型工程的插桩解决方案。

我们还提供了babel-plugin- canyon的babel插件，可以在各种流水线内（aws，gitlab ci）读取环境变量(branch、sha)，以供后续覆盖率数据与对应的gitlab源代码关联。

需要特别注意的是，代码探针的插桩会在构建产物上下文加上代码探针，会是代码整体产物增大30%，建议不要上生产环境。

```js
var cov_ac7rkuoyv = function () {
  var path = "/Users/test/shenlvmeng/nyc-demo/src/App.js";
  var hash = "7dec600464f484deef063d183319f809a7c25687";
  var global = new Function("return this")();
  var gcv = "__coverage__";
  var coverageData = {
    path: "/Users/shenlvmeng/nyc-demo/src/App.js",
    statementMap: {
      "0": {
        start: {
          line: 8,
          column: 2
        },
        end: {
          line: 14,
          column: 9
        }
      }
      // ...
    },
    fnMap: {
      "0": {
        name: "App",
        decl: {
          start: {
            line: 7,
            column: 9
          },
          end: {
            line: 7,
            column: 12
          }
        },
        loc: {
          start: {
            line: 7,
            column: 15
          },
          end: {
            line: 33,
            column: 1
          }
        },
        line: 7
      },
      // ...
    },
    branchMap: {},
    s: {
      "0": 0,
      // ...
    },
    f: {
      "0": 0,
      // ...
    },
    b: {},
    _coverageSchema: "43e27e138ebf9cfc5966b082cf9a028302ed4184",
    hash: "7dec600464f484deef063d183319f809a7c25687"
  };
  var coverage = global[gcv] || (global[gcv] = {});

  if (coverage[path] && coverage[path].hash === hash) {
    return coverage[path];
  }

  return coverage[path] = coverageData;
}();

var _jsxFileName = "/Users/test/shenlvmeng/nyc-demo/src/App.js";

function App() {
  cov_ac7rkuoyv.f[0]++;
  cov_ac7rkuoyv.s[0]++;
  Object(react__WEBPACK_IMPORTED_MODULE_0__["useEffect"])(() => {
    cov_ac7rkuoyv.f[1]++;
    cov_ac7rkuoyv.s[1]++;

    (async () => {
      cov_ac7rkuoyv.f[2]++;
      cov_ac7rkuoyv.s[2]++;
      console.log(window.__coverage__);
      cov_ac7rkuoyv.s[3]++;
      axios__WEBPACK_IMPORTED_MODULE_1___default.a.defaults.headers.post['Access-Control-Allow-Origin'] = '*';
      cov_ac7rkuoyv.s[4]++;
      axios__WEBPACK_IMPORTED_MODULE_1___default.a.post('http://localhost:4000/coverage/client', window.__coverage__);
    })();
  }, []);
  cov_ac7rkuoyv.s[5]++;
  return react__WEBPACK_IMPORTED_MODULE_0___default.a.createElement("div", {
    className: "App",
    __source: {
      fileName: _jsxFileName,
      lineNumber: 16
    },
    __self: this
  }, react__WEBPACK_IMPORTED_MODULE_0___default.a.createElement("header", {
    className: "App-header",
    __source: {
      fileName: _jsxFileName,
      lineNumber: 17
    },
    __self: this
  }, react__WEBPACK_IMPORTED_MODULE_0___default.a.createElement("img", {
    src: _logo_svg__WEBPACK_IMPORTED_MODULE_2___default.a,
    className: "App-logo",
    alt: "logo",
    __source: {
      fileName: _jsxFileName,
      lineNumber: 18
    },
    __self: this
  }), react__WEBPACK_IMPORTED_MODULE_0___default.a.createElement("p", {
    __source: {
      fileName: _jsxFileName,
      lineNumber: 19
    },
    __self: this
  }, "Edit ", react__WEBPACK_IMPORTED_MODULE_0___default.a.createElement("code", {
    __source: {
      fileName: _jsxFileName,
      lineNumber: 20
    },
    __self: this
  }, "src/App.js"), " and save to reload."), react__WEBPACK_IMPORTED_MODULE_0___default.a.createElement("a", {
    className: "App-link",
    href: "https://reactjs.org",
    target: "_blank",
    rel: "noopener noreferrer",
    __source: {
      fileName: _jsxFileName,
      lineNumber: 22
    },
    __self: this
  }, "Learn React")));
}
```

## 二、触发器

当代码插桩完成以后其实已经完成了一大步，第二步即触发器（case）的选择，我们的canyon解决方案主要面对的是端到端的测试，所以可以选择 cypress.io 之类的端到端测试工具来完成覆盖率代码探针的触发。携程拥有自己的UI自动化平台，在此过程中充当触发器的角色，代码覆盖率也是用来反映UI自动化case的重要指标，我们通过录制生产环境的case，然后在flytest上进行case的调试，最后再在测试环境回放对应UI自动化。

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