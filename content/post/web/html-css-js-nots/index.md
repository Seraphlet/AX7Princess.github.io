---
description: ""
title: "前端学习笔记"
draft: true
date: "2026-07-05T11:59:38+08:00"
slug: "html-css-js-nots"
categories:
 - notes
tags:
 - html
 - css
 - js
image: ""
---

# Html5笔记

## 一、基础框架
1. 文件以<DOCTYPE html开头>
基本结构为
```
<!DOCTYPE html> 
<html> 
<head> 
<meta charset="utf-8" /> 
<title></title> 
</head> 
<body> 
</body> 
</html> 
```
2. 标签是最基本单位，由于<>组成，标签分为单标签和双标签，即是否有结束标记，```<title></title>```<br/>
```<hr/>```单标记

3. 常用标签有
```
<html>定义html文档
<head>定义文档头
<a>链接标签
<div>html层
<form>表单
<body>文档体
<tittle>标题
<img>图片
<table>表格
<td>表格列
<tr>表格行
<input>输入
```
<br/>
3. 标签属性

属性是标签一部分，用于包含额外信息，属性和属性值成对出现
```
<img src=“../image/a.png” width=“100” height=“100”/> 
<!-- 结构是 属性名=”属性值” -->
```
4. ```<head>```用于标识网页元数据，描述网页基本信息，主要包括```<title> <meta>```
```
<meta charset="UTF-8"> 
```
用于设置字符编码

name属性，将信息写给搜索引擎看，常用属性值有，author，keywords，description，
```<meta name="viewport" ...>```
用于适配手机，否则手机看到字很小

5. ```<link>```标签 加载一个图片作为网页图标
re属性：声明被链接文件和当前文件关系，这里选icon
type属性：声明类型
href属性：标识图片地址
```
<link rel="icon" type="image/x-icon" href="img/icon.jpg" />
```
### 二、常用块标签
1. 
```<h1></h1>...<h6></h6>标题
<hr/>水平线
<p></p>段落标签
<br/>换行
<pre></pre>预览格式标签，按原格式输出
<ul><li></li></ul>无序列表标签
<ol><li></li></ol>有序列表标签 
<div></div>分区标签 ，将文档分割为独立的、不同的部分
```

## 三、表单
表单用于用户填写数据源并交给服务器出来，像qq登录页就是表单。

**表单由```<form></form>```包裹
1. 常见属性有
action用于指定给谁提交数据，即发送服务器的地址
method用于像服务器提交数据的方法，get或post

2.input输入框属性
type 默认是text
name 标识输入框名称，一般必填，传递数据时使用```name=value```的形式传递
value 输入框默认值
placeholder 提示信息，当有值时就会消失，浅灰色的提示文字
checked="cheked" 默认选中

**其他type**
-  text 文本框，默认宽度为20个字符
- password 密码输入框
- radio 单选按钮
```
<form action="form.php" method="post"> 
<input type="radio" name="sex" value="男" checked="checked" />男 
<input type="radio" name="sex" value="女" />女 
</form>
```
- checkbox复选框

-  file 上传文件按钮
- submit提交按钮，当没有value值时，submit 按钮中默认显示的文字为“提交”
- reset 重置表单为初始状态
- button 单机按钮
    ```
      <form action="form.php" method="post"> 
      <input type="button" value="点我！"  onclick="alert('这是一个按钮！')" /> 
      </form>
     ```
3. select 下拉选择
```
<form action="form.php" method="post"> 
 <select name="week"> 
  <option value="1">1</option> 
  <option value="2">2</option> 
  <option value="3">3</option> 
  <option value="4">4</option> 
  <option value="5">5</option> 
  <option value="6">6</option> 
  <option value="7">7</option> 
 </select> 
</form> 
```
4. textarea 文本域
readonly="readonly" 只读

5. button按钮


# CSS基础
css由选择器及一条或多条声明，基本语法如下：
```
选择器{ 
属性:属性值;     
[属性:属性值; …] 
} 
```
CSS添加注释的方式为/*……*/

行内样式表，即将css代码写在HTML代码内部，作为标签的属性
```
<a href="#" style="color:red;font-size:10px;">日用百货</a>
```
内嵌式则是将css代码内嵌到HTML代码中，通常放在head标签内部
```
<head> 
<!--charset="UTF-8"表示当前文档采用字符集中utf-8,支持中文--> 
<meta charset="UTF-8"> 
<title>内部样式表</title> 
<!--内部样式表 代码要放置在style标签内--> 
<style type="text/css"> 
div{ 
color:red; 
} 
</style> 
</head>
```

