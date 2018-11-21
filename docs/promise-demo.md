
function getNumber(){
   var p=new Promise(function(resolve,reject){
      setTimeout(function(){
          var num=Math.ceil(Math.random()*10); // 生成1-10 之间的随机数 Math.ceil(): 大于或等于给定数字的最小整数
          if(num<=5){
            resolve(num);
           }else{
             reject('数字太大了')
            }
        },2000);
    });
   return p;
}

getNumber()
  .then(function(data){
       console.log('resolved');
       console.log(data);
    },function(reason,data){
          console.log('resolved');
       console.log(reason); // 数字太大
        console.log(data); // undefined
      });