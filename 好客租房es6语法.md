@[TOC]
# 基础语法

``` 
<!DOCTYPE html>
<meta charset="utf-8">

<script>
    // var 定义全局变量
    for (var i = 0; i < 5; i++) {
        console.log(i);
    }
    console.log("循环外：" + i);

    // let 控制变量作用域
    for (let j = 0; j < 5; j++) {
        console.log(j);
    }
    console.log("循环外：" + j);

    // const声明常量 不可修改
    const a = 1;
    console.log("a=", a);
    a = 2;
    console.log("a=", a)

    // 字符串拓展函数
    console.log("hello world".includes("hello"));
    console.log("hello world".startsWith("hello"));
    console.log("hello world".endsWith("world"));

    // 字符串模板保留换行源格式
    let str = `
        hello
        world
    `;
    console.log(str);

    // 新定义数组解构获取顺序获取值
    let arr = [1, 2, 3]
    const [x, y, z] = arr;
    console.log(x, y, z)
    const [m] = arr;
    console.log(m)

    // 新定义对象结构顺序获取属性值
    const person = {
        name: 'wj',
        age: 21,
        language: ['java', 'js', 'php']
    }
    const {name, age, language} = person
    console.log(name, age, language)
    const {name: n, age: a, language: l} = person;
    console.log(n)

    // 函数默认值
    function add(a, b = 1) {
        // b = b || 1;
        return a + b;
    }

    console.log(add(10))

    // 箭头函数
    var print = obj => (console.log(obj))     //一个参数
    var sum = (a, b) => a + b                  // 多个参数
    var sayHello1 = () => console.log("hello")     // 没有参数
    var sayHello2 = (a, b) => {        // 多个函数
        console.log("hello")
        console.log("world")
        return a + b;
    }

    // 对象函数属性简写
    let persons = {
        name: "jack",
        eat: food => console.log(persons.name + "再吃" + food), // 这里拿不到this
        eat1(food) {
            console.log(this.name + "再吃" + food)
        },
        eat2(food) {
            console.log(this.name + "再吃" + food)
        }
    };
    persons.eat("西瓜")

    // 箭头函数结合解构表达式
    var hi = ({name}) => console.log("hello," + name)
    hi(persons)

    // map reduce
    let array = ['1', '2'].map(s => parseInt(s));      //将原数组所有元素用函数处理哇年后放入新数组返回
    let array1 = arr.map(function (s) {
        return parseInt(s);
    })
    console.log(array)
    let num = [1, 20, 6, 5];
    console.log(num.reduce((a, b) => a + b))     // 从左到友依次用reduce处理，并把结果作为下次reduce的第一个参数
    let result = arr.reduce((a, b) => {
        return a + b;
    }, 1); //接受函数必须，初始值可选


    // 扩展运算符
    console.log(...[2, 3], ...[1, 20, 6, 5], 0)       // 数组合并
    let add1 = (x, y) => x + y;
    console.log(add1(...[1, 2]))
    const [f, ...l] = [1, 2, 3, 4, 5]        //结合结构表达式
    console.log(f, l)
    console.log(...'heool')     //字符串转数组

    // promise 异步执行
    const p = new Promise((resolve, reject) => {
        setTimeout(() => {
            const num = Math.random();
            // 随机返回成功或失败
            if (num < 0.5) {
                resolve("成功 num=" + num)
            } else {
                reject("失败 num=" + num)
            }
        }, 300)
    })
    p.then(function (msg) {
        console.log(msg)
    }).catch(function (msg) {
        console.log(msg)
    })

    // set map
    let set = new Set()
    set.add(1)  // clear delete has forEach(function(){}) size
    set.forEach(value => console.log(value))
    let set2 = new Set([1, 2, 2, 1, 1])
    // map为<object,object>
    const map = new Map([
        ['key1', 'value1'],
        ['value2', 'value2']
    ])
    const set3 = new Set([
        ['t', 't'],
        ['h', 'h']
    ])
    const map2 = new Map(set3)
    const map3 = new Map(map)
    map.set('z', 'z') // clear delete(key) has(key) forEach(function(value,key){}) size values keys entries
    for (let key of map.keys()) {
        console.log(key)
    }
    console.log(...map.values())

    // 类的基本用法
    class User {
        constructor(name, age = 20) {
            this.name = name;
            this.age = age;
        }

        sayHi() {
            return "hi"
        }

        static isAdult(age) {
            if (age >= 18) {
                return "成年人"
            }
            return "未成年人"
        }
    }

    class zhangsan extends User {  // 类的继承
        constructor() {
            super("张三", 10)
            this.address = "上海"
        }
        test(){
            return "name="+this.name;
        }
    }

    let user = new User("张三")
    let zs = new zhangsan()
    console.log(user)
    console.log(user.sayHi())
    console.log(User.isAdult(20))
    console.log(zs.name, zs.address)
    console.log(zs.sayHi())


    // Generator函数
    function* hello() {
        yield "h"
        yield "e"
        return "a"
    }

    let h = hello();
    for (let obj of h) {      //循环遍历或next遍历
        console.log("===" + obj)
    }
    console.log(h.next())
    console.log(h.next())
    console.log(h.next())


    // 修饰器 修改类的行为
    @T
    class Animal {
        constructor(name,age=20){
            this.name=name;
            this.age=age;
        }
    }
    function T(target){
        console.log(target);
        target.contry = "china" //通过修饰器添加的属性是静态属性
    }
    console.log(Animal.contry)
    // 无法运行，需要转码：将ES6活ES2017转为ES5使用（将箭头函数转为普通函数）
</script>

</html>
```
# umi转码
> node -v  v8.12.0
	npm i yarn tyarn -g    tyarn使用淘宝源
	tyarn -v  1.16.0      若报错通过yarn global bin获取路径加入Path
	tyarn global add umi
	umi     

index.js
``` 
// 修饰器
function T(target) {
  console.log(target);
  target.country="中国"
}
@T
class People{
  constructor(name,age=20){
    this.name=name;
    this.age=age;
  }
}
console.log(People.country);

import Util from './Util';
console.log(Util.sum(10, 5));

```
Util.js

``` 
class Util {
    static sum = (a,b) => {
        return a + b;
    }
}

export default Util;
```
> umi dev,通过http://localhost:8000/ 查看控制台

# reactjs
> tyarn init -y
	tyarn add umi --dev
	tyarn add umi-plugin-react --dev
详情见：
https://github.com/OneJane/blog