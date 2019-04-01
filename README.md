# Vue_MVVM
简单实现Vue的双向绑定以及详细注释
### 1.任务划分

（1）实现数据绑定，实现View和Model的绑定，View中与Model相关的节点，可以完成初始化。

（2）实现Model随View的变化而变化，简称Model动态响应View。

（3）实现View随Model的变化而变化，简称View动态响应Model。
![流程图.png](https://upload-images.jianshu.io/upload_images/14720179-b528e16e23eff6cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 2.实现数据绑定

- 任务描述

  ```html
  <div id='chai'>
    <input id='in' v-model="test">
    {{test}}
  </div>
  
  <script>
    var vm = new Vue({
      el: 'chai',
      data: {
        test: 'hello two_way'
      }
    })
   </script>
  ```

  希望实现，上面这段html页面打开时可以初始化DIV的值和{{test}}的值，如下图所示：
![初始化.png](https://upload-images.jianshu.io/upload_images/14720179-43a6ce10bac0269e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


主要的功能模块

（1）Dom劫持

利用DocumentFragment劫持你想要处理的元素，可以将DocumentFragment简单理解为一个Dom容器，将处理过后的节点存入这个容器中，待所有节点处理完毕之后，再将容器中的所有的节点利用appendChild方法重新插入到响应的根节点之中。

```javascript
//劫持你想要处理的元素，将元素外部的包裹元素传入
  function domToFragment(node,vm){
    var domf = document.createDocumentFragment();
    var child;
    while(child = node.firstChild){
      //compile(child, vm);     //将dom和vue实例中的data相互关联
      domf.appendChild(child);
    }
    return domf;
  }
```

显而易见，上面这个方法，当你将根节点作为入参时，该方法会对根节点下的所有子节点一一处理（这里的compile方法就是编译DOM，将其与Model相互关联，下一步讲解），最后再将存有编译后的所有子节点的容器返回。

（2）编译DOM

这部分，主要是针对不同类型的DOM，对其进行分别进行处理，并将其关联的Model的值赋予给相关的Dom。

```javascript
function compile(node, vm){
    var reg = /\{\{(.*)\}\}/;//能够匹配{{test}}这种dom的正则
    //依据节点的类型做不同的处理
    //若节点为一般普通元素
    if(node.nodeType === 1){
      var attr = node.attributes;//得到该元素的所有属性
      for(let i=0;i<attr.length;i++){
        if(attr[i].nodeName === 'v-model'){
          let name = attr[i].nodeValue;
          node.value = vm.data[name];
          node.removeAttribute('v-model');
        }
      }
    }

    //节点类型是text。也就是{{...}}这种形式的话
    if(node.nodeType === 3){
      if(reg.test(node.nodeValue)){
        var name = RegExp.$1;
        name = name.trim();
        node.nodeValue = vm.data[name];
      }
    }
  }
```

显而易见，上面的compile方法中以node.nodeType为区分，分别进行处理，并将Model的值赋予给不同的节点，以本文的示例来说，input就是属于node.nodeType = 1，{{test}}就是属于node.nodeType = 2。

（3）Vue的构造函数

这部分主要就是将根节点下面的所有子节点进行编译，编译后再重新插入到根节点下面。

```javascript
function Vue(options){
    this.data = options.data;
    this.id = options.el;
    var root = document.getElementById('chai');
    var dom = domToFragment(root, this);
    root.appendChild(dom);
 }
```

由此已经实现了数据绑定，可以讲这几段代码复制到一块，看书否可以达到我们在这一节开头的期望结果，完成数据的初始化。

### 3.实现Model动态响应View

- 任务描述

以本文的示例来说，就是当你在input框中改变其值时，其利用v-model关联的Model的值也随之改变，显然这里需要用到Dom监听，再进一步思考平日里我们使用vue的时候，若需要取data中的test属性的值，会使用this.test，但是仔细一想Vue实例中并没有test这个属性，那么为什么this.test可以取到data中的值呢？这里用到的是Object.defineProperty这个方法，在定义访问器属性的时候将data中的属性与Vue实例中的属性相互关联。

（1）监听Dom的input事件

为input加上监听器，当输入值的同时改变Model的值。

```javascript
function compile(node, vm){
    var reg = /\{\{(.*)\}\}/;//能够匹配{{test}}这种dom的正则
    //依据节点的类型做不同的处理
    //若节点为一般普通元素
    if(node.nodeType === 1){
      var attr = node.attributes;//得到该元素的所有属性
      for(let i=0;i<attr.length;i++){
        if(attr[i].nodeName === 'v-model'){
          let name = attr[i].nodeValue;
//************以下新增代码**************
          //添加dom监听，将发生变化的dom的值赋给vue实例
          node.addEventListener('input', e => {
            vm.data[name] = e.target.value;
          })
//************以上新增代码**************
		 node.value = vm.data[name];
          node.removeAttribute('v-model');
        }
      }
    }
```

显而易见，当监听到输入时，将Model更新为最新的输入值。

（2）关联Vue实例属性和data属性

这部分主要是利用Object.defineProperty定义访问器属性的时候，将两者关联。

```javascript
	//定义访问器属性
    function defProperty(obj, key, val) {
      Object.defineProperty(obj, key, {
        get: function(){
          return val;
        },
        set: function(newVal){
          if(newVal === val) return;
          val = newVal;
          console.log(val)
        }
      })
    }
    
    //将对于data中属性的操作都转化为对vm属性的操作，所以，原本vm.data[key]就可以直接写		为vm.key这种直接的形式
    function translateDataToVm(obj,vm) {
      Object.keys(obj).forEach(key => {
        defProperty(vm, key, obj[key])
      });
    }
```

简单理解上面的方法就是当你调用this.test取值的时候，会执行Vue实例的get方法，而在get方法中返回的就是data中的test属性的值。当你调用this.test赋值的时候，会执行Vue实例的set方法，在这个方法中对data的test属性进行赋值。

所以编译代码要进一步优化，所有取值、赋值的操作都不需要再调用vm.data[name]，而是直接使用vm.name。

```javascript
function compile(node, vm){
    var reg = /\{\{(.*)\}\}/;//能够匹配{{test}}这种dom的正则
    //依据节点的类型做不同的处理
    //若节点为一般普通元素
    if(node.nodeType === 1){
      var attr = node.attributes;//得到该元素的所有属性
      for(let i=0;i<attr.length;i++){
        if(attr[i].nodeName === 'v-model'){
          let name = attr[i].nodeValue;

//************以下新增代码**************
          //添加dom监听，将发生变化的dom的值赋给vue实例
          node.addEventListener('input', e => {
            vm[name] = e.target.value;
          })
//************以上新增代码**************

//************以下变化代码**************
          node.value = vm[name];
//************以上变化代码**************

          node.removeAttribute('v-model');
        }
      }
    }

    //节点类型是text。也就是{{...}}这种形式的话
    if(node.nodeType === 3){
      if(reg.test(node.nodeValue)){
        var name = RegExp.$1;
        name = name.trim();

        //************以下变化代码**************
        node.nodeValue = vm[name];
        //************以上变化代码**************
      }
    }
  }
```

这一部分的最后一步就是在构造函数中调用translateDataToVm方法将Vue实例的属性与data的属性相互关联。

```javascript
function Vue(options){
    this.data = options.data;
    this.id = options.el;

//************以下新增代码**************
    //为vm实例添加访问器属性，对vm的相关属性的读写都会与data中的属性相互关联
    translateDataToVm(this.data, this);
//************以上新增代码**************

    var root = document.getElementById('chai')
    var dom = domToFragment(root, this);
    root.appendChild(dom);
  }
```

4.实现View动态响应Model

- 任务描述

这部分主要是利用发布-订阅模式，当Model发生改变时，发布者会通知所有订阅了该主题的订阅者进行更新。这里的主题就是Model（本文中的test），订阅者指的就是每一个与Model相互关联的Dom节点（input和{{test}}），可能有多个节点都关联了一个Model，所以这些订阅者应该统一管理（放入一个数组中），于是可以看出，对于不同Model的订阅者应该是管理在不同的数组中，当Model发生变化时，通知对应的数组中的每一个订阅者更新自己的值。
![订阅发布模式图解.png](https://upload-images.jianshu.io/upload_images/14720179-cfdc84b6c0e5fb08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


（1）订阅者定义

订阅者无非就是得到发布者的指示之后更新自己的数据，其中最需要注意的一点是，需要有一个标志位来明确只有在订阅者初始化的时候才会将自己放入到某个主题的订阅数组之中。

```javascript
function Watcher(vm, node, name){
    Subscripters.target = this;//Subscripters大兄弟，我是'name'的订阅者，初来乍到，记得把我加入到主	   	   题'name'的数组中哦。
    this.vm = vm;
    this.node = node;
    this.name = name;
    this.update();//这一步主要完成更新和加入更新数组
    Subscripters.target = null;//Subscripters大兄弟，你既然已经把我加入了主题'name'的更新数组，那么下次	 执行update方法的时候就不需要再加入一次啦！！
  }
  Watcher.prototype = {
    update: function(){
    //这一步会触发vm的访问器属性，进而将watcher放入观察者的队列（只有在watcher初始化的时候会有放入队列这个	操作，所以在放入时需要做一下判断）
    this.node.nodeValue = this.vm[this.name];
    }
  }
```

Subscripters.target作为订阅者和发布者的一次对话：这段对话在代码中写的很清楚。

（2）发布者定义

发布者无非就是将订阅者加入到对应主题的更新数组之中，然后在主题内容发生更新之后通知对应的更新数组更新自己的内容。

```javascript
  //这个构造方法主要是维护订阅者的队列
  function Subscripters(){
    this.target = null; //初始化时置为空，只有在订阅者初始化的时候才会有值
    this.subs = [];//主题的更新数组
  }

  Subscripters.prototype = {
    //将订阅者添加到主题更新数组中
    addSubs: function(sub){
      this.subs.push(sub)
    },
    //通知更新数组完成更新
    notify: function(){
      this.subs.forEach(sub => {
        sub.update();
      })
    }
  }
```

与此同时，编译方法中的更新步骤就可以由订阅者的初始化完成。

```javascript
//编译以及实现数据绑定和dom监听
  function compile(node, vm){
    var reg = /\{\{(.*)\}\}/;//能够匹配{{test}}这种dom的正则
    //依据节点的类型做不同的处理
    //若节点为一般普通元素
    if(node.nodeType === 1){
      var attr = node.attributes;//得到该元素的所有属性
      for(let i=0;i<attr.length;i++){
        if(attr[i].nodeName === 'v-model'){
          let name = attr[i].nodeValue;
          //添加dom监听，将发生变化的dom的值赋给vue实例
          node.addEventListener('input', e => {
            vm[name] = e.target.value;
          })
//****************以下变化代码****************
          //node.value = vm[name];
          new Watcher(vm, node, name);
//****************以上变化代码****************
          node.removeAttribute('v-model');
        }
      }
    }

    //节点类型是text。也就是{{...}}这种形式的话
    if(node.nodeType === 3){
      if(reg.test(node.nodeValue)){
        var name = RegExp.$1;
        name = name.trim();
//****************以下变化代码****************
        //执行watcher构造函数需要完成的工作：
        //1、获取model的值并赋给该文本节点
        //2、将自己添加到订阅者的队列中
        new Watcher(vm, node, name);
//****************以上变化代码****************
      }
    }
  }
```

究竟什么时候是添加订阅者到更新数组的好时机，又什么时候是通知订阅者更新的好时机？这就要用到访问器属性。

```javascript
//定义访问器属性
  function defProperty(obj, key, val) {
    var subscripters = new Subscripters();
    Object.defineProperty(obj, key, {
      get: function(){
//***************以下新增代码********************
        //将订阅了这个属性的订阅者放入到这个属性对应的队列中，有了这个if判断就可以保证只有在watcher初始化		 的时候才会有加入队列这个动作，保证不会重复加入
        if(Subscripters.target != null){
          subscripters.addSubs(Subscripters.target);
        }
//***************以上新增代码********************
        return val;
      },
      set: function(newVal){
        if(newVal === val) return;
        val = newVal;
//****************以下变化代码****************
        subscripters.notify();//通知更新
//****************以上变化代码****************
      }
    })
  }
```

至此都已完成。详细代码可以去到github上下载查看。若有什么问题可以给我留言，会及时回复。觉得值得鼓励的，给个赞哦。
