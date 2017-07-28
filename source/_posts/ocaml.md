---
title: Ocaml笔记(一)
tags:
  - Ocaml
  - 静态分析
date: 2017-03-27 18:40:00
---
### Ocaml
网上关于Ocaml的资料比较少，可见它是一门偏小众的语言。不过在DSL和程序分析方面，Ocaml是十分强大的。由于目前研究需要使用的Uroboros(<https://github.com/s3team/uroboros>)是由Ocaml来编写的，因此我也打算一探究竟，学一学这门语言。
#### 函数的定义和调用
Ocaml中，定义函数的语法很简单。这个函数是输入两个浮点数后计算它们的平均值。

	let average a b =
		(a +. b) /. 2.0;;
		
在C当中，如果要定义一个相同的函数，其定义是这样的：

	double average(double a, double b){
	return (a + b) / 2;
	}
	
可以看到，OCaml没有定义a和b的类型，而且也没有所谓的`return`，而且写的是`2.0`，没有用隐式转换。这其实是由Ocaml语言的特性决定的：

+ Ocaml是强静态类型的语言  
+ OCaml使用类型推倒，不需要注明类型
+ OCaml不做任何隐式转换(所以要写浮点数就必须是2.0)
+ OCaml不允许重载，`+.`表示两个浮点数相加，也就是说操作符是和类型相关的
+ OCaml的返回值是最后的表达式，不需要`return`

和大多数基于C的语言不同，OCaml的函数调用，是没有括号的。例如定义了一个函数`repeated`，它的参数是一个字符串`s`和一个数`n`，那么它的调用形式会是：
	
	repeated "hello" 3	(*this is Ocaml code*)

可以看到既没有括号，也没有都好。不过`reoeated ("hello", 3)`也是合法的，只不过它的参数是一个含两个元素的对(pair)。
#### 基本类型、转换与推倒
类型 | 范围
---|---
int|32bit:31位;64bit:63位
float|双精度，类似于C中的double
bool|true/flase
char|8bit字符
string|字符串
unit|写作()，类似void

这里，Ocaml内部使用了int中的一位来自动管理内存(垃圾收集)，因此会少一位。
前面也提到，OCaml是没有隐式类型转换的。因此

	1 + 2.5;;
	1 +. 2.5;;

在OCaml中是会报错的。但是如果一定要让一个整数和浮点数相加，就必须显示的进行转换，比如：

	float_of_int i +. f;;
	float i +. f;;

许多情况下，不需要声明函数的变量和类型，因为Ocaml自己会知道，它会一直检查所有的类型匹配。比如前面的average函数，就能够自给判断出这个函数需要两个浮点数参数和返回一个浮点数。  
	
#### 函数的递归和类型
和基于C的语言不同之处在于，OCaml中，函数一般不是递归的，除非用let rec代替let定义递归函数。这是一个递归函数的例子：

	let rec range a b =
		if a > b then []
		else a :: range (a+1) b
		
let和let rec的唯一区别，就是函数的定义域。举个例子，如果用`let`定义`range`，那么`range`会去找一个已经定义好的函数，而不是它自身。不过在性能上，`let`和`let rec`并没有太大的差异。所以即使全部用`let rec`来定义也可以。 
而OCaml的类型推倒，也使得几乎不用显式的写出函数的类型。不过Ocaml经常以这样的实行显示参数和返回值的类型：

	f:arg1 -> arg2 -> ... -> argn -> rettype
	f: 'a->int	(*单引号表示人意类型*)
	
#### 表达式
在Ocaml当中，局部变量/全局变量其实都是一个表达式。例如，局部表达式有：

	let average a b =
		let sum = a +. b in
		sum /. 2.0;;
		
标准短语`let name = expression in`是用来定义一个命名的局部表达式的。`name`在这个函数当中，就可以代替`expression`，直到一个`;;`结束这个代码块。这里把`let ... in`视为一个整体。和C中不一样，OCaml中`name`只是`expression`的一个别名，我们是不能给`name`赋值或者改值的。  
而全局表达式，也可以像定义局部变量一样定义全局名，但这些也不是真正的变量，而只是缩略名。  

	let html =
  	let content = read_whole_file file in
	  GHtml.html_from_string content
	  ;;
	
	let menu_bold () =
	  match bold_button#active with
	  | true -> html#set_font_style ~enable:[`BOLD] ()
	  | false -> html#set_font_style ~disable:[`BOLD] ()
	  ;;
	
	let main () =
	  (* code omitted *)
	  factory#add_item "Cut" ~key:_X ~callback: html#cut
	  ;;
	  
这里，`html`实际上是一个“小部件”，没有指针去保存它的地址，也不能赋值，而是在之后的两个函数中被引用。  
#### Let-绑定
绑定，`let ...`，能够在OCaml中，实现真正的变量。在OCaml中，引用使用关键字`ref`来进行定义。例如，  

	let my_ref = ref 0;;(*引用保存着一个整数0*)
	myref := 100(*引用被赋值为100*)
	
`:=`用来给引用赋值，而`！`用来取出引用的值。以下是一个C和OCaml的比较   

	OCaml                   C/C++

	let my_ref = ref 0;;    int a = 0; int *my_ptr = &a;
	my_ref := 100;;         *my_ptr = 100;
	!my_ref                 *my_ptr
	
#### 嵌套函数
与C语言不同的是，OCaml是可以使用嵌套函数的。  

	  let read_whole_channel chan =
	  let buf = Buffer.create 4096 in
	  let rec loop () =
	    let newline = input_line chan in
	    Buffer.add_string buf newline;
	    Buffer.add_char buf '\n';
	    loop ()
	  in
	  try
	    loop ()
	  with
	    End_of_file -> Buffer.contents buf;;
这里，`loop`是只有一个嵌套函数，在`read_whole_channel`中，是可以调用`loop()`的，但它在`read_whole_channel`当中并没有定义，嵌套函数可以使用主函数当中的变量，它的格式和局部命名表达式是一致的。
#### 模块和OPEN
OCaml也提供了很多模块，包括画图、数据结构、处理大数等等。这些库位于`usr/lib/ocaml/VERSION`。例如一个简单的模块`Graphics`，如果想使用其中的函数，有两种方法。第一种是在开头声明`open Graphics`，第二种是在函数调用之前加上前缀，例如`Graphics.open_graph`。  
如果想用`Graphics`当中的符号，也可以通过重命名的方式，简化前缀。  
	
	module Gr = Graphics;;
	
这个技巧在模块嵌套时十分有用。

#### `;;`还是`;`，或者什么都不用？
在OCaml中，有时候会使用`;;`，有时候会使用`;`，有时候却什么都不用，这就让初学者很容易迷惑。这里，OCaml实际上定义了一系列的规则。  
 #1 必须使用`;;`在代码的最顶端，来分隔不同语句(不同代码段之间的分隔)，并且**不要**在函数定义或其他语句中使用  
 #2 可以在某些时候省略掉`;;`，包括`let`，`open`，`type`之前，文件的最后，以及OCaml能自动判断的地方  
 #3 `let ... in`是一条单独道语句，不能在后面加单独的`;`  
 #4 所有代码块中其他语句后面，跟上一个单独的`;`，最后一个例外  
 
 看到这些规则，我依然没有完全理解这三者的用法。我想，只有实际接触过Ocaml代码，才能逐渐体会到其中的精髓吧。

