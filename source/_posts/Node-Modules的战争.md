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



#### **默认使用CJS是因为ESM改变了很多基础的东西**



ESM改变了好多javascript基础的东西。ESM scripts 默认使用严格模式（`use strict`）,他们的this并不会指向全局对象，作用域的机制也变得不同，等等。



这就是为甚么，即使是浏览器，`<script>`标签默认也是 non-ESM（默认是不使用ESM）你必须给标签添加 `type="module"`属性来指定ESM模式。



在做向下兼容时，从默认的CJS切换到ESM需要非常大的勇气。（Deno,最近较火的Node替代方案，拿ESM作为默认模块管理方式，但这就造成一种现象，他的生态就得从0开始）



#### **CJS 不可以 require() ESM 是由于await是顶级的**



CJS不能require() ESM最直接的原因就是 ESM可以用顶级的await 而CJS不可以。



高阶 await 让我们可以在一个[`async function`](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Async_await)的前面使用await.

ESM的多相multi-parse loader 使ESM能够去执行高阶的await，使他避免成为最差实现（without making it a "footgun"）引用至[V8团队的blog 文章](https://v8.dev/features/top-level-await).





> 也许你曾经看到过 [Rich Haris](https://gist.github.com/Rich-Harris/0b6f317657f5167663b493c722647221)一系例关于高阶await的不看好并力劝javascript语言不要去用这个新特性。一些具体的担心如下：
>
> - 高阶 `await `会阻塞代码执行。
> - 高阶 `await `会阻塞资源的加载。
> - 没有明确描述CommonJS模块如何操作
>
> 
>
> 现阶段的3个针对这些问题的提议如下：
>
> - 相邻代码可以执行，不要有最终阻塞
> - 高阶`await` 发生在模块依赖关系执行阶段。基于此所有的资源已经被连接和加载，不要有阻塞资源加载的风险
> - 高阶`await`  仅限于【ESM】模块使用。要明确不支持CommonJS 模块使用。



（Rich 现在已经通过了现有的高阶`await` 提案）



因为CJS不支持 高阶 `await` ，所以现在几乎没有办法把ESM 中top-level `await`转成CJS的写法，就像下面这段代码，我们该如何用CJS来写？



```javascript
export const foo = await fetch('./data.json');
```



这很令人头疼，因为绝大部份的ESM脚本没有使用高阶`await`,但是，有个评论员在这方面曾这么写道过：“我不认为设计一个被假设一些功能特性不会有人用系统是一个可行的办法。



[这方面有个关于如何 require() ESM的激烈的讨论](https://github.com/nodejs/modules/issues/454)。（在发表自己的观点前请先看完讨论区的内容，如果你参与进来，你会发现高阶await甚至是不是仅有的一个问题... 当你同步的requireESM模块的时候，而ESM模块里异步的import 一些CJS模块，CJS又同步的require() 一些ESM模块的时候，你觉得结果会怎样？你得到将是一个同步/异步的死亡斑马线，就是这结果！高阶的`await`也就是这棺材板上最后一颗钉而已，也是最容易解释的。）

回头审视下这个讨论，看起来以后我门不大可能再 reuqire() ESM 这么用了。



#### **CJS 可以import() eSM，但是并不好**

到目前为止，如果你在写CJS 并且你想 去 `import` 一个 ESM 模块，那你就得用异步动态 `import()`

```
(async () => {
    const {foo} = await import('./foo.mjs');
})();
```



目前来看...还可以，我觉得，只要你没有去exports.如果你确实需要做一些导出 (exports)，那你就得导出一个Promise作为替代，

对用户来说，将会是一个极大的不方便。



```
modules.exports.foo =(async()=>{
	const {foo} = await import('./foo.mjs');
	return foo
})();
```



ESM 不能 import CJS 中具名的导出（named exports） 除非 CJS 脚本有问题



你可以这么操作：



```
import _ from './lodash.cjs'
```





但是你不能这么做：

```
import { shuffle } from './lodash.cjs'
```



这是因为CJS是在执行时候计算他们的具名导出，鉴于ESM的具名导出必须在解析阶段计算好。





幸运的是，有个变通的方法。这个变通方法很烦，但是有用，我们只需要像下面这样 import CJS 模块

```
import _from './lodash.cjs';

const { shuffle }  = _;
```



这么写也没有啥不好，有ESM意识（想兼顾ESM写法的）的CJS库甚至会为我们提供他们的包装压缩ESM版。



这已很不错，我只是希望能做的更好。



#### **无次序的执行有可能正常工作，但这将会非常的糟糕。**



很多人曾建议在ESM imports 前面执行 CJS imports，令人无法接受。这样的话， CJS 命名导出将和ESM的命名导出同时被计算。



这将会产生一个[新的问题](https://github.com/nodejs/modules/issues/81#issuecomment-391348241)

```
import {liquor} from "liquor";
import {beer} from 'beer';
```



如果`liquor` 和 `beer` 初始都是 CJS，从CJS把liquor改为ESM 将会把 `liquor,beer` 顺序改为 `beer,liquor`,这样如果`beer`依赖一些`liquor`先执行的东西就将会变得很恶心了。



是否因该无序执行仍然在争论不休，虽然这个话题几周前几乎已经关闭。



#### **动态模块可以拯救我们，但是他的🌟导出有毒**



有个替代方案可以不 require 无序 执行或 封装的 scripts。叫作[动态模块](https://github.com/nodejs/dynamic-modules)



在ESM规范中，导出功能静态定义所有具名导出。在动态模块下，导出功能在导入的地方将定义导出的名字。ESM的loader将初始地仅仅认为这些动态模块（CJS scripts）将提供所有引用的具名导出，如果没有满足这个约定随后会抛出异常。



很不幸，动态模块需要TC39语言协会通过一些javascript语言变动，但他们没有同意。



具体来说，ESM scripts 可以 export * from './foo.cjs', 这就意味着要重新导出 foo 模块所有导出的名字（这个叫着 * **star export** 🤩）



不幸又来了，loader没有办法知道星导出到底从动态模块导出啥。





动态模块星导出在规范遵循方面也产生一些争议。例如，`export * from 'omg'; export * from 'bbq'`,当 `bbq` 和`omg` 都具名导出同样名字的 `wtf` 估计会抛异常。允许用户自定义导出名字意味着这个校验的环节需要以某种方式让用户可手选/忽略。





动态模块倡导者提议禁止在动态模块使用星导出，但是 [TC39拒绝了这一提议](https://github.com/tc39/ecma262/pull/1306#issuecomment-467761625)，有个TC39成员认为说动态模块被星导出毒害的提议是“语法破坏”。



![1_YFBh0ZgMSA9Qzx1mBqlfgA](/Users/hgx/Downloads/1_YFBh0ZgMSA9Qzx1mBqlfgA.png)

​																							毒星星表示非常愤怒💢哈哈

（在我看来，我们已经在生活在一个语法破坏的世界。在node14,具名导出有毒，在动态模块，星导出也有毒。由于具名导出非常普遍，星导出相对少见，在node生态动态模块的毒害会减少）



这也许不是动态模块的归宿。一个提议是所有的node模块都转到动态模块，甚至是存粹的esm 模块，摒弃node 中ESM的多相loader。令人惊奇的是，对用户没有明显可见的影响，除了会轻微影响页面启动性能；ESM 多相loader的设计是建立在一个低网速加载脚本的场景。



但我不觉得幸运。Github上关于动态模块的issue最近关了，因为[在过去的一年已经没有关于动态模块的讨论](https://github.com/nodejs/modules/issues/252)。



另外一个想法正在酝酿之中，[在做CJS句法分析的时候尽可能查明他们的导出](https://github.com/nodejs/node/pull/33416)，但是这个方法永远不会100%覆盖所有的场景。（最新的PR仅覆盖到npm top1000包中的62%）由于这个探索阶段的尝试很不可靠，有些Node modules 工作组的成员还反对这个尝试。





ESM 可以 `require()`,但是值不当这么做

`require()` 不在 ESM默认范围内,但你可以很[容易把他引进来](https://nodejs.org/api/modules.html#modules_module_createrequire_filename)



```
import {createRequire} from 'module';
const require =createReruire(import.meta.url);

const {foo} = require('./foo.cjs');
```

这个方式的问题在于这样做起不到什么作用；还比默认 `import` 和结构赋值多了一行代码。



```
import cjsModule from './foo.cjs';
const {foo} = cjsModule；
```



另外，像webpack 和rollup 这样的打包工具碰到createRequire写法不知道该如何做。所以这样做图啥？



#### **怎样去创建一个“双重的包”同时包含CJS和ESM呢**

如果你现在有个库需要支持CJS和ESM，帮助你的用户遵循这些原则来创建一个“双重包”可以用在CJS，也可以很好的用在ESM。



1. ##### 给你的库提供个CJS版本	

   这是为了方便CJS用户。这也是为了确保你的库可以在老版本node下工作。

   （如何你用ts写的或别的语言转成的js，要转成CJS）	
   **如果你的库仅提供一个默认（unnamed）导出**，那就可以了，无需再往下看。

   你不需要做任何事情，如果你希望用户这么写 `import mylibrary from "mylibrary"`.

   如果你的用户希望用具名导出，e.g.` import {foo} from 'mylibrary'`接着往下看。

2. ##### **为CJS具名导出提供个简单ESM封装**

   （要知道为cjs写个esm的封装很简单，但是反过来为esm写个cjs封装就不太现实）

   ```
   import cjsModule from '../index.js';
   
   export const foo  = cjsModule.foo
   ```

   把ESM封装放到一个叫esm的子目录，接着再放一个写有`{"type":"module"}`的整行package.json文件。（你可以把封装文件名字改成.mjs 作为替代方案，这在node14中能很好的正常运行，但是一些工具碰到.mjs没法正常运行，所以我更倾向于创建个子目录）

   不要双重编译。如果你在编译typescript，你可以编译成CJS和ESM，但是这会引出个危害，用户可能偶然间同时单独地import 你的ESM和require() 你的CJS。（例如：假设一个叫着omg.mjs依赖于index.mjs,然后另外一个bbq.cjs依赖于index.cjs，然后你依赖于omg.mjs和bbq.cjs）

   Node 正常情况会消除重复的模块,但是Node不知道你的CJS和你的ESM是”一样“的文件，因此你的代码会执行两次，存有两份你的库的数据的副本。这会引起诡异的bugs。

3. ##### *在package.json文件中加个导出映射map**

像这样：

```
"exports":{
	"require":'.index.js',
	"import":'./esm/wrapper.js'
}
```



**注意：增加 导出map也是个语意层面的突破性变化。**默认情况下，你的用户能找到你的package然后require() 任何他们想要的脚本，甚至那些你内部使用的文件。导出map确保用户仅 require/import 你故意爆漏的入口文件。



这几乎可以肯定是个好的事情！但是这是一个[breaking change](https://medium.com/javascript-in-plain-english/is-promise-post-mortem-cab807f18dcc) 如果你允许用户在你的模块去import或者 require()别的文件，那你也得分别为他们设置入口文件。[详情请参照Node在ESM的文档](https://nodejs.org/api/esm.html)。



*在导出map目标文件中永远带着文件的后缀名*。指的是 `“index.js"` 不要仅仅写个`"index"` 或者像这样只写个文件目录 `./build`.

如果你遵循这些指导原则，你的用户will be ok.所有事情都会正常运行。





 











