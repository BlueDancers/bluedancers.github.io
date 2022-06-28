---
title: JavaScript设计模式
categories:
  - JavaScript
tags:
  - 设计模式
toc: true
date: 2022-06-18
---

## JavaScript设计模式基础

JavaScript是一门经典动态类型语言，对变量类型的宽容给实际编码带来了很大灵活性。由于无需进行类型检测，我们可能尝试调用对象的任何方法，而无需去考虑它原本是否被设计拥有该方法。

​	这一切都建立在鸭子类型的概念上，鸭子类型：如果它走起路来像鸭子，叫起来像鸭子，那他就是鸭子

​	利用鸭子类型，我们就不必借助类型的帮助，实现一个动态语言专有原则：面向接口编程，而不是面向实现变成，例如一个对象，具备length属性，那我们就可以将其视为数组，而不需要关心它的实际类型。



### 多态

​	同一个操作作用于不同的对象上面，可以产生不同的解释和不同的执行结果

​	例如：小狗汪汪叫 小猫喵喵叫 他们都是动物，都会发生，但是各自发出的声音并不一样

​	其背后的思想是将“做什么”于“谁去做以及怎么样”分离开来，也就是将“不变的事物”于“变化的事物”分离开来。这给予了我们拓展程序的能力，程序看起来是可生长的，也是符合开放封闭原则的，相对于修改代码，增加代码显然优雅安全的多。

```js
function sound(animal) {
  animal.sound();
}

var Dog = function () {};
Dog.prototype.sound = () => {
  console.log("汪汪汪");
};

var Cat = function () {};
Cat.prototype.sound = () => {
  console.log("喵喵喵");
};

new Dog().sound()
new Cat().sound()
```



如果是强类型语言就需要借助继承来实现**向上转型**，从狗可以发出叫声转变为动物可以发出叫声，从而避免我们指定了发出声音对象是某一个类型，他就不可能被替换成为另一个类型。

​	多态最根本的作用就是通过把过程化的条件分支语句转化为对象的多态性，从而消除这些条件分支语句。

### 封装

封装的目的是将信息隐藏，封装应该被视为“任何形式的封装”，也就是说，封装不仅仅是隐藏数据，还包括隐藏实现细节，设计细节以及隐藏对象的类型。

### 原型编程

JavaScript本身就是基于原型的面向对象语言，它的对象系统就是使用原型模式来搭建的，在这里称之为原型编程范型业务更加合适。

​	在JavaScript中不存在类的概念，对象也并非从类中创建出来，所有的JavaScript对象都是从某个对象上复制出来的。

​	原型编程存在一个重要特性，即当对象无法响应某个请求的时候，就会把该请求委托给自己的原型；这里更好的说法是把请求委托给它的构造器的原型

​	在JavaScript中，一个function并不一定仅仅是一个普通函数，也可以是一个函数构造器，当使用new运算符来调用函数的时候，此时函数就是一个构造器。使用new运算符来创建对象的过程，实际上也只是先克隆`Object.prototype`，再进行一些其他额外操作的过程。



#### 原型链查找对象的过程

````js
var A = function () {};
A.prototype = { name: "sven" };

var B = function () {};
B.prototype = new A();

var b = new B();
console.log(b.name);
````

1. 首先尝试遍历对象b中的所有属性，但是没有找到name这个属性
2. 查找name属性的请求被委托到对象b的构造器原型，它被`b._proto_`记录并指向B.prototype，而B.prototype又直线new A()创建的对象
3. 再该对象中依旧没有找到name属性，于是请求又被委托到这个对象的构造器的原型A.prototype
4. 在A.prototype中找到了name属性，返回值



### 闭包

#### 闭包案例

```js
var func = function () {
  var a = 1;
  return function () {
    a++;
    alert(a);
  };
};

var ff = func();
ff();
ff();
ff();
```

​	局部变量在函数执行结束后将会被销毁，但是以上的例子中，局部变量a并没有消失，而是似乎一直在某个地方存活着。这是因为当执行func的时候，func返回了一个匿名函数的引用，它可以访问到func被调用时产生的环境，而局部变量所在的环境一直处于这个环境中。既然局部变量所处的环境还能被外界访问，这个局部变量就有了不被销毁的理由；在这样的闭包结构中，局部变量实现了生命的延续。



#### 闭包与面向对象

**过程与数据的结合**是形容面向对象中的**对象**时常用的表达

对象以方法的形式包含了过程，而闭包则是在过程中以环境的形式包含了数据

通常用面向对象实现的功能，用闭包也能实现，反之亦然。



闭包版本

```js
var app = function () {
  var value = 0;
  return {
    call: function () {
      value++;
      console.log(value);
    },
  };
};

var App = app();
App.call(); // 1
App.call(); // 2
App.call(); // 3
```



#### 对象版本

```js
var app = {
  value: 0,
  call: function () {
    this.value++;
    console.log(this.value);
  },
};

app.call(); // 1
app.call(); // 2
app.call(); // 3
```



#### 类版本

```js
var App = function () {
  this.value = 0;
};
App.prototype.call = function () {
  this.value++;
  console.log(this.value);
};

var app = new App();

app.call(); // 1
app.call(); // 2
app.call(); // 3
```



### 高阶函数

高阶函数是指最少满足下列条件之一的函数

- 函数可以作为参数被传递
- 函数可以作为返回值输出

​	JavaScript语言的函数显然满足高阶函数，在实际开发中将函数作为参数进行传递，让函数的执行结果返回一个另一个函数都是非常普遍的情况，例如函数执行的callback函数。

​	通过高阶特性，我们可以实现AOP，也就是面向切面编程