外联式也是最推荐的，方便维护，各部分代码分离
```
<head> 
<meta charset="UTF-8"> 
<title>外部样式表</title> 
<link rel="stylesheet" type="text/css" href="css/ch05.css" />  
</head>
```
## 一、CSS选择器

选择器是指css选择要修饰的元素，对指定元素进行修饰美化。

1. 通用选择器
*{}

作用：选中页面中的的所有标签，一般用于定义最通用的属性，设置默认值，优先级最低，低于所有选择器。

2. 标签选择器

HTML标签名{样式属性:样式属性值;...}

作用：选中页面所有的对应标签，对某类标签进行同意样式设置，高于通用选择器。

3. 类选择器

.类名称{}

作用：在需要改变样式的标签上使用class="选择器名称"调用对应选择器，优先级高于标签选择器。

4. id选择器

#id 名称{}

作用：在需要改变的样式标签上，使用id="选择器名称"调用对应选择器，优先级大于类选择器。

### 后代选择器与子代选择器

1. 后代选择器

选择器1 选择器2 选择器3……{} ，每个选择器之间用空格分隔。 
```
div .li{ 
color: yellow; 
} 
```
div.li{}表示选中的元素包括 div 里面的 class="li"的元素，其中 class="li"的元素可以是div 的子代，也可以是div的后代，也就是孙代及往后。 

2. 子代选择器 

选择器1>选择器2>选择器3……{} ，每个选择器之间用大于号分隔。 

```
div>ul{ 
color: blue; 
} 
```
div>ul{}表示 ul 必须是div的直接子代，孙代以后不选中。

### 交集选择器与并集选择器

1. 交集选择器

选择器1选择器2……{} ，选择器之间没有分隔符。 

```
.list#li{ 
color: red; 
}
```

.list#li{} 元素必须同时具备 class="list"并且 id="li"样式才能生效。

2. 并集选择器

选择器1,选择器2,……{} ，选择器之间用逗号分隔。

```
.li,#li{ 
color: red; 
}
```

.li,#li{} 元素只要具备 class="li"或者 id="li"，样式即可生效。

### 伪类选择器

选择器名称:伪类状态{}

常见的伪类状态：

link：未访问状态。 
visited：已访问状态。 
hover：鼠标指向时，即悬停在元素上方时。 
active：激活选定状态（鼠标点下去没松开）。 
focus：获得焦点时（input常用）。 
超链接多种伪类共存时的顺序如下：link→visited→hover→active

## CSS常见属性

### 字体字号颜色

1. 字体

- font-family 字体族，设置字体，可以同时设置多个字体，多个字体样式间用逗号分隔，浏览器解析时，会从左往右依次查找。

- font-style 社会之字体样式，通常只有normal正常和italic斜体

- font 缩写形式，依次为，为font-style、font-weight、font-size/line-height、font-family，使用时必须按顺序，多个样式间用空格分隔，font-size 和 font-family 必须指定，其他样式不指定将采用默认样式显示。

```
font:italic  bold  75%/1.8      'Microsoft Yahei', sans-serif;
```

2. 字号

- font-weight 设置字体粗细，lold加粗，lighter细体，或100-900数字，400正常，700加粗

- font-size 设置大小，值为..px或..%,浏览器默认16px

3. 颜色

- color 设置颜色，直接用英文颜色单词，或16禁止写法，或rgb写法

- opacity 设置透明度，值为0~1

### 文本属性

1. line-height 行高

# Javascript

## 一、基础

js使用也是三种方式，内嵌式，页面中使用和引入外部文件
内嵌式
```
<button onclick="JavaScript:alert('Hello JavaScript！')">点我</button> 
```
在页面中使用
```
<script type="text/javascript"> 
// JavaScript 代码 
</script>

```
外部引入
```
<script language="JavaScript" src=" JavaScript 文件路径"></script>   
```

### 1. 变量

js共有6种基本数据类型，不需要声明数据类型，同意使用var关键字声明，不适用var则默认为全局变量。

声明基本规范，变量名只能由字母、数字、下画线、$ 组成，且开头不能是数字。
变量区分大小写

空引用，Null式一种特殊数据类型，同时作为关键字不区分大小写，NULL，Null，null均是合法的

**布尔类型**
**Number**数值，不区分整数小数类型，统一用数值代替
**String**字符串
**Object** 对象类型
### 2. 数据类型转换

