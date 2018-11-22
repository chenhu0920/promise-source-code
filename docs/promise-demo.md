before
---
众所周知javascript语言的一大特色就是异步,但是也有一定的问题，异步操作过多的时候，代码内会充斥着众多回调函数，乃至形成回调金字塔，如下<br/>
```
// 传统写法
step1(function (value1) {
  step2(value1, function(value2) {
    step3(value2, function(value3) {
      step4(value3, function(value4) {
        // ...
      });
    });
  });
});
```
NOW
---
为了解决回调函数带来的问题，Promise作为一种更优雅的异步解决方案。Promise 的设计思想是，所有异步任务都返回一个 Promise 实例，所以就不需要上面一次又一层的方法了。Promise 实例有一个then方法，用来指定下一步的回调函数.如下<br/>
```
(new Promise(step1))
  .then(step2)
  .then(step3)
  .then(step4)
```
下面是promise的一个例子：<br/>
```
function getNumber(){
   var p=new Promise(function(resolve, reject){
      setTimeout(function(){
          var num=Math.ceil(Math.random()*10); // 生成1-10 之间的随机数 Math.ceil(): 大于或等于给定数字的最小整数
          if(num < 5){
            resolve(num);
           }else{
             reject('数字太大了')
            }
        },2000);
    });
   return p;
}

getNumber()
  .then(function (data) {
       console.log('resolved');
       console.log(data);
    },function(reason, data){
         console.log('resolved');
         console.log(reason); // 数字太大
         console.log(data); // undefined
      });
 ```
 运行结果：
  当随机数字小于5时：<br/>
      resolved<br/>
      4
  
  当随机数字大于等于5时：<br/>
     resolved<br/>
     数字太大了<br/>
     undefined<br/>

这是一个Promise的例子，Promise是一个构造函数，实例化之后，


