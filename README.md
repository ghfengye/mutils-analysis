# 从零搭建自己的js工具库 typescript+rollup+karma+mocha+coverage

## 前言

随着公司产品线的增多，开发维护的项目也越来越多，在业务开发过程中，就会发现经常用到的cookie处理，数组处理，节流防抖函数等工具函数，这些工具函数在很多的项目中会使用到，为了避免一份代码多次复制粘贴使用的low操作，笔者尝试从零搭建JavaScript工具库**typescript+rollup+karma+mocha+coverage** , 写这篇文章主要是分享给有同样需求的朋友提供参考，希望对你有所帮助。

***项目源码在文章结尾处，记得查收哦~***

## 目录结构说明

```
├── scripts ------------------------------- 构建相关的文件
│   ├── config.js ------------------------- 生成rollup配置的文件
│   ├── build.js -------------------------- 对 config.js 中所有的rollup配置进行构建
├── coverage ---------------------------------- 测试覆盖率报告
├── dist ---------------------------------- ts编译后文件的输出目录
├── lib   ---------------------------------- 构建后后文件的输出目录
├── test ---------------------------------- 包含所有测试文件
│   ├── index.ts --------------------------自动化单元测试入口文件
│   ├── xx.spec.ts ------------------------------ 单元测试文件
├── src ----------------------------------- 工具函数源码
│   ├── entry-compiler.ts -------------------------- 函数入口文件
│   ├── arrayUtils ------------------------------ 存放与数组处理相关的工具函数
│   │   ├── arrayFlat.ts ---------------------- 数组平铺
│   ├── xx ------------------------------ xx
│   │   ├── xxx.ts ----------------------xxx
├── package.json  ----------------------------- 配置文件
├── package-lock.json ----------------------------- 锁定安装包的版本号
├── index.d.ts ------------------------- 类型声明文件
├── karma.conf.js ------------------------- karma配置文件
├── .babelrc ------------------------------ babel 配置文件
├── tsconfig.json ----------------------------- ts 配置文件
├── tslint.json ----------------------------- tslint 配置文件
├── .npmignore -------------------------  npm发包忽略配置
├── .gitignore ---------------------------- git 忽略配置

```

