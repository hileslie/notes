# Require机制及实现

## 什么是CommonJS
每一个文件就是一个模块，拥有自己独立的作用域、变量、以及方法等，对其他模块都不可见。每个模块内部，module变量代表当前模块。该变量是一个对象，它的exports属性（module.exports）是对外的接口。

## 模块分类
- 原生模块：Node提供的模块
  - 内建模块：C/C++编写
  - 全局模块：process
  - 其他：http、net
- 文件模块：用户编写的模块
  - 普通文件模块
  - C++拓展模块

## 模块加载
- 原生模块加载
- 文件模块加载

```javascript
function Module(id = '') {
    this.id = id; // require路径
    this.path = path.dirname(id); // path 内置模块，获取对应文件夹路径
    this.exports = {}; // 导出内容
    this.filename = null; // 模块对应的文件名
    this.loaded = false; // 用来标识当前模块是否已经加载
}
Module._cache = Object.create(null); // 创建空的缓存对象
Module._extensions = Object.create(null); // 创建空的拓展点名类型函数对象

Module.prototype.require = function(id) {
    return Module._load(id, this, /*isMain*/ false)
}
Module._load = function(request) {
    // request传入路径参数

    // 路径分析并定位到文件
    const filename = Module._resolveFilename(request);

    // 判断模块是否加载过，也可解决模块重复引用问题
    const cacheModule = Module._cache[filename];
    if(cacheModule !== undefined) {
        return cacheModule.exports;
    }

    // 去加载node原生模块逻辑
    // const mod = loadNativeModule(filename, request);
    // if (mod && mod.canBeRequiredByUsers) return mod.exports;

    // 如果缓存不存在，自行加载模块
    const module = new Module(filename);

    // load之前加入缓存
    Module._cache[filename] = module;

    // module.load 加载文件代码
    module.load(filename)

    // 返回模块的module.exports
    return module.exports
}

Module._resolveFilename = function(request) {
    const filename = path.resolve(request); // 获取传入参数对应的绝对路径
    const extname = path.extname(request);

    // 如果没有文件后缀名，判断是否可以添加.js和.json
    if(!extname) {
        const exts = Object.keys(Module._extensions);
        for(let i = 0; i < exts.length; i++) {
            const currentPath = `${filename}${exts[i]}`;
            // 如果拼接后的文件存在，返回拼接的路径 
            if (false.existsSync(currentPath)) {
                return currentPath;
            }
        }
    }
    return filename
}

Module.prototype.load = function(filename) {
    // 获取文件后缀名
    const extname = path.extname(filename)

    // 根据不同的后缀名进行不同的处理
    Module._extensions[extname](this, filename)

    this.loaded = true
}

Module._extensions['.js'] = function(module, filename) {
    const content = fs.readFileSync(filename, 'utf-8');
    module._compile(content, filename);
}

Module.wrapper = [
    '(function(exports, require, module, __filename, __dirname) {',
    '\n});'
]

Module.wrap = function(script) {
    return Module.wrapper[0] + script + Module.wrapper[1];
}

Module.prototype._compile = function (content, filename) {
    // 获取包装后的函数体
    const wrapper = Module.wrap(content);

    // vm是Node.js的虚拟机模块
    const compilesWrapper = vm.runInThisContext(wrapper, {
        filename,
        lineOffset: 0,
        displayErrors: true,
    })
    const dirname = path.dirname(filename)

    compilesWrapper.call(this.exports, this.require, this.exports, this, filename, dirname);
}
```



## 总结

### require加载是同步加载

```javascript
fs.existsSync(currentPath)
fs.readFileSync(filename, 'utf8');
```

### exports和module.exports的区别

在一个模块没有做任何代码编写之前，exports和module.exports都是{}

### require不会造成循环引用问题

在加载模块时，将模块缓存起来，加载到时直接返回。

直接返回会造成内容不完整，解决将module.exports放在文件最顶部

### CommonJS和ES6Module的区别

| 区别     | require/exports                                              | import/export                                                |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 加载方式 | 同步加载，运行时动态加载，加载的是一个对象，对象需要在脚本执行完成后才会生成 | 异步加载，静态编译，ES6模块不是对象，它对外接口只是一种静态定义，在代码静态解析阶段就会生成 |
| 输出对比 | 输出的是一个值得拷贝，一旦输出一个值，模块内部的变化不会影响到这个值 | 输出的是值的引用，JS引擎对脚本静态分析的时候，遇到模块加载命令import，就会生成一个只读引用。等到脚本真正执行是，再根据这个只读引用，到被加载的模块里取值。若文件引用的模块值改变，require引入的模块值不会改变，而import引用的模块值会改变 |
|          |                                                              |                                                              |

