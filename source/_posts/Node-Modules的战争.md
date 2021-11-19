---

title: Node模块的战争:为什么CommonJS和ES Modules不能统一

---

作者：[Dan Fabulich](https://danfabulich.medium.com/?source=post_page-----9617135eeca1-----------------------------------) [Follow](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fsubscribe%2Fuser%2Fa0b6e5159037%2F9617135eeca1&operation=register&redirect=https%3A%2F%2Fredfin.engineering%2Fnode-modules-at-war-why-commonjs-and-es-modules-cant-get-along-9617135eeca1&user=Dan Fabulich&userId=a0b6e5159037&source=post_page-a0b6e5159037----9617135eeca1---------------------follow_byline--------------)

文章来源：https://redfin.engineering/node-modules-at-war-why-commonjs-and-es-modules-cant-get-along-9617135eeca1



二者虽然可以配合着使用，但却非常麻烦。

在Node14版本中，存在着两个模块模式：分别是老一代的CommonJS(CJS)模式和新一代ESM(也被称为EJS)模式。

CJS使用 `require()` 和 `module.exports`;ESM使用 `import`和`export`。



**ESM 和CJS 是两个完全不同的种类。**表面上看，ESM与CJS非常相似，但是他们的执行过程完全不同。

他们俩一个是小蜜蜂，一个是大黄蜂。(作者意思是，他们犹如小蜜蜂和大黄蜂，放在一起，看起来根本分不清谁是谁。)

![](/Users/hgx/Downloads/1_ljab8kLJtGC-o36VQ7vBZg.jpeg)




**在ESM中使用CJS是很麻烦的事，反之依然。**

下面是他们配合使用的一些规则，后面我会详细的去解释他们：

1. 我们不能 `require()`ESM代码块；我们只能 `import`ESM代码块，就像这样：`import {foo} from "foo"`
2. CJS不能像ESM那样用静态 `import`声明导入代码
3. CJS可以利用异步动态 `import()` 来使用ESM,但跟使用同步 `require()` 比起来麻烦的多
4. ESM 可以 `import` CJS，但是只能使用”默认导出“ （default import）语法 `import _ from  'lodash'` ,而不能用命名方式（named exports）导出“named import” 语法 `import { shuffle } from 'lodash'`。CJS用命名导出语法会比较麻烦。
5. ESM 可以`require` CJS ,即使用named exports,但是这样有点不值得，因为这样又需要引用更多的文件，更糟糕的是，像webpack和rollup这样的打包工具碰到ESM中用`require()`的情况将会不知道如何工作。
6. Node默认使用的是CJS模式；你得手选ESM 模式。你可以通过把script文件后缀.j s改成.mjs。轮到切换CJS的时候，你可以在package.json文件中设置 "type":"module"，然后你就可以通过把script文件.js后缀改成.cjs来切回到默认模式（你甚至可以在子目录下放个只有一行{ "type":"module" }的package.json的文件就行）。



这些规则遵循起来很痛苦。更糟糕是，对于大多数的用户，特别是Node新手，这些规则霉涩难懂。（也不用怕，我会在这篇文章中一一解释）



很多node生态的观察员（评论者）推测，这些规则要归因于领导者的错误决策，或出于对ESM的敌视。但是，我们会发现，所有的这些规则都有他门存在的原因，而且这些原因也将使这些规则在将来很难被打破。



下面是我给包的作者总结出的三条准则：

- 为你的library 提供一个CJS版本
- CJS版本命名导出部分（named exports）提供一个用ESM进行一层封装的版本
- 在`package.json`文件添加一个 `exports` 的map映射



**It’s all gonna be OK.**

一切都会好起来滴!



### **背景：什么是CJS？什么又是ESM？**



至Node推出已来，Node的模块都是按照CommonJS来写的。我们使用 `require()`来导出他们。当我们准备一个给别人使用的模块的时候，我们可以定义`exports`，或者是通过设置 `module.exports.foo=''bar “` 来进行命名导出“named exports”，再或者通过设置 `module.exports='baz'` 来进行默认导出。



下面是一个使用**命名导出named exports**的CJS🌰，在util.cjs里有一个名叫num的导出。

```javascript
// @filename: util.cjs
module.exports.sum = (x, y) => x + y;


// @filename: main.cjs
const {sum} = require('./util.cjs');
console.log(sum(2, 4));
```



再下面是一个在util.cjs里设置了个**默认导出** default export 的🌰。默认导出的内容没有名字；使用`require()`的模块自己定义他们的名字。

```javascript
// @filename: util.cjs
module.exports = (x, y) => x + y;

// @filename: main.cjs
const whateverWeWant = require('./util.cjs');
console.log(whateverWeWant(2, 4));
```



在 ESM 模式下，import 和 export  是语言的一部分；就像CJS有两种不同的语法，分别是 named exports 和 default exports。



下面是一个使用了**named exports** 的 ESM的🌰,在util.mjs 里有个命名为sum的导出。

```javascript
// @filename: util.mjs
export const sum = (x, y) => x + y;

// @filename: main.mjs
import {sum} from './util.mjs'
console.log(sum(2, 4));
```





下面是一个使用默认导出形式 的ESM 🌰，也和CJS一样，默认导出没有名字，但是在import 的地方可以自己定义他的名字。

```javascript
// @filename: util.mjs
export default (x, y) => x + y;

// @filename: main.mjs
import whateverWeWant from './util.mjs'
console.log(whateverWeWant(2, 4));
```





### ESM 和 CJS 是完全不同种类

**对于CommonJS，require() 操作时同步的；**他不会返回一个promise 或者是callback。require()是直接从硬盘读取数据（或者读取来至网络的数据），然后立即执行脚本，读取的那些内容或许他们自己也会有些I/0操作，再或者别的副作用，然后不管返回的是啥，都会被赋给 `module.exports`.



**对于 ESM，模块的loader是异步地进行工作的。**首先，他会对 import 和 exports进行语法分析，而不是去执行导入的模块。语法分析阶段，ESM loader 可以立即检查命名导出模式名字类型错误并会抛出异常，而不会真正去执行依赖的代码。



ESM 模块 loader 接下来会异步下载和分析所有你的导入，然后你所有的导入会构建一个模块相互依赖图，直到最后loader发现一段脚本没有导入任何东西。最后，脚本才被允许执行，脚本中导入的内容才会被执行，诸如此类。



在ESM模块依赖图中，所有的相邻脚本是并行下载的，但是执行的时候是按照顺序执行的,这是由loader规范控制的。

