```js
// 面向切面编程
Function.prototype.before = function (beforeFn) {
    console.log("before");
    var _self = this;
    return function () {
        beforeFn.apply(this, arguments); // 执行before本身
        return _self.apply(this, arguments); // 返回函数本身
    };
};

Function.prototype.after = function (afterFn) {
    console.log("after");
    var _self = this;
    return function () {
        var ret = _self.apply(this, arguments); // 先执行before
        afterFn.apply(this, arguments); // 最后执行after
        return ret;
    };
};

var func = function () {
    console.log(2);
};
func = func
    .before(function () {
    console.log(1);
})
    .after(function () {
    console.log(3);
});

func();
```

- 首先执行`before`，打印‘before’，然后执行`after`，打印‘after’
- 执行func()，开始执行after，进入after闭包中，然后执行ret，进入before
- before中首先执行了自己beforeFn，打印‘1’，然后执行func本身，打印‘2’，并返回本身
- ret执行结束，开始执行afterFn，打印‘3’，返回func本身





#### 高阶应用 - 函数柯里化

​	柯里化又被称为部分求值，一个柯里化函数首先会接受一些参数，接收参数后不会立刻求职而是继续返回当前函数，之前传入的值在函数形成的闭包种被保存了起来。待函数真正需要求值的时候，之前传入的所有参数都会被一次性求值。



​	例如实现一个计算每个月花费多少钱的函数，但是在实现中，我们并不关心吗，每天花费了多少，只想知道月底花掉了多少，实际上只需要计算一次

```js
function currying(fn) {
    var args = [];
    return function () {
        if (arguments.length == 0) {
            var res = fn.apply(this, args);
            args = [];
            return res;
        } else {
            [].push.apply(args, arguments);
            return arguments.callee; // 当前正在执行的函数
        }
    };
}
var cost = (function () {
    var money = 0;
    return function () {
        for (var i = 0, l = arguments.length; i < l; i++) {
            money += arguments[i];
        }
        return money;
    };
})();
var cost = currying(cost);
cost(100)(100)(100)(100);
cost(100);
console.log(cost()); 500
cost(100);
console.log(cost()); 600
```



## 单例模式

​	要实现一个单例模式并不复杂，无非是用一个变量来标志是否已经为某个类创建过对象，如果是，则下一次获取该类的实例，直接返回之前创建的对象。

> vue2.x 中的vuex在页面与组件中进行挂载使用的就是单例模式

### 使用代理实现单例模式

```js
var createDiv = function (html) {
    this.html = html;
    this.init();
};
createDiv.prototype.init = function () {
    var div = document.createElement("div");
    div.innerHTML = this.html;
    document.body.appendChild(div);
};

var proxySingletonCreateDiv = (function () {
    var instance;
    return function (html) {
        if (!instance) {
            instance = new createDiv(html);
        }
        return instance;
    };
})();
var a = new proxySingletonCreateDiv("one");
var b = new proxySingletonCreateDiv("two");
console.log(a === b); // true
```



### JavaScript的单例模式

​	单例模式的核心是确保只有一个实例，比提供全局访问。在JavaScript中很多都会通过全局变量进行实现，但是JavaScript的全局变量并不是非常好的特性，在中大型项目中会存在命名冲突问题，所以应当尽量使用命名空间。



### 惰性单例

​	在未使用之前，相关逻辑不会被创建，并且只有第一次使用的时候才会创建，同时我们别忘记了单一职责原则

​	在下面的代码中，我们将创建单例与具体单例逻辑进行分离，这两个方法独立变化而且互不影响，这样避免了下次出现其他元素，我们需要将整个单例函数都复制一遍的情况，而是只需要创建对应的创建函数即可。

```js
function getSingle(fn) {
    let result;
    return function () {
        return result || (result = fn.apply(this, arguments));
    };
}

function createLoginLayer() {
    var div = document.createElement("div");
    div.innerHTML = "我是登录弹窗";
    div.style.display = "none";
    document.body.appendChild(div);
    return div;
}

var createSingLoginLayer = getSingle(createLoginLayer);

document.getElementById("loginBtn").onclick = function () {
    var loginLayer = createSingLoginLayer();
    loginLayer.style.display = "block";
};
```



### 小结

​	单例模式是一种简单，但是非常实用的模式，特别是惰性单例技术，在合适的时候再去创建对象，并且只创建唯一一个，同时我们将创建对象与管理单例的职责分开到不同方法中，这样的模式更加体验单例模式的优势。



## 策略模式

​	策略模式：定义一系列的算法，把它们一个个的封装起来，并且使它们可以相互替换。

​	案例：某个公司年终奖方式为基础工资乘以效绩等级，S为基础工资的4倍，A为基础工资的3倍，我们实用策略模式进行实现

```js
var performatceS = function () {}; // 效绩为S 工资算法
performatceS.prototype.calculate = function (salary) {
    return salary * 4;
};

var performatceA = function () {}; // 效绩为A 工资算法
performatceA.prototype.calculate = function (salary) {
    return salary * 3;
};

var Bonus = function () {
    this.salary = null; // 基础工资
    this.strategy = null; // 具体算法
};

Bonus.prototype.setSalary = function (salary) {
    this.salary = salary;
};
Bonus.prototype.setStrategy = function (strategy) {
    this.strategy = strategy;
};
Bonus.prototype.getBonus = function () {
    return this.strategy.calculate(this.salary);
};

var bonus1 = new Bonus();
bonus1.setSalary(10000);
bonus1.setStrategy(new performatceS());
console.log("效绩为A", bonus1.getBonus()); // 40000

bonus1.setStrategy(new performatceA());
console.log("效绩为A", bonus1.getBonus()); // 30000
```

### JavaScript中策略模式的体现

以上是类的实现方法，在JavaScript中我们可以通过函数进行实现，代码将会简洁很多