- str转换为num，使用形如，Number("111");
- isNaN，监测变量是否是NaN，函数会尝试使用Number()函数进行转换，如果能转换为数字则不是非数值，否则为false
- 将字符串转换为整型，用parseInt，如parseInt(""),parseInt 函数只能转换 String 类型，对 Boolean、null、Undefined 进行转换结果均为NaN。
- parseFloat 将字符串转换为浮点型，：parseFloat 同样也是只能转换 String 类型，对 Boolean、null、Undefined 进行转换结果均为NaN。
- typeof，变量类型检测

### 3. Js中的输入输出

document.write 文档中打印输出

- 用来网页向文档中输出内容，方法将()内的内容打印在浏览器中，除变量或常量意外的任何内容，打印时必须放在""中，变量和常量放在""以外。
- 拼接字符串用+里连接
- alert 浏览器弹窗输出，他会弹出一个带有"确定"的按钮。浏览器的alert(),弹窗弹窗警告，()中的语句与doucment.write()括号中的的相同。如

alert("弹窗警告内容"); 

- prompt 浏览器弹窗输入

用于提示用户在进入页面前输入某个值，用户需要点击确定或者取消，点击确定则返回输入的值，带年纪取消则返回null。还可以定义变量来接收输入内容，例如var name = prompt("请输入您的名字：")

- confirm 浏览器弹窗确认

会问用户一个“是/否”的问题，用户可以点“确定”或“取消”，然后返回 true 或 false。
```     
var is = confirm("在吗？？？"); 
if(is){ 
alert("在"); 
}else{ 
alert("不在"); 
} 
```
- console.log 浏览器控制台输出
    
### 4. 运算符
- 算数运算
    有 +（加）、-（减）、*（乘）、/（除）、%（取余）、++（自增）、--（自减）

- 赋值运算 =
    
- 复合赋值运算符有+=、-=、*=、/=、%=五种，基本语法如下： a += 5;  等价于 a = a + 5;
 -  关系运算与逻辑运算 
    关系运算符有七种：==（等于）、===（严格等于）、!=（不等于）、>（大于）、<（小于）、>=（大于等于）、<=（小于等于）。 
              
        ===：严格等于，左右两边的数据类型不同时，返回 false；类型相同时，再进行下一步值的判断。 
        ==：等于，左右两边的数据类型相同时，再进行下一步值的判断；类型不同时，尝试将等式两边转为数值类型，再进行判断。
         
- 逻辑运算符有三种：&&（与）、||（或）、!（非）。

## 二、流程控制

### 1. if-else

if结构

```
if(判断条件){ 
   // if条件成立时，代码执行语句块 
}
```
if-else结构
```
if(判断条件){ 
 //条件为true时执行的代码块 
}else{ 
 //条件为false时执行的代码块 
} 
```
### 2. switch-case结构

基本结构如下
```
switch(表达式){ 
 case 常量表达式1: 
  语句1; 
  break; 
 case 常量表达式2: 
  语句2; 
  break; 
  …… 
 default: 
  语句N 
  break; 
} 
```
### 3. whhile循环
- while
```
while (条件){ 
需要执行的代码块 
}
```
- do-while 
```
do{ 
需要执行的代码块 
}while (条件); 
```
### 4.for循环

```
for (初始化循环变量; 循环条件; 修改循环变量的值){ 
需要执行的代码块 
}
```
- for-in循环

for-in循环，这种结构主要用于遍历对象的属性

```
var arr = [1,2,3,4,5]; // 声明一个数组 
for(item in arr){  
 // 读出数组中的每一个值(数组有几个值，for-in便循环几次) 
 console.log(item); 
}
```
### 5. 流程控制语句

- break语句
跳出本次循环，执行循环后面代码

- continue语句 
跳过本次循环进入下一轮循环

- return 
只能用于函数中，直接终止当前函数

## 三、函数

### 1. 有参函数

```
function 函数名(参数1,参数2,……){ 
// 函数体 
return 结果; 
}
```
### 2.无参函数
```
function 函数名(){ 
// 函数体 
return 结果; 
} 
```
### 3. 匿名函数


事件调用匿名函数
```
// window.onload 表示当文档加载完成后，自动调用匿名函数 
window.onload=function(){ 
alert("杰瑞教育"); 
} 
```

### 4. 自执行函数

自执行函数是指无须手动调用，在声明函数时自动调用的函数类型，有三种声明方式如下：

