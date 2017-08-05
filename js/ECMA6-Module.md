# 模块化

ES6 模块的设计思想，是尽量的静态化，使得编译时就能确定模块的依赖关系，以及输入和输出的变量

通过 `export` 命令显式指定输出的代码，再通过 `import` 命令输入

## export 和 import

模块（通常就是一个文件）功能主要由两个命令构成： `export` 和 `import` 。 `export` 命令用于规定模块的对外接口(对象、函数、变量等)， `import` 命令用于输入其他模块提供的功能

### export

语法，每个模块只能有一个 `default` 的 `export`

```javascript

export { name1, name2, …, nameN };
export { variable1 as name1, variable2 as name2, …, nameN };
export let name1, name2, …, nameN; // also var
export let name1 = …, name2 = …, …, nameN; // also var, const

export default expression;
export default function (…) { … } // also class, function*
export default function name1(…) { … } // also class, function*
export { name1 as default, … };

export * from …;
export { name1, name2, …, nameN } from …;
export { import1 as name1, import2 as name2, …, nameN } from …;
```

- `export default` 命令其实只是输出一个叫做 `default` 的变量，本质是将该命令后面的值赋给 `default` 变量，所以它后面不能跟变量声明语句
- 有 `default` 在引用时可以自定义名称，而没有 `default` 时需要使用 {} 括起来且名称必须和 `export` 时指定的名称一致
- 每个文件里只能有一个 `default` 接口

`export` 命令规定的是对外的接口，必须与模块内部的变量建立一一对应关系

```javascript
// 报错，1只是一个值，不是接口
export 1;

// 正确
export default 42;

// 报错
var m = 1;
export m;
```

正确写法

```javascript
// 写法一
export var m = 1;

// 写法二
var m = 1;
export {m};

// 写法三
var n = 1;
export {n as m};
```

### import

有提升效果，静态执行，所以 **不能使用表达式和变量**，这些只有在运行时才能得到结果的语法结构。

语法

```javascript

import defaultMember from "module-name";  //导入 default export 方式导出的接口
import * as name from "module-name";    //加载整个模块
import { member } from "module-name";
import { member as alias } from "module-name";
import { member1 , member2 } from "module-name";
import { member1 , member2 as alias2 , [...] } from "module-name";
import defaultMember, { member [ , [...] ] } from "module-name";
import defaultMember, * as name from "module-name";
import "module-name";
```

不能使用表达式和变量

```javascript

// 报错
import { 'f' + 'oo' } from 'my_module';

// 报错
let module = 'my_module';
import { foo } from module;

// 报错
if (x === 1) {
  import { foo } from 'module1';
} else {
  import { foo } from 'module2';
}
```