```js
var srtategies = {};
srtategies.S = function (salary) {
    return salary * 4;
};
srtategies.A = function (salary) {
    return salary * 3;
};
var calclateBonus = function (level, salary) {
    return srtategies[level](salary);
};
console.log("效绩为S", calclateBonus("S", 10000));
console.log("效绩为A", calclateBonus("A", 10000));
```



### 多态在策略模式中的体现

​	通过使用策略模式，我们可以消除程序中大量的ifelse语句，并将我们将具体逻辑与实际执行函数进行分离，执行函数没有计算能力，而是委托某个策略对象来完成奖金计算，这正是多态性的体现。



### 策略模式在表单校验的应用

​	在通过JavaScript表单校验的场景中,我们可以通过ifelse进行校验判断，但是这种方式不符合单一职责，开放封闭原则，我们可以通过策略模式来优化他，将通用的校验逻辑与具体校验条件进行解耦合。

```js
// 校验逻辑
/**
 * 如果同时设置了required与verify，将会忽略required
 * verify为自定义校验函数 可以理解为一旦写了verify,其他参数都不需要写了
 * @param data 被校验对象
 * @param validate 校验规则
 * @param isOne 是否校验到错误就立刻返回
 * @returns
 */
function starValidate(data, validate, isOne) {
  let errBack: any[] = []
  for (const key in data) {
    if (validate[key]) {
      if (validate[key].verify) {
        validate[key].verify({ data: data[key], allData: data }, (errMsg) => {
          if (errMsg) {
            errBack.push(errMsg)
          } else {
            errBack.push(validate[key].callback(data))
          }
        })
      } else {
        // 开启校验
        if (validate[key].required) {
          // 数据不存在
          if (!data[key]) {
            errBack.push(validate[key].callback(data))
          }
        }
      }
    }
    if (isOne && errBack.length != 0) {
      break
    }
  }
  console.log('处理结果', errBack)
  if (errBack.length == 0) {
    return Promise.resolve()
  } else {
    if (isOne) {
      return Promise.reject(errBack[0])
    } else {
      return Promise.reject(errBack)
    }
  }
}

// 校验条件
const validateRules = {
  cashingInstructions: {
    required: true,
    callback: () => ({ selector: '.open_prize', message: '请输入字段cashingInstructions' }),
  },
  lotteryDescription: {
    verify: ({ data }, err) => {
      if (data == '[]') {
        err({
          selector: '.launch_total',
          message: '请输入字段lotteryDescription',
        })
      }
    },
  },
}

let data = {
	cashingInstructions:'',
    lotteryDescription:'[]'
}

// 实现表单校验
 starValidate(data, validateRules, true)
```





### 策略模式的优缺点

优点

- 策略模式利用组合，委托和多态等技术与思想，可以有效避免多重条件选择语句
- 策略模式符合开放封闭原则，将具体逻辑单独封装，使其易于理解易于拓展
- 策略模式的策略函数可以再多项目之间复用，避免复制粘贴工作

缺点

- 相对于ifelse，策略模式的整体代码量会有所增加
- 调用者需要对策略细节可能了解，才能很好的使用该策略，这违反了最少知识原则，增加了使用成本



### 一等公民函数与策略模式

​	在函数作为一等公民的语言中，策略模式是隐形的具体策略的值就是函数变量。

​	在JavaScript这种将函数作为一等对象的语言中，策略模式已经融入到语言中，例如我们经常使用高阶函数来封装不同行为，并且将它传递到另一个函数中，当我们对这些函数发出“调用”的消息，不同的函数会返回不同的结果，函数对象的多态性来到更加简单。



## 代理模式

​	代理模式的关键是，当客户不方便直接访问一个对象或者不满足需要的时候，提供一个提升对象来控制对这个对象的访问，客户实际上访问的是替身对象。



小红想找心仪的对象让小明作为自己的媒人(代理人)

保护代理：张三找过来了，但是张三没车没房，小红便直接帮他拒绝

虚拟代理：介绍给小明是非常重要的事情，李四对小红有兴趣，给小明好处费，小明便在小红心情好的时候给其介绍（延迟到正常需要的时候再创建）



### 单一职责原则

​	对一个类/函数/对象而言，应该仅有一个引起它变化的原因，如果一个对象承担了多种职责，就意味着这个对象将变得巨大，引起它变化的原因将会有多种。面向对象估计设计将行为分布到细颗粒度的对象中，如果一个对象承担的职责过多，等于把这些职责耦合在一起，这种耦合会导致脆弱和低内聚的设计，当变化发生时，设计会遭到意外的破坏。

### 开放封闭原则

​	例如我们为了更好的性能将一些数据处理成为另外的数据格式，但是2年后上游帮助我们处理过了，我们不再需要额外处理，就不得不在改动原本函数中的代码

​	我们可以使用代理模式 达到不改动原对象的情况下，为其提供新的行为，他们各自变化，也不影响对象。



### 代理与本体接口的一致性

​	通常来说，代理对象对外提供的方法名称会与本体名称保持一致，这样可以在任何使用本体的地方替换成使用代理



### 代理模式-合并http请求

​	这是一个应用案例，文中的例子我在日常生活中也经历过，将每次点击都请求转变为收集2s类所有请求，并统一发送出去，发送请求时一个函数，何时发送，发送什么，时另一个函数，其中用到了节流函数来控制请求频率



### 代理模式 - 空间复杂度换取时间复杂度

面对非常复杂的计算逻辑，我们可以保存每一次的计算结果，下一个再来同样的参数可以直接走缓存，不再需要计算，这样增加空间，但是缩小了时间。



### 代理模式示例

