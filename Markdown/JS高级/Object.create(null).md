## 定义

```javascript
Object.create(proto,[propertiesObject])
```

传入参数：

- proto：新创建的对象的原型对象；
- propertiesObject：可选参数，要添加到新对象的可枚举的属性（是其自身的属性，不是其原型链上的属性）；

```javascript
const person = {
  isHuman: false,
  printIntroduction: function() {
    console.log(`My name is ${this.name}. Am I human? ${this.isHuman}`);
  }
};

const me = Object.create(person);

me.name = 'Matthew'; 
// "name" is a property set on "me", but not on "person"
me.isHuman = true;  // 继承的属性可以被覆写
// inherited properties can be overwritten
me.printIntroduction();
// expected output: "My name is Matthew. Am I human? true"
```

## 比较

### ｛｝创建的对象

{}新创建的对象会继承 Object 自身的方法，可以在新对象上直接使用

```javascript
var o = {a：1};
console.log(o)
```

在chrome控制台打印如下：

![img](https://user-gold-cdn.xitu.io/2018/4/11/162b2eeff41e8f5d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### Object.create()创建的对象

Object.create(null)创建的对象除了自身的属性，原型链上没有任何属性，没有继承 Object 的任何东西；

```javascript
var o = Object.create(null,{
    a:{
        writable:true,
        configurable:true,
        value:'1'
    }
})
console.log(o)
console.log(o.toString()) // Uncaught TypeError
```

在chrome控制台打印如下：

![img](https://user-gold-cdn.xitu.io/2018/4/11/162b2ef2d7089a2f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```
var o = Object.create({},{
    a:{
        writable:true,
        configurable:true,
        value:'1'
    }
})
console.log(o)
```

在chrome控制台打印如下：

![img](https://user-gold-cdn.xitu.io/2018/4/11/162b2ef45967219d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```
var o = Object.create(Object.prototype,{
    a:{
        writable:true,
        configurable:true,
        value:'1'
    }
})
console.log(o)
```

在chrome控制台打印如下：

![img](https://user-gold-cdn.xitu.io/2018/4/11/162b2ef5f507c834?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 使用场景

很多源码作者会使用 Object.create(null) 来初始化一个新对象；

- 使用 create 创建的对象,没有任何属性，可以当作一个纯净的 map 来使用，自定义 hasOwnProperty、toString 等方法，不必担心会将原型链上同名方法覆盖；
- 使用 create（null）在使用 for...in 循环时不会再去遍历对象原型链上的属性；

```javascript
//Demo1:
var a= {...省略很多属性和方法...};
//如果想要检查a是否存在一个名为toString的属性，你必须像下面这样进行检查：
if(Object.prototype.hasOwnProperty.call(a,'toString')){
    ...
}
//为什么不能直接用a.hasOwnProperty('toString')?因为你可能给a添加了一个自定义的hasOwnProperty
//你无法使用下面这种方式来进行判断,因为原型上的toString方法是存在的：
if(a.toString){}

//Demo2:
var a=Object.create(null)
//你可以直接使用下面这种方式判断，因为存在的属性，都将定义在a上面，除非手动指定原型：
if(a.toString){}
```

总结：

1. 需要一个非常干净且高度可定制的对象当作数据字典的时候；
2. 想节省 hasOwnProperty 带来的一丢丢性能损失并且可以偷懒少些一点代码的时候；