- !function(形参列表){}(实参列表);  
可以使用任意运算符号开头，但一般使用英文叹号“!”，在函数{}后面紧跟一个圆括号，表示自动调用当前匿名函数，函数的实参可以通过最后的圆括号传入。 
```
!function(num1,num2){ 
var sum = num1 + num2; 
}(1,2);
```
- (function(形参列表){})(实参列表);   

```
(function(num1,num2){
    var sum = num1 + num2; 
})(1,2); 
```
- (function(形参列表){})(实参列表);   
```
(function(num1,num2){
    var sum = num1 + num2; 
})(1,2); 
```
## 四、BOM和DOM

### 1.window对象

浏览器对象模型（Browser Object Model，BOM）使 JavaScript 有能力与浏览器“对话”。由于现代浏览器几乎实现了 JavaScript 交互性方面的相同方法和属性，所以常被认为BOM 的方法和属性。BOM 由多个对象组成，其中代表浏览器窗口的 window 对象是 BOM的顶层对象，其他对象都是该对象的子对象

所有浏览器都支持 window 对象。window 对象表示浏览器窗口。所有 JavaScript 全局对象、函数、变量均自动成为window对象的成员。 全局变量是 window 对象的属性。全局函数是 window 对象的方法。

**window对象的常用方法**

1. window弹窗输入输出

  - promt()
  - alert()
  - confirm()
2. open()与close()

  - open重新打开一个窗口，主要传入三个参数URL地址、窗口名称、矿口特征
```
function openWindow(){ 
window.open("http://www.baidu.com"," 百度一下","height=700px,width=1000px, top=200px, 
left=300px"); 
} 
<button onclick="openWindow()">打开一个新的浏览器窗口</button> 
```
3. close()关闭浏览器当前选项卡
```
function closeWindow(){ 
window.close(); 
} 
<button onclick="closeWindow()">关闭当前窗口</button>
```

4.  setTimeout()与 clearTimeout()

   - setTimeout()：设置延时执行（以ms为单位计时），只会执行一次。
   - clearTimeout()：清除延时，传入参数：调用 setTimeout 时返回一个 id，通过变量接收id，传入clearTimeout。

5.  setInterval()与 clearInterval() 
   - setInterval()：设置定时器，循环每个 N 毫秒执行一次，两个参数：需要执行的function / 毫秒数。 
   - clearInterval()：清除定时器，传入参数：调用 setInterval 时返回一个 id，通过变量接收id，传入clearInterval。

6. screen 屏幕对象

screen 对象包含有关客户端显示屏幕的信息，有四个常用属性，分别是屏幕宽度、屏幕高度、可用宽度和可用高度。

```
screen.width;    
//屏幕宽度 
screen.height;    
//屏幕高度 
screen.availWidth;  
//屏幕可用宽度 
screen.availHeight;  
//屏幕高度 = 屏幕高度-底部工具栏
```
7.  location 地址栏对象

location 对象包含有关当前 URL 的信息,。它存储在 window 对象的 location 属性中，表示那个窗口中当前显示的文档的 Web 地址，可通过 window.location 属性来访问,location作为window对象属性时，可以设置页面跳转
```
window.location = "http://www.baidu.com"; 
```

- URL完整路径结构
```
完整的URL路径： 
协议名://主机名(IP地址):端口号/文件路径?传递参数(name1=value1&name2=value2)#锚点 
例如： 
http://127.0.0.1:8080/wenjianjia/index.html?name=jredu#top
```
location对象常用属性
```
console.log(location);   //取到浏览器的完整URL信息 
location.href;      //返回当前完整路径   
location.protocol;      //返回协议名  http:// 
location.host;      //返回主机名+端口号    127.0.0.1:8080 
location.hostname;      //返回主机名   127.0.0.1 
location.port;      //返回端口号        :8080 
location.pathname;     //返回文件路径      
location.search;     //返回?开头的参数列表 
location.hash;      //返回#开头的锚点 
```
除了 URL 属性外，location对象还具有三个常用方法。location 对象的 reload() 方法可以重新装载当前文档；replace()方法可以装载一个新文档而无须为它创建一个新的历史记录，即在浏览器的历史列表中，新文档将替换当前文档；assign()方法可以加载新的文档.

8.  history历史记录对象

9. navigator浏览器配置对象 

### 2. Core DOM

1. DOM树结构分析

DOM 节点分为三大类：元素节点、文本节点、属性节点。其中，元素节点又叫标签节点，指文档中的各种 HTML 标签；文本节点和属性节点为元素节点的两个子节点，分别表示标签中的文字和标签的属性。通过 getElement 系列方法，可以取到元素节点。