```javascript
var muit = function () {
  var a = 1;
  for (let i = 0; i < arguments.length; i++) {
    a = a * arguments[i];
  }
  return a;
};

var plus = function () {
  var a = 0;
  for (let i = 0; i < arguments.length; i++) {
    a = a + arguments[i];
  }
  return a;
};

// 代理模式函数
var ceateProxyFactory = function (fn) {
  var cache = {};
  return function () {
    var args = Array.prototype.join.call(arguments, ",");
    if (args in cache) {
      console.log("存在缓存", args,cache);
      return cache[args];
    }
    cache[args] = fn.apply(this, arguments);
    return cache[args];
  };
};

var muitFun = ceateProxyFactory(muit);
var plusFun = ceateProxyFactory(plus);

console.log(muitFun(1, 2, 3, 4, 5)); // 120
console.log(muitFun(1, 2, 3, 4, 5)); // 走缓存 120
console.log(plusFun(1, 2, 3, 4, 5)); // 15
console.log(plusFun(1, 2, 3, 4, 5)); // 走缓存 15
```



### 总结

​	总体来说代理模式相对简单并且常用，就算一名开发人员没听过这个名词也会写出比较优秀的代理模式代码，并且代理模式不需要预先考虑，需要用到的时候再编写代理函数也不迟。



## 迭代器模式

### 内部迭代器

​	完全接手整个迭代过程，外部只需要初始调用即可，外界不需要关心迭代器的内部实现，但是这也是内部迭代器的缺点

​	例如JavaScript的`map` `forEach`





### 外部迭代器

​	外部迭代器必须显式的请求迭代下一个元素，外部迭代器增加了程序的复杂度，但是也增强了迭代器的灵活性。

```js
var current = 0;
var aa = function (obj) {
  var next = function () {
    current += 1;
  };
  var getItem = function () {
    return obj[current];
  };
  return {
    next,
    getItem,
  };
};
```

​	再具体业务中，使用何种迭代器并无优劣，根据实际场景而定。



### 总结

​	大部分语言已经内置了迭代器，并且使用频率高、门槛低；迭代器是一种非常简单设计模式，简单到大部分人不认为他是一种迭代器。



## 发布-订阅模式

​	发布-订阅模式它订阅了一种一对多的依赖关系,当一个对象的状态发生改变的时，所有依赖于它的对象都将得到通知



### 案例

​	小明看重了某一个小区的热门户型，并且得到消息，后期还会开放一批，但是时间未知，于是小明找到售楼处，预留了自己的电话号码，让售楼处在开发房源的时候通知他，同理，小张、小王都预留了手机号码，于是售楼处就会在房源发布的时候通知预留电话的客户。

​	客户想知道房源开售消息，于是他订阅了售楼处，售楼处得到消息后，第一时间将消息发布给订阅者，这样具备显而易见的优点。

- 小明不需要天天给售楼处打电话，在合适的时间售楼处会通知购房者
- 购房者于售楼处不再有强耦合关系



### 发布-订阅模式的作用

​	以上场景于程序中的异步场景是非常相似的，例如我们订阅ajax的error事件，我们无需关心异步运行期间的内部状态，只需要订阅需要的事件发生点即可。

​	另外发布-订阅模式可以取代对象之间硬编码的通知机制，一个对象不用再显式的调用另一个对象的某个接口。



### dom事件

​	我们使用dom绑定事件函数就是发布-订阅模式的实际应用，我们不知道用户会在什么时候点击点击，所以我们订阅了dom本身的click事件。



### 自定义发布-订阅事件

```js
var salesOffices = {};
salesOffices.clientList = [];
salesOffices.listen = function (key, fn) {
    // 创建订阅关联关系
    if (!this.clientList[key]) {
        this.clientList[key] = [];
    }
    this.clientList[key].push(fn);
};
salesOffices.trigger = function () {
    // 获取订阅数组
    var key = Array.prototype.shift.call(arguments);
    var fns = this.clientList[key];
    // 不存在订阅数组则直接返回
    if (!fns || fns.length === 0) {
        return false;
    }
    // 执行订阅数组
    for (let i = 0; i < fns.length; i++) {
        let fn = fns[i];
        fn.apply(this, arguments);
    }
};
// 小明订阅
salesOffices.listen("sq88", function (price) {
    console.log("我是小明，88平方");
    console.log("价格=", price);
});
salesOffices.listen("sq88", function (price) {
    console.log("我是小强，88平方");
    console.log("价格=", price);
});
// 小红订阅
salesOffices.listen('sq110', function (price) {
    console.log("我是小红，110平方");
    console.log("价格=", price);
});
salesOffices.trigger("sq88", 20000000);
salesOffices.trigger("sq110", 30000000);

// 我是小明，88平方
// 价格=20000000
// 我是小强，88平方
// 价格=20000000
// 我是小红，110平方
// 价格=30000000
```



### 取消订阅

​	取消订阅只需要将订阅数组中的指定订阅函数删除即可

```js
/**
 * key 订阅类型
 * fn 订阅函数
 */
salesOffices.remove = function (key, fn) {
    var fns = this.clientList[key];
    if (!fns) {
        return false;
    }
    if (!fn) {
        // 没有传入具体的回调地址，则取消所有订阅函数
        if (fns) {
            fns.length = 0;
        }
    } else {
        for (let i = 0; i < fns.length; i++) {
            const fnItem = fns[i];
            if (fnItem === fn) {
                fns.splice(i, 1); // 删除订阅函数回调
                break;
            }
        }
    }
};
```



### 关于网站登录的实际应用

场景：用户登录完成后，我们需要刷新不相邻模块的数据，这种异步问题，我们一般通过回调函数的方式解决

```js
login.succ(() => {
	header.setAvatar(data.avatar)
    nav.setAvatar(data.avatar)
    message.refresh()
    // ....
})
```

