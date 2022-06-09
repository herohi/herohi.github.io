---
title: Decorator 装饰器的用法、实现与应用
date: '2022-05-27'
description: decorator, typescript, Reflect, reflect-metadata
order: 49
---

Decorator 是 ES7 的一个新语法，它可以用来装饰一个类、类中的某个方法、类中的某个属性或类中的某个方法的某个入参。装饰器本质上是一个函数，被装饰的对象（target）会以参数的形式传递给装饰器函数，在函数中可以对这个对象（target）进行操作，达到修改或者记录等目的。

## 一、开始调试

由于装饰器是 ES7 的语法，js 中需要借助 babel 插件进行语法转换，而 ts 只需要对 tsconfig.json 文件进行配置就可以使用了，故这边选用 ts 进行调试。

**tsconfig.json**

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "es2017",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": false,
    "noImplicitAny": false,
    "strictBindCallApply": false,
    "forceConsistentCasingInFileNames": false,
    "noFallthroughCasesInSwitch": false
  },
  "exclude": ["node_modules"]
}
```

## 二、装饰器基本用法

### 1. 装饰一个类

装饰一个类的时候入参有且只能有一个 target，表示这个类，多传或少传参数 ts 都会提示错误。

```ts
@dec
class A{}

function dec(target) {
    console.log(target) // A

    return Object // 支持返回一个新的类取代原来的A
}

console.log(A) // Object
```

### 2. 装饰类中的一个方法

装饰一个类中的方法时，target 是这个类的原型对象，name 是被装饰的方法名

```ts
class B {
    @dec
    b() {}
}

function dec(target, name) {
    console.log(target.constructor); // [class B]
    console.log(target === B.prototype, name); // true b
}
```

### 3. 装饰类中的一个属性

装饰一个类中的属性时，target 是这个类的原型对象，name 是被装饰的属性名

```ts
class C {
    @dec
    c = 3;
}

function dec(target, name) {
    console.log(target === C.prototype, name); // true c
}
```

### 4. 装饰类中的某个方法的某个参数

其中，desc 为参数的坐标，比如这边是第二个参数，就是 1

```ts
class D {
    d(params1, @dec params2) {}
}

function dec(target, name, desc) {
    console.log(target === D.prototype, name, desc); // true d 1
}
```

## 三、装饰器实现

### 1. 先看看 __decorate 方法

使用 `tsc` 命令对源码进行编译，可以看到输出的代码中定义了一个 decorate 方法。

下面是我经过优化后的代码。

```ts
function __decorate(decorators, target, key, desc) {
    const len = arguments.length;
    let res =
        len < 3
            ? target
            : desc === null
            ? (desc = Object.getOwnPropertyDescriptor(target, key))
            : desc;

    for (let i = decorators.length - 1; i >= 0; i--) {
        const decorator = decorators[i];

        res =
            (len < 3
                ? decorator(res)
                : len > 3
                ? decorator(target, key, res)
                : decorator(target, key)) || res;
    }

    if (len > 3 && res) {
        Object.defineProperty(target, key, res);
    }

    return res;
}
```

其中

**len：**表示传入的参数个数。若 len = 2 可以判断为该装饰器为类装饰器，若 len = 4 可以判断为该装饰器为其他类型装饰器。

**res：**新值，用来替换老值，可以被装饰器的返回值改变。不同类型的装饰器最终要替换的东西也不一样：

- 类装饰器，res 用来替换原来那个类
- 方法装饰器，res 用来替换该方法的描述对象（该方法挂在类的prototype上）
- 属性装饰器，res初始为undefined（因为它是一个类的成员变量，只有当类实例化后才会出现），但如果装饰器返回了一个新的对象，则最终也会给原型上的名为 `[key]` 的属性定义一个描述对象
- 如果是参数装饰器，res 用来替换该方法的描述对象（同方法装饰器）

### 2. 几种装饰器的代码补全

看明白了decorate方法，再补全几种装饰器的完整代码就比较容易理解了。

**1. 类装饰器**

```ts

