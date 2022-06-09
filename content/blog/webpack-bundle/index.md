---
title: Webpack 打包产物分析
date: '2022-06-06'
description: CommonJs, ESModule, import/export, require, module.exports
order: 45
---

首先，创建两个模块供主文件导入，分别为 `es.js`（使用 export 方式导出模块） 和 `cjs.js`（使用 module.exports 方式导出模块）。

然后，在主文件中分四种情况来导入模块，来测试一下 Webpack 打包后的产物，他们分别为：

- 使用 import 导入 es 模块
- 使用 import 导入 cjs 模块
- 使用 require 导入 es 模块
- 使用 require 导入 cjs 模块

```js
(() => {
    // 每个 module 都会转化成一个 function，然后用一个map去保存这些module，其中，以这个模块文件所在的相对路径作为该模块的key，同时也是该模块的id
    const __webpack_modules__ = ({
        './src/es.js': (__unused_webpack_module, __webpack_exports__, __webpack_require__) => {
            /**
                源码：
                    export let esVal = 3;

                    setTimeout(() => {
                        esVal = 333;
                    })

                    export default esVal;
             */
            __webpack_require__.r(__webpack_exports__);

            __webpack_require__.d(__webpack_exports__, {
                // 通过闭包的方式，让其他模块可以访问到该模块内部的变量
               'esVal': () => esVal,
               'default': () => __WEBPACK_DEFAULT_EXPORT__
            });

            let esVal = 3;
            setTimeout(() => {
                esVal = 333;
            })
            const __WEBPACK_DEFAULT_EXPORT__ = esVal;
        },
        './src/cjs.js': (module) => {
            /**
                源码：
                let val = 3;

                setTimeout(() => {
                    val = 333;
                }, 1000)

                module.exports = {
                    val
                }
             */

            // 源码是 commonjs 模块，直接拷贝源码即可
            // 函数参数传入所需要的 module 对象
            var val = 3;
            setTimeout(function () {
                val = 333;
            }, 1000);
            module.exports = {
                val: val
            };
        }
    });

    // 缓存已经执导入的 module 结果
    const __webpack_module_cache__ = {};


    // 导入函数，接收一个 moduleId 表示需要被导入的模块
    function __webpack_require__(moduleId) {
        const cachedModule = __webpack_module_cache__[moduleId];
        
        // 该模块是否已经被导入过
        if (cachedModule !== undefined) {
            // 最终实现都是借鉴了 commonjs 的导出方式，即导出一个 exports 对象
            return cachedModule.exports;
        }

        // 如果没有被导入过，初始化一个新的 module 结果对象，并将他保存在 cache 中
        // (这里最能体现，实现是借鉴了 commonjs 的导出方式，导出的值就是 module.exports)
        const module = { exports: {} };
        __webpack_module_cache__[moduleId] = module;

        
        // 执行该模块对应的函数，在函数执行过程中，导出的值会被保存到 module.exports 对象中
        __webpack_modules__[moduleId](module, module.exports, __webpack_require__);

        // 返回结果
        return module.exports;
    }

    // define __esModule on exports
    __webpack_require__.r = (exports) => {
        Object.defineProperty(exports, '__esModule', { value: true });
    };

    // 为 exports 导出的成员定义 getter 函数
    __webpack_require__.d = (exports, definition) => {
        for (let key in definition) {
            if (
                Object.prototype.hasOwnProperty.call(definition, key) &&
                !Object.prototype.hasOwnProperty.call(exports, key)
            ) {
                Object.defineProperty(exports, key, {
                    enumerable: true,
                    // 1. 只定义get，说明无法再模块外部对其进行修改
                    // 2. definition[key] 是一个函数，也就是通过闭包的方式，巧妙地访问到了模块内部的变量
                    get: definition[key]
                });
            }
        }
    }

    // 获取 默认导出（default）兼容不和谐的模块（比如 import 了一个 commonjs 模块）
    __webpack_require__.n = (exports) => {
        const getter = exports && exports.__esModule ? 
            () => exports['default'] :
            () => exports;

        __webpack_require__.d(getter, { a: getter });

        return getter;
    };

    const __webpack_exports__ = {};


    // 执行入口模块
    (() => {
        /**
         * 源码
         *  // import 一个 es 模块
         * 
            import esValDefault from './es';
            import { esVal } from './es';

            setTimeout(() => {
                console.log('es default val: ', esValDefault);
                console.log('es val: ', esVal);
            }, 3000)
         */

        __webpack_require__.r(__webpack_exports__);
        
        var _es__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__('./src/es.js');

        setTimeout(() => {
            console.log('es default val: ', _es__WEBPACK_IMPORTED_MODULE_0__.exports['default']);
            console.log('es val: ', _es__WEBPACK_IMPORTED_MODULE_0__.exports.esVal);
        }, 3000);


        /**
         * 
         * 源码：
         * 
         * // import 一个 commonjs 模块

            import cjs from './cjs';
            import { val } from './cjs';

            console.log(cjs);

            setTimeout(() => {
                console.log(cjs.val);
                console.log(val);
            }, 3000)
         */
        __webpack_require__.r(__webpack_exports__);
        var _cjs__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__("./src/cjs.js");
        var _cjs__WEBPACK_IMPORTED_MODULE_0___default = __webpack_require__.n(_cjs__WEBPACK_IMPORTED_MODULE_0__);

        console.log((_cjs__WEBPACK_IMPORTED_MODULE_0___default()));
        setTimeout(function () {
            console.log((_cjs__WEBPACK_IMPORTED_MODULE_0___default().val));
            console.log(_cjs__WEBPACK_IMPORTED_MODULE_0__.val);
        }, 3000);


        /**
         * 源码
         * 
         * // require 一个 cjs 模块
         * 
         * 
            const cjs = require('./cjs');
            const { val } = require('./cjs');

            console.log(cjs);

            setTimeout(() => {
                console.log(cjs.val);
                console.log(val);
            }, 3000)

         */
        var cjs = __webpack_require__("./src/cjs.js");

        var _require = __webpack_require__("./src/cjs.js"),
            val = _require.val;
        
        console.log(cjs);
        setTimeout(function () {
            console.log(cjs.val);
            console.log(val);
        }, 3000);


        /**
         * 源码
         * 
         * // require 一个 es 模块
         * 
         * 
            const es = require('./es');
            const { esVal } = require('./es');

            setTimeout(() => {
                console.log(es.esVal);
                console.log(esVal);
            }, 3000)
         */
        var es = __webpack_require__("./src/es.js");

        var _require = __webpack_require__("./src/es.js"),
        // 如果使用 require + 对象解构的方式，相当于 esVal 已经被赋值给 esVal 这个变量了，后续再取 esVal 就是一个固定值了
        // 而 es.esVal（或者说_require.esVal 同一个东西） 他们其实是一个 getter 函数，执行了一个闭包，可以访问到模块内部的 esVal 变量，所以调用的时候能实时拿到最新值
            esVal = _require.esVal;
        
        setTimeout(function () {
            console.log(es.esVal);
            console.log(esVal);
        }, 3000);
    })()
})()
```