​	这种编写方式将组件数据于信息产生了强耦合关系，如果在未来，我们又增加了一个模块，则需要再次修改改回调函数

​	而通过发布-订阅模式，我们就可以在不同模块中订阅用户信息状态的变化，当登录成功的时候，登录模块发布消息到订阅他的模块中，至于各个模块做了什么，登录模块并不关心。

```js
login.listen('loginSucc',() => {
    // 登录成功，用户数据获取完毕
})
```



### 全局模式下的发布-订阅模式

​	全局状态下的发布-订阅可以在两个毫不相关的模块之间进行使用，这样就能保持模块的封装性

​	但是这里也需要留意一个问题，如果模块之间又太多的全局发布-订阅模式，就会造成消息流向混乱问题，这会导致维护上出现一些问题。

```js
var Event = (function () {
    var clientList = {};
    var listen;
    var trigger;
    var remove;
    listen = function (key, fn) {
        // 创建订阅关联关系
        if (!clientList[key]) {
            clientList[key] = [];
        }
        clientList[key].push(fn);
    };
    trigger = function () {
        // 获取订阅数组
        var key = Array.prototype.shift.call(arguments);
        var fns = clientList[key];
        // 不存在订阅数组则直接返回
        if (!fns || fns.length === 0) {
            return false;
        }
        // 执行订阅数组
        for (let i = 0; i < fns.length; i++) {
            let fn = fns[i];
            fn.apply(this, arguments);
        }
    };
    remove = function (key, fn) {
        var fns = clientList[key];
        if (!fns) {
            return false;
        }
        if (!fn) {
            // 没有传入具体的回调地址，则取消所有订阅函数
            if (fns) {
                fns.length = 0;
            }
        } else {
            for (let i = 0; i < fns.length; i++) {
                const fnItem = fns[i];
                if (fnItem === fn) {
                    fns.splice(i, 1); // 删除订阅函数回调
                    break;
                }
            }
        }
    };
    return {
        listen,
        trigger,
        remove,
    };
})();
var xm = function (price) {
    console.log("小明价格", price);
};
Event.listen("sq88", xm); // 订阅
Event.listen("sq110", xm); // 订阅
Event.remove("sq88", xm); // 取消订阅
Event.trigger("sq88", 220000); // 发布
Event.trigger("sq110", 2020000); // 发布
```



### JavaScript实现发布-订阅模式的便利性

#### 推模型

​	事情发生的时候，发布者会一次性将所有改变的状态与数据都推送给订阅者

#### 拉模型

​	事情发生的时候，发布者只会告诉所有订阅者，需要订阅者手动去拉去

​	而在JavaScript中，因为语言特性的存在，是我们可以非常方便的将所有参数通过arguments传入订阅者，所以我们使用推模型来完成消息的订阅与发布。



### 总结

#### 优点

- 对象之间的解耦合，可以帮助我们写出更好的应对异步编程的场景。
- 通过订阅-发布模式可以实现以此为特性的解决方案，例如MVVM。

#### 缺点

- 创建订阅-发布模式需要消耗一定的时间与内存。
- 订阅的消息会一直留存在内存中，产生了无意义的消耗。
- 过度使用订阅-发布会导致程序难以追踪与维护。

​	



## 命令模式

​	有时候需要向某些对象发送请求，但是并不知道请求的接收者是谁，也不知道被请求的操作是什么。此时希望用一种松耦合的方式来设计程序，使得请求发送者与请求接收者能够消除彼此之间的耦合关系

​	命令模式还需要支持撤销、排队等等操作

### 命令模式的例子-菜单程序（面向对象）

````js
var btn1 = document.getElementById("btn1");
var btn2 = document.getElementById("btn2");
var btn3 = document.getElementById("btn3");

var setCommand = function (btn, commm) {
    btn.onclick = function () {
        commm.execute();
    };
};

var MenuBar = {
    refresh: function () {
        console.log("刷新菜单目录");
    },
};
var SubMenu = {
    add: function () {
        console.log("增加子菜单");
    },
    del: function () {
        console.log("删除子菜单");
    },
};

var RefreshMenuBarCommand = function (receiver) {
    this.receiver = receiver;
};

RefreshMenuBarCommand.prototype.execute = function () {
    this.receiver.refresh();
};

var AddSubMenuCommand = function (receiver) {
    this.receiver = receiver;
};

AddSubMenuCommand.prototype.execute = function () {
    this.receiver.add();
};

var DelSubMenuCommand = function (receiver) {
    this.receiver = receiver;
};

DelSubMenuCommand.prototype.execute = function () {
    this.receiver.del();
};

var refreshMenuBarCommand = new RefreshMenuBarCommand(MenuBar);
var addSubMenuCommand = new AddSubMenuCommand(SubMenu);
var delSubMenuCommand = new DelSubMenuCommand(SubMenu);

setCommand(btn1, refreshMenuBarCommand); // 将div与方法做好绑定关系,同时约定一个触发指令点击btn1触发refresh内部预留的execute方法
setCommand(btn2, addSubMenuCommand);
setCommand(btn3, delSubMenuCommand);
````



### 命令模式的例子-菜单程序（面向函数）

```js
var bindClick = function (btn, func) {
    btn.onclick = func;
};
bindClick(btn1, MenuBar.refresh);
bindClick(btn2, SubMenu.add);
bindClick(btn3, SubMenu.del);
```



命令模式的由来，其实就是回调（callback）函数的一个面向对象的替代品

而再JavaScript这样函数作为一等公平的语言中，命令模式早已经融入到语言之中，函数本身就可以被四处传递，即时我们依旧需要请求“接收者”，那也未必使用面向对象的方式，闭包同样可以完成同样的功能。