2. 操作元素节点

在DOM操作中，操作元素节点是最基础的一步，使用HTML操作任何内容都需要选中一个标签，才能对标签以及标签的文字、属性、样式进行修改。

-  getElementById

getElementById() 方法可返回对拥有指定 id 的第一个对象的引用。如果需要查找文档中一个特定的元素，最有效的方法是 getElementById()。在操作文档中一个特定的元素时，最好给该元素一个 id 属性，为它指定一个（在文档中）唯一的名称，然后就可以用此 id 查找想要的元素。
```
<div id="box" >div 文字</div> 
var divById = getElementById("box"); 
```
*通过id获取唯一的节点，如果存在多个同名id，则只会选取第一个。*

- getElementsByName / getElementsTagName /getElementsClassName 

通过Name、TagName、ClassName 取到一个数组，包含多个节点。 它们的使用方式是，通过循环取到每一个节点。而循环的次数是从0开始，到数组的最大长度后结束。

```
<div name="div1" class="div2">div 文字</div> 
var divByName = getElementsByName("div1");  
var divByTagName = getElementsByTagName("div"); 
var divByClassName = getElementsByClassName("div2"); 
```
- document.querySelector / querySelectorAll 

querySelector() 方法仅仅返回匹配指定 CSS 选择器的第一个元素。如果需要返回所有的元素，则使用 querySelectorAll() 方法替代。 

```
document.querySelector("#demo"); 
```

3. 操作文本节点

操作元素节点取到标签节点后，接下来就可以通过 JavaScript 中提供
的方法对标签中的文字进行获取、修改操作。

- tagName

tagName 属性返回元素的标签名，即获取节点的标签名

```
<div id="div"></div> 
<button id="btn">单击</button> 
<script type="text/javascript"> 
var btn = document.getElementById("btn"); 
var div = document.getElementById("div");
btn.onclick = function(){  /* 单击按钮弹出对话框 */ 
alert(div.tagName);     
} 
</script>
```
-  innerHTML 
innerHTML 属性设置或返回表格行的开始和结束标签之间的 HTML，即设置或者获取节点内部的所有HTML。 

```
<div id="box"></div> 
<button id="btn">单击</button> 
<script type="text/javascript"> 
var btn = document.getElementById("btn"); 
var div = document.getElementById("box"); 
btn.onclick = function(){ 
div.innerHTML = "<h1>杰瑞教育</h1>" 
alert(div.innerHTML);    
} 
</script>
```
-  innerText 

innerText 属性用来定义对象所要输出的文本，即它可以用来设置或者获取节点内部的所有文字。

*innerHTML 指的是从对象的起始位置到终止位置的全部内容，包括HTML标签。*
*innerText 指的是从起始位置到终止位置的内容，但它去除HTML标签。*

4. 操作属性节点 

在 DOM 操作中，操作元素节点取到标签节点后，接下来就可以通过 JavaScript 中提供的方法对标签的属性进行获取、修改操作。

-  查看属性节点 

getAttribute() 方法返回指定属性名的属性值，如果以 attr 对象返回属性，则需要使用getAttributeNode。
```
getAttribute("属性名"); 
```

- 设置属性节点

setAttribute() 方法添加指定的属性，并为其赋指定的值。如果这个指定的属性已存在，则仅设置或更改属性值。

```
setAttribute("属性名","属性值"); 

例：

<button onclick="getAttr()">取到 a 的 href 属性值</button> 
<button onclick="setAttr()">修改 a 的 href 属性值</button> 
<a href="http://www.baidu.com" id="a1"/>这是一个超链接</a>
function getAttr(){ 
var a1 = document.getElementById("a1"); 
alert(a1.getAttribute("href")); 
} 
function setAttr(){ 
var a1 = document.getElementById("a1"); 
a1.setAttribute("href","http://www.jredu100.com"); 
console.log(a1.href); 
}
```

5.  JavaScript 修改元素样式

-  className

使用 className 直接设置 class 类，为元素设置一个新的 class 名字,注意写法是className。

```
div.className = "cls1"; 
```

- style

使用 style 设置单个属性，为元素设置一个新的样式，注意属性名要使用小驼峰命名法则。

```
div.style.backgroundColor = "red";
```

-  style.cssText 

style.cssText 为元素同时修改多个样式。使用 style 或 style.cssText 可以设置多个样式属性。

```
div.style = "background-color:red;color:yellow;" 
div.style.cssText = "background-color:red;color:yellow;" 
```