let A = class A {};

A = __decorate([
    decA
], A);

function decA(target) {
    console.log(target);
    return Object;
}
console.log(A);
```

**2. 方法装饰器**

先忽略 `__metadata`，后面看 reflect-metadata 的时候会看到。

```ts
class B {
    b() {}
}

__decorate([
    decB,
    __metadata("design:type", Function),
    __metadata("design:paramtypes", []),
    __metadata("design:returntype", void 0)
], B.prototype, "b", null);

function decB(target, name) {
    console.log(target.constructor);
    console.log(target === B.prototype, name);
}
```

**3. 属性装饰器**

```ts
class C {
    constructor() {
        this.c = 3;
    }
}
__decorate([
    decC,
    __metadata("design:type", Object)
], C.prototype, "c", void 0);
function decC(target, name, c) {
    console.log(c);
    console.log(target === C.prototype, name);
}
```

**4. 参数装饰器**

通过 `__param` 方法可以看出，参数装饰器比方法装饰器多接收了一个 `paramIndex` 参数。

```ts
var __param = (this && this.__param) || function (paramIndex, decorator) {
    return function (target, key) { decorator(target, key, paramIndex); }
};
class D {
    d(params1, params2) { }
}
__decorate([
    __param(1, dec),
    __metadata("design:type", Function),
    __metadata("design:paramtypes", [Object, Object]),
    __metadata("design:returntype", void 0)
], D.prototype, "d", null);
function dec(target, name, desc) {
    console.log(target === D.prototype, name, desc);
}
```

## 四、Reflect 和 reflect-metadata

Reflect 是一个内置对象，上面挂载了13个静态方法，他的设计的目的有以下几点：

1. 将 Object 对象上一些明显属于语言内部的方法，比如 Object.defineProperty 放到 Reflect 上
2. 修改某些 Object 方法的返回结果，使他们变得更合理
3. 让 Object 操作都变成函数行为。比如，它会将 in, delete 操作符，将他们替换成 Reflect.has 和 Reflect.deleteProperty
4. Reflect 方法与 Proxy 对象的方法一一对应，就算 Proxy 的默认行为被修改了，也可以从 Reflect 上获取到默认行为

### reflect-metadata

reflect-metadata 不仅实现了 Reflect 的 polyfill，还额外提供了 Reflect.metadata, Reflect.defineMetadata 和 Reflect.getMetadata。

##### 1. 使用

默认的元数据装饰器可以被用于类, 类成员以及参数。

```ts
import 'reflect-metadata';

@Reflect.metadata('className', 'A')
class A {
    @Reflect.metadata('propName', 'name')
    name = 'a'

    @Reflect.metadata('funcName', 'getName')
    getName() {}
}

console.log(Reflect.getMetadata('className', A)); // A
console.log(Reflect.getMetadata('propName', new A, 'name')); // name
console.log(Reflect.getMetadata('propName', A.prototype, 'name')); // name
console.log(Reflect.getMetadata('funcName', A.prototype, 'getName')); // getName
```

`defineMetadata` 该方法是 `metadata` 的定义版本, 也就是非@版本, 会多传一个参数 target, 表示待装饰的对象。

```ts
Reflect.defineMetadata('className', 'NewA', A);
console.log(Reflect.getMetadata('className', A)); // NewA
```

##### 2. 原理

首先要明确我们的修饰器修饰的是哪个 target，如果是类修饰器，target 就是被修饰的类，如果是其他修饰器，那么 target 就是修饰器所在类的原型对象。

其次，targetKey 在类修饰器中没有，在属性修饰器中为被修饰的属性名，方法/参数修饰器中，为被修饰的方法名。

如果系统内部已经实现了，则会在 target 上增加一个隐藏属性 `[[metadata]]` ，而 reflect-metadata 中是通过 WeakMap 来实现的一个 polyfill。


## 四、装饰器应用