### 命令模式的例子-菜单程序（闭包）

```js
var setCommand = function (btn, func) {
    btn.onclick = function () {
        func();
    };
};
var RefreshMenuBarCommand = function (receiver) {
    return function () {
        receiver.refresh();
    };
};

var refreshMenuBarCommand = RefreshMenuBarCommand(MenuBar);

setCommand(btn1, refreshMenuBarCommand);
```



### 命令模式 - 回放

```js
var Ryu = {
    attack: function () {
        console.log("攻击");
    },
    defense: function () {
        console.log("防御");
    },
    jump: function () {
        console.log("跳跃");
    },
    crouch: function () {
        console.log("蹲下");
    },
};
var makeCommand = function (receiver, state) {
    return function () {
        receiver[state]();
    };
};
var commandStack = []; // 保存命令堆栈
document.onkeypress = function (ev) {
    var commands = {
        119: "jump", // w
        115: "crouch", // s
        97: "defense", // a
        100: "attack", // d
    };
    if (commands[ev.keyCode]) {
        var command = makeCommand(Ryu, commands[ev.keyCode]);
        command(); // 执行命令
        commandStack.push(command); // 保存到堆栈
    }
};
document.getElementById("replay").onclick = function () {
    var command;
    while ((command = commandStack.shift())) {
        command();
    }
};
```



### 宏命令

​	宏命令是一组命令的集合，通过执行宏命令的方式，可以一次执行一批命令。

​	在创建命令模式的时候，增加一个add方法来增加命令，并保存到任务对略，最后调用execute方法依次执行即可



### 总结

​	命令模式在JavaScript中因为高阶函数的存在，让其不太显眼，本质上他是将具体调用与调用的具体逻辑进行分离，具体逻辑就是命令的体现。



## 组合模式

​	组合模式需要通过对象的多态性进行体现，是的用户对单个对象和组合对象的使用具有一致性

