
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
         console.log('reject');
         console.log(reason); // 数字太大
         console.log(data); // undefined
      });
 ```
 运行结果：
  当随机数字小于5时：
  ![image](https://github.com/chenhu0920/promise-source-code/blob/master/docs/img/1542804249955.jpg)
  
  
  当随机数字大于等于5时：
  ![image](https://github.com/chenhu0920/promise-source-code/blob/master/docs/img/1542805473904.jpg)