目录结构会随着时间迭代，建议查看[库](https://github.com/ghfengye/mutils.git)上最新的目录结构


## 构建打包

#### 该选用何种构建工具？

目前社区有很多的构建工具，不同的构建工具适用场景不同，Rollup是一个js模块打包器，可以将小块代码编译成复杂的代码块，偏向应用于js库，像vue，vuex，dayjs等优秀的开源项目就是使用rollup，而webpack是一个js应用程序的静态模块打包器，适用于场景中涉及css、html，复杂的代码拆分合并的前端工程，如element-ui。

> 简单来说就是，在开发应用时使用webpack，开发库时使用Rollup

*如果对Rollup还不熟悉，建议查看[Rollup官网文档](https://www.rollupjs.com)*


#### 如何构建？

主要说明下项目中config.js和script/build.js的构建过程

> 第一步，构建全量包，在cofig.js配置后，有两种方式打包：
>  1.  package.json的script字段自定义指令打包指定格式的包并导出到lib下
>  2.  在build.js获取config.js导出rollup配置，通过rollup一次性打包不同格式的包并保存到lib文件夹下


##### 自定义打包
在config.js配置umd，es，cjs格式，及压缩版min的全量包，*对于包umd/esm/cjs不同格式之间的区别请移步 [
JS 模块化规范](
https://qwqaq.com/b8fd304a.html)*
```
......
......
const builds = {
    'm-utils': {
        entry: resolve('dist/src/entry-compiler.js'), // 入口文件路径
        dest: resolve('lib/m-utils.js'), // 导出的文件路径
        format: 'umd', // 格式
        moduleName: 'mUtils', 
        banner,  // 打包后默认的文档注释
        plugins: defaultPlugins // 插件
    },
    'm-utils-min': {
        entry: resolve('dist/src/entry-compiler.js'),
        dest: resolve('lib/m-utils-min.js'),
        format: 'umd',
        moduleName: 'mUtils',
        banner,
        plugins: [...defaultPlugins, terser()]
    },
    'm-utils-cjs': {
        entry: resolve('dist/src/entry-compiler.js'),
        dest: resolve('lib/m-utils-cjs.js'),
        format: 'cjs',
        banner,
        plugins: defaultPlugins
    },
    'm-utils-esm': {
        entry: resolve('dist/src/entry-compiler.js'),
        dest: resolve('lib/m-utils-esm.js'),
        format: 'es',
        banner,
        plugins: defaultPlugins
    },
}


/**
 * 获取对应name的打包配置
 * @param {*} name 
 */
function getConfig(name) {
    const opts = builds[name];
    const config = {
        input: opts.entry,
        external: opts.external || [],
        plugins: opts.plugins || [],
        output: {
            file: opts.dest,
            format: opts.format,
            banner: opts.banner,
            name: opts.moduleName || 'mUtils',
            globals: opts.globals,
            exports: 'named', /** Disable warning for default imports */
        },
        onwarn: (msg, warn) => {
            warn(msg);
        }
    }
    Object.defineProperty(config, '_name', {
        enumerable: false,
        value: name
    });
    return config;
}

if(process.env.TARGET) {
    module.exports = getConfig(process.env.TARGET);
}else {
    exports.defaultPlugins = defaultPlugins;
    exports.getBuild = getConfig;
    exports.getAllBuilds = () => Object.keys(builds).map(getConfig);
}
...... 
......
```
 
为了打包文件兼容node端，以及浏览器端的引用，getConfig该方法默认返回umd格式的配置，根据**环境变量process.env.TARGET**返回指定格式的rollup配置并导出rollup的options配置

在package.json ，````--environment TARGET:m-utils`-cjs```
 指定了 ````process.env.TARGET```` 的值， 执行````npm run dev:cjs```` m-utils-cjs.js保存到lib下
```
"scripts": {
 
   ......
    "dev:umd": "rollup -w -c scripts/config.js --environment TARGET:m-utils",
    "dev:cjs": "rollup -w -c scripts/config.js --environment TARGET:m-utils-cjs.js",
    "dev:esm": "rollup -c scripts/config.js --environment TARGET:m-utils-esm",
     ......
  },
```

##### build.js构建脚本

```
  ......
let building = ora('building...');
if (!fs.existsSync('lib')) {
  fs.mkdirSync('lib')
}

// 获取rollup配置
let builds = require('./config').getAllBuilds()

// 打包所有配置的文件
function buildConfig(builds) {
  building.start();
  let built = 0;
  const total = builds.length;
  const next = () => {
    buildEntry(builds[built]).then(() => {
      built++;
      if (built < total) {
        next()
      }
    }).then(() => {
      building.stop()
    }).catch(logError)
  }
  next()
}

function buildEntry(config) {
  const output = config.output;
  const { file } = output;
  return rollup(config).then(bundle => bundle.generate(output)).then(({ output: [{ code }] }) => {
    return write(file, code);
  })
}
...... 
......
```

从config.js暴露的getAllBuilds()方法获取所有配置，传入buildConfig方法，打包所有配置文件，即m-utils-cjs.js、m-utils-esm.js等文件。

> 看过lodash.js的源码就知道，它每个方法都是一个独立的文件，所以需要什么就 import lodash + '/' + 对应的方法名就可以的，这样有利于后续按需加载的实现。参考该思路，**此项目每个方法是一个独立的文件，并打包保存到lib路径下，实现如下：**

```
...... 
......

// 导出单个函数
function buildSingleFn() {
  const targetPath1 = path.resolve(__dirname, '../', 'dist/src/')
  const dir1 = fs.readdirSync(targetPath1)
  dir1.map(type => {
    if (/entry-compiler.js/.test(type)) return;
    const targetPath2 = path.resolve(__dirname, '../', `dist/src/${type}`)
    const dir2 = fs.readdirSync(targetPath2)
    dir2.map(fn => {
      if (/.map/.test(fn)) return;
      try {
        const targetPath3 = path.resolve(__dirname, '../', `dist/src/${type}/${fn}`)
        fs.readFile(targetPath3, async (err, data) => {
            if(err) return;
            const handleContent = data.toString().replace(/require\(".{1,2}\/[\w\/]+"\)/g, (match) => {
              // match 为 require("../collection/each") => require("./each")
              const splitArr = match.split('/')
              const lastStr = splitArr[splitArr.length - 1].slice(0, -2)
              const handleStr = `require('./${lastStr}')`
              return handleStr
            })
            const libPath = path.resolve(__dirname, '../', 'lib')
            await fs.writeFileSync(`${libPath}/${fn}`, handleContent)
             //单个函数rollup打包到lib文件根目录下
            let moduleName = firstUpperCase(fn.replace(/.js/,''));
            let config = {
              input: path.resolve(__dirname, '../', `lib/${fn}`),
              plugins: defaultPlugins,
              external: ['tslib', 'dayjs'], // 由于函数用ts编写，使用external外部引用tslib，减少打包体积
              output: {
                file: `lib/${fn}`,
                format: 'umd',  
                name: `${moduleName}`,
                globals: {
                  tslib:'tslib',
                  dayjs: 'dayjs',
                },
                banner: '/*!\n' +
                ` * @author mzn\n` +
                ` * @desc ${moduleName}\n` +
                ' */',
              }
            }
            await buildEntry(config);
          })
      } catch (e) {
        logError(e);
      }
    })
  })
}
// 构建打包（全量和单个）
async function build() {
  if (!fs.existsSync(path.resolve(__dirname, '../', 'lib'))) {
    fs.mkdirSync(path.resolve(__dirname, '../', 'lib'))
  }
  building.start()
  Promise.all([
    await buildConfig(builds),
    await buildSingleFn(),
  ]).then(([result1, result2]) => {
    building.stop()
  }).catch(logError)
}
build();

...... 
......
```
> 执行 `npm run build`，调用build方法，打包全量包和单个函数的文件。

打包所有单个文件的方法待优化
## 单元测试

单元测试使用`karma + mocha + coverage + chai`，`karma` 为我们自动建立一个测试用的浏览器环境，能够测试涉及到Dom等语法的操作。

引入`karma`，执行`karma init`，在项目根路径生成`karma.config.js`配置文件，核心部分如下：

```

module.exports = function(config) {
config.set({
    // 识别ts
    mime: {
      'text/x-typescript': ['ts', 'tsx']
    },
    // 使用webpack处理，则不需要karma匹配文件，只留一个入口给karma
    webpackMiddleware: {
      noInfo: true,
      stats: 'errors-only'
    },
    webpack: {
      mode: 'development',
      entry: './src/entry-compiler.ts',
      output: {
        filename: '[name].js'
      },
      devtool: 'inline-source-map',
      module: {
        rules: [{
            test: /\.tsx?$/,
            use: {
              loader: 'ts-loader',
              options: {
                configFile: path.join(__dirname, 'tsconfig.json')
              }
            },
            exclude: [path.join(__dirname, 'node_modules')]
          },
          {
            test: /\.tsx?$/,
            include: [path.join(__dirname, 'src')],
            enforce: 'post',
            use: {
            //webpack打包前记录编译前文件
              loader: 'istanbul-instrumenter-loader',
              options: { esModules: true }
            }
          }
        ]
      },
      resolve: {
        extensions: ['.tsx', '.ts', '.js', '.json']
      }
    },
    // 生成coverage覆盖率报告
    coverageIstanbulReporter: {
      reports: ['html', 'lcovonly', 'text-summary'],
      dir: path.join(__dirname, 'coverage/%browser%/'),
      fixWebpackSourcePaths: true,
      'report-config': {
        html: { outdir: 'html' }
      }
    },
   // 配置使用的测试框架列表，默认为[]
    frameworks: ['mocha', 'chai'],
    // list of files / patterns to load in the browser
    files: [
      'test/index.ts'
    ],
    //预处理
    preprocessors: {
      'test/index.ts': ['webpack', 'coverage']
    },
    //使用的报告者（reporter）列表
    reporters: ['mocha', 'nyan', 'coverage-istanbul'],
    // reporter options
    mochaReporter: {
      colors: {
        success: 'blue',
        info: 'bgGreen',
        warning: 'cyan',
        error: 'bgRed'
      },
      symbols: {
        success: '+',
        info: '#',
        warning: '!',
        error: 'x'
      }
    },
    // 配置覆盖率报告的查看方式,type查看类型，可取值html、text等等，dir输出目录
    coverageReporter: {
      type: 'lcovonly',
      dir: 'coverage/'
    },
    ...
  })
}

```
配置中webpack关键在与打包前使用`istanbul-instrumenter-loader`，记录编译前文件，因为webpack会帮我们加入很多它的代码，得出的代码覆盖率失去了意义。

> 查看测试覆盖率，打开coverage文件夹下的html浏览，

* 行覆盖率（line coverage）
* 函数覆盖率（function coverage）
* 分支覆盖率（branch coverage）
* 语句覆盖率（statement coverage）

![](https://user-gold-cdn.xitu.io/2019/10/29/16e16996adc57085?w=1918&h=316&f=png&s=19825)

## 发布

### 添加函数

当前项目源码使用typescript编写，*若还不熟悉的同学，请先查看[ts官方文档](https://www.tslang.cn/docs/home.html)*

在````src```` 目录下， 新建分类目录或者选择一个分类，在子文件夹下添加子文件，每个文件为单独的一个函数功能模块。(如下：src/array/arrayFlat.ts）

```

/**
 * @author mznorz
 * @desc 数组平铺  
 * @param {Array} arr
 * @return {Array}
 */
function arrayFlat(arr: any[]) {
  let temp: any[] = [];
  for (let i = 0; i < arr.length; i++) {
    const item = arr[i];
    if (Object.prototype.toString.call(item).slice(8, -1) === "Array") {
      temp = temp.concat(arrayFlat(item));
    } else {
      temp.push(item);
    }
  }
  return temp;
}
export = arrayFlat;


```
然后在 src/entry-compiler.ts中暴露arrayFlat

> 为了在使用该库时，能够获得对应的代码补全、接口提示等功能，在项目根路径下添加```index.d.ts```声明文件，并在````package.json````中的````type````字段指定声明文件的路径。

```
...... 
declare namespace mUtils {
    
    /**
   * @desc 数组平铺
   * @param {Array} arr 
   * @return {Array}
   */
  export function arrayFlat(arr: any[]): any[];
   ...... 
}

export = mUtils;
```


### 添加测试用例

在test文件下新建测试用例
```
import { expect } from "chai";
import _ from "../src/entry-compiler";

describe("测试 数组操作 方法", () => {
  it("测试数组平铺", () => {
    const arr1 = [1,[2,3,[4,5]],[4],0];
    const arr2 = [1,2,3,4,5,4,0];
    expect(_.arrayFlat(arr1)).to.deep.equal(arr2);
  }); 
});
...... 
......
```


### 测试并打包
执行`npm run test`，查看所有测试用例是否通过，查看/coverage文件下代码**测试覆盖率报告**，如若没什么问题，执行`npm run compile`编译ts代码，再执行`npm run build`打包


### 发布到npm私服
[1] 公司内部使用，一般都是发布到内部的npm私服，对于npm私服的搭建，在此不做过多的讲解
[2] 在此发布npm作用域包，修改`package.json`中的`name` 为`@mutils/m-utils`
[3] 项目的入口文件，修改 `mian`和`module` 分别为`
lib/m-utils-min.js` 和 `lib/m-utils-esm.js`

* main : 定义了 npm 包的入口文件，browser 环境和 node 环境均可使用
* module : 定义 npm 包的 ESM 规范的入口文件，browser 环境和 node 环境均可使用

[4] 设置发布的私服地址，修改`publishConfig`字段
```
"publishConfig": {
    "registry": "https://npm-registry.xxx.cn/"
  },
```
[5] 执行`npm publish`，登录账号密码发布


### 使用
 
1. 直接下载`lib` 目录下的 m.min.js，通过 `<script>` 标签引入
```
 <script src="m-utils-min.js"></script> 
 <script> 
  var arrayFlat = mUtils.arrayFlat() 
 </script>
```

2. 使用npm安装

```
npm i @mutils/m-utils -S
```
直接安装会报找不到该包的错误信息，需在项目根路径创建 `.npmrc` 文件，并为作用域包设置registry

```
registry=https://registry.npmjs.org

# Set a new registry for a scoped package
# https://npm-registry.xxx.cn  私服地址

@mutils:registry=https://npm-registry.xxx.cn  
```

```
import mUtils from '@mutils/m-utils';
import { arrayFlat } from '@mutils/m-utils';
```


## 相关链接

* [源码地址](https://github.com/ghfengye/mutils.git)

今天的分享就到这里，后续会继续完善，希望对你有帮助~~

***~~未完待续***