![image-20220618165949858](https://www.vkcyan.top/image-20220618165949858.png)

### 示例

​	这里定义了一个通用函数execute来作为组合模式的桥梁，完成对象树的构建。

```js
<button id="button">按我</button>
<script>
    var MacroCommand = function () {
        return {
            commandsList: [],
            add: function (command) {
                this.commandsList.push(command);
            },
            execute: function () {
                for (let i = 0; i < this.commandsList.length; i++) {
                    this.commandsList[i].execute();
                }
            },
        };
    };
var openAcCommand = {
    execute: function () {
        console.log("打开空调");
    },
};
var openTvCommand = {
    execute: function () {
        console.log("打开电视");
    },
};
var openSoundCommand = {
    execute: function () {
        console.log("打开音响");
    },
};
var macroCommand1 = MacroCommand();
macroCommand1.add(openTvCommand);
macroCommand1.add(openSoundCommand);

var closeDoorCommand = {
    execute: function () {
        console.log("关门");
    },
};
var openPcCommand = {
    execute: function () {
        console.log("打开电脑");
    },
};
var openQQCommand = {
    execute: function () {
        console.log("登录QQ");
    },
};
var macroCommand2 = MacroCommand();
macroCommand2.add(closeDoorCommand);
macroCommand2.add(openPcCommand);
macroCommand2.add(openQQCommand);

var macroCommand = MacroCommand();
macroCommand.add(openAcCommand); // 如果是基本对象,就是直接触发到其本身的execute方法
macroCommand.add(macroCommand1); // 如果是复杂对象,则触发到下一级的execute,然后以深度优点遍历直到最底部的基本对象
macroCommand.add(macroCommand2);

var setCommand = (function (command) {
    document.getElementById("button").onclick = function () {
        command.execute();
    };
})(macroCommand);
</script>
```

​	组合模式最大的优点在于可以一致地对待组合对象与基本对象。客户不需要关心当前处理的是谁，只要它是一个命令，并且有execute方法，这个命令就可以被执行。

​	得益于JavaScript是动态类型语言，对象的多态性与生俱来，不会存在编辑器检查，所以我们实现组合模式并不需要编写抽象类，只需要保证组合对象与叶对象拥有相同的方法即可，并且用鸭子类型的思想进行接口检查



### 组合模式-扫描文件夹

​	我们通过组合模式，可以做到更新树的结构，但是却不需要改变原有代码，这符合开放封闭原则

```js
var Folder = function (nameParams) {
    let name = nameParams;
    let files = [];
    function add(file) {
        files.push(file);
    }
    function scan() {
        console.log("开始扫描文件夹", name);
        for (let i = 0; i < files.length; i++) {
            files[i].scan();
        }
    }
    return {
        add,
        scan,
    };
};
var File = function (nameParams) {
    let name = nameParams;
    function add() {
        throw new Error("文件中不能增加文件夹");
    }
    function scan() {
        console.log("开始扫描文件", name);
    }
    return {
        add,
        scan,
    };
};
var folder = new Folder("学习资料");
var folder1 = new Folder("JavaScript");
var folder2 = new Folder("jQuery");
var file1 = new File("JavaScript设计模式与开发实践");
var file2 = new File("精通jQuery");
var file3 = new File("重构与模式");

folder1.add(file1);
folder2.add(file2);
folder.add(folder1);
folder.add(folder2);
folder.add(file3);

var folder3 = new Folder("Nodejs");
var file4 = new File("深入浅出Node.js");
folder3.add(file4);
var file5 = new File("JavaScript语言精髓与编程实战");
folder.add(folder3);
folder.add(file5);

folder.scan();
```



### 一些需要注意的地方

- 组合模式不是父子关系
- 对一组叶对象的操作必须具有一致性，只有用一致的方式对待列表中的每一个叶对象，才适合使用组合模式
- 如果存在一个叶子元素存在多个父级，可能就需要管理映射关系，避免子元素多次被执行

### 总结

​	组合模式可以让我们把相同的操作应用在组合对象和单个对象上。

- 组合模式的美国和对象看起来都和其他对象差不多，他们的区别只能在运行中才能显现出来，这会使代码难以理解
- 组合模式会大量创建变量，会让系统负担不起



## 模板方法模式

​	模板方法是一种只需要继承就可以实现的非常简单的模式（多态性）

- 模板方法由2部分组成，第一部分是抽象父类，第二部分是具体的实现子类
- 在模板方法模式中，子类实现中的相同部分被上移到父类中，而将不同的部分留到子类中进行实现，这很好的体现了泛化的思想。

​	在模板方法中，子类实现中的相同部分被上移到父类中，而将不同的部分留给子类实现，子类可以复写其具体实现。

### 咖啡与茶

```js
var Beverage = function () {};
Beverage.prototype.boilWater = function () {
    console.log("把水煮沸");
};
Beverage.prototype.brew = function () {};
Beverage.prototype.pourInCup = function () {};
Beverage.prototype.addCondiments = function () {};
Beverage.prototype.init = function () {
    this.boilWater();
    this.brew();
    this.pourInCup();
    this.addCondiments();
};

var Coffee = function () {};
Coffee.prototype = new Beverage();
Coffee.prototype.brew = function () {
    console.log("沸水冲泡咖啡");
};
Coffee.prototype.pourInCup = function () {
    console.log("把咖啡倒进杯子");
};
Coffee.prototype.addCondiments = function () {
    console.log("加糖和牛奶");
};
var Coffee = new Coffee();
Coffee.init();

var Tea = function () {};
Tea.prototype = new Beverage();
Tea.prototype.brew = function () {
    console.log("沸水冲泡咖啡");
};
Tea.prototype.pourInCup = function () {
    console.log("把咖啡倒进杯子");
};
Tea.prototype.addCondiments = function () {
    console.log("加糖和牛奶");
};
var Tea = new Tea();
Tea.init();
```

​	在以上例子中`Beverage.prototype.init`就是所谓的模板方法，因为该帆帆中封装了子类的算法框架。

### 抽象类

模板方法模式是一种严格依赖抽象类的设计模式。

抽象帆帆被声明在抽象类中，抽象方法并没有具体的实现过程，是一些哑巴方法

如果每个子类中都有一些同样的具体实现方法，那么这些方法也可以选择放在抽象类中，这样可以节省代码以达到复用的效果，这些方法被叫做具体方法。



### 钩子方法

​	模板方法是固定不变的，但是在某些场景下却又要求他变化，有什么办法可以让子类不受这个约束呢？

​	我们可以使用钩子方法来实现，放置一个钩子在特定的逻辑。例如以上的例子中咖啡有些人不希望加调料

```js
// ...
Beverage.prototype.custonmerWantsCondiments = function () {
    return true; // 默认需要调料
};
Beverage.prototype.init = function () {
    this.boilWater();
    this.brew();
    this.pourInCup();
    if (this.custonmerWantsCondiments()) {
        this.addCondiments();
    }
};

// ...
Coffee.prototype.custonmerWantsCondiments = function(){
    return window.confirm('请问需要调料吗？')
}
var Coffee = new Coffee();
Coffee.init();
```



### 好莱坞原则

​	好莱坞无疑是演员的天堂，但好莱坞也有很多找不到工作的新人演员，许多新人演员在好莱坞把简历投递过去之后，只能回家等电话，有些等不及的就会打电话过去问，而好莱坞每次都会回答：“不太来找我，有消息我会通知你”

​	在设计中，这种模式被称为好莱坞原则，在程序中，高层组件决定什么时候以何种方式使用这些底层组件

​	这种模式在模板方法模式中很常见，在发布订阅模式，回调函数都非常适用，就像出租车司机告诉你别问我还有多远到，到了我会告诉你。



### 小结

​	模板方法是一种典型的通过封装变化提高系统拓展性的设计模式。我们把部分抽象逻辑抽象到父类的模板方法，而子类的方法具体怎么实现是可变的，于是我们把这部分变化的逻辑封装到子类中。



## 享元模式

### 案例

​	假设有一个服装工厂，目前里面50个男士样式，50个女士样式，他们都需要模特穿上拍宣传片，正常情况下就需要分别50个模特来拍照，程序实现逻辑为

```js
var Model = function (sex, underwear) {
    this.sex = sex;
    this.underwear = underwear;
};
Model.prototype.takePhoto = function () {
    console.log(`${this.sex}:${this.underwear}`);
};
for (let i = 0; i < 50; i++) {
    var maleModel = new Model("male", `underwear${i}`);
    maleModel.takePhoto();
}
for (let i = 0; i < 50; i++) {
    var femaleModel = new Model("female", `underwear${i}`);
    femaleModel.takePhoto();
}
```

​	现在分别50种内衣，一共有100个对象，后面如果越来越多，10000个，可能就会导致程序崩溃。其实我们仔细想想就会发现，我们不需要一套内衣都搭一个模特，只需要一个男模特，一个女模特就够了，我们根据这样的思路再次改写代码

```js
var Model = function (sex) {
    this.sex = sex;
};
Model.prototype.takePhoto = function (underwear) {
    console.log(`${this.sex}:${underwear}`);
};
var maleModel = new Model("male");
var femaleModel = new Model("female");

for (let i = 0; i < 50; i++) {
    maleModel.takePhoto(`underwear${i}`);
}
for (let i = 0; i < 50; i++) {
    femaleModel.takePhoto(`underwear${i}`);
}
```

​	改造之后，我们只需要两个对象就实现了相同的功能，并且开销是固定的2个，就算10000间衣服也不会出现问题

### 外部状态与内部状态

- 享元模式的目标是尽量减少共享对象的数量，是优先使用时间换取空间的优化模式

### 上传文件的例子

```js
var id = 0;
window.startUpload = function (uploadType, files) {
    for (let i = 0; i < files.length; i++) {
        let file = files[i];
        var uploadObj = new Upload(uploadType, file.fileName, file.fileSize); // 实例化传入变量
        uploadObj.init(id++); // init中创建dom
    }
};
var Upload = function (uploadType, fileName, fileSize) {
    this.uploadType = uploadType;
    this.fileName = fileName;
    this.fileSize = fileSize;
    this.dom = null;
};
Upload.prototype.init = function (id) {
    var that = this;
    this.id = id;
    this.dom = document.createElement("div");
    this.dom.id = id;
    this.dom.innerHTML = `<span>文件名称：${this.fileName} 文件大小：${this.fileSize} 上传方式:${this.uploadType}</span><button class="delFile">删除</button>`;
    this.dom.querySelector(".delFile").onclick = function () {
        that.delFile();
    };
    document.body.appendChild(this.dom);
};
Upload.prototype.delFile = function () {
    if (this.fileSize < 3000) {
        return this.dom.parentNode.removeChild(this.dom);
    }
    if (window.confirm("确定删除文件吗？" + this.fileName)) {
        return this.dom.parentNode.removeChild(this.dom);
    }
};
startUpload("plugin", [
    {
        fileName: "1.txt",
        fileSize: 1000,
    },
    {
        fileName: "2.txt",
        fileSize: 2000,
    },
]);
startUpload("flash", [
    {
        fileName: "5.txt",
        fileSize: 6000,
    },
    {
        fileName: "6.txt",
        fileSize: 7000,
    },
]);
```

​	在以上例子中，我们上传多少文件就需要创建多少个对象，接下来我们用享元模式重构以上代码

```js
var Upload = function (uploadType, fileName, fileSize) {
        this.uploadType = uploadType;
      };
      Upload.prototype.delFile = function (id) {
        let carry = uploadManager.setExternalState(id);
        return carry.dom.parentNode.removeChild(carry.dom);
      };
      var UploadFactoy = (function () {
        var createFlyWeghtObjs = {};
        return {
          create: function (uploadType) {
            if (createFlyWeghtObjs[uploadType]) {
              return createFlyWeghtObjs[uploadType];
            }
            createFlyWeghtObjs[uploadType] = new Upload(uploadType);
            return createFlyWeghtObjs[uploadType];
          },
        };
      })();

      var uploadManager = (function () {
        var uploadDataBase = {};
        return {
          add: function (id, uploadType, fileName, fileSize) {
            var flyWeight = UploadFactoy.create(uploadType);
            var dom = document.createElement("div");
            dom.innerHTML = `<span>文件名称：${fileName} 文件大小：${fileSize} 上传方式:${uploadType}</span><button class="delFile">删除</button>`;
            dom.querySelector(".delFile").onclick = function () {
              flyWeight.delFile(id);
            };
            document.body.appendChild(dom);
            uploadDataBase[id] = {
              fileName,
              fileSize,
              dom,
            };
            console.log(uploadDataBase);
            return flyWeight;
          },
          setExternalState: function (id) {
            return uploadDataBase[id];
          },
        };
      })();
      var id = 0;
      window.startUpload = function (uploadType, files) {
        for (let i = 0; i < files.length; i++) {
          let file = files[i];
          uploadManager.add(++id, uploadType, file.fileName, file.fileSize);
        }
      };
      startUpload("plugin", [
        {
          fileName: "1.txt",
          fileSize: 1000,
        },
        {
          fileName: "2.txt",
          fileSize: 2000,
        },
      ]);
      startUpload("flash", [
        {
          fileName: "5.txt",
          fileSize: 6000,
        },
        {
          fileName: "6.txt",
          fileSize: 7000,
        },
      ]);
```

​	通过享元模式创建后，实例化的对象因为工厂模式的存在只创建了2个。

### 享元模式的适用性

- 一个程序中使用了大量相似的对象，并且这些对象大多数状态是可以成为外部状态的
- 可以使用共享对象取代大量对象，将外部状态剥离出去

### 对象池

​	对象池维护一个装载空闲对象的池子，如果需要对象的时候，不会再去new，还是从对象池中进行获取，如果对象池不存在可用对象，则创建一个新对象，当获取处的对象完成了他的职责之后，再次进入池子等待下次获取

地图标点demo	

​	进入地图软件后，首先搜索A地点，存在2个坐标点，通过工厂函数便创建了2个，而后搜索了B地点，存在6个坐标，便会利用之前空闲的2个，再新增加4个坐标点

​	对象池的模式与享元模式类，知识没有状态分离的过程。

```js
var objectPoolFactory = function (createObjFun) {
    var objectPool = [];
    return {
        create: function () {
            // 创建对象
            if (objectPool.length === 0) {
                // 如果对象池中没有对象，就创建一个新的对象
                return createObjFun.apply(this, arguments);
            } else {
                // 如果对象池中有对象，就从对象池中取出一个对象
                return objectPool.shift();
            }
        },
        recover: function (obj) {
            // 回收对象
            objectPool.push(obj);
        },
    };
};
```

### 总结

​	享元模式主要为解决性能问题，在一个存在大量相似对象的系统中，享元模式可以很好的解决大量对象带来的性能问题。



## 职责链模式

​	使多个对象都有机会处理请求，从而避免请求的发送者与接收者之间的耦合关系，将这些对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止。

​	职责链优点：请求发送者只需要知道链中的第一个节点，从而弱化了发送者和一组接收者之间的强联系



