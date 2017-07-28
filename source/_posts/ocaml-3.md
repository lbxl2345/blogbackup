---
title: Ocaml笔记(二)
tags:
  - Ocaml
  - 静态分析
date: 2017-03-27 22:40:00
---
#### 模块
OCaml把每一段代码，都包装成一个模块。例如两个文件`amodule.ml`和`bmodule.ml`都会定义一个模块，分别为Amodule和Bmodule。
通常模块是一个个编译的，比如

	ocamlopt -c amodule.ml
	ocamlopt -c bmodule.ml
	ocamlopt -o hello amodule.cmx bmodule.cmx
	
那么访问模块中的内容可以使用`open`，也可以使用`module.func`这样的方式。  
通常模块会定义为`struct...end`的形式，这样能够形成一个有效的闭包，防止命名的重复等，它需要和一个`module`关键字绑定。比如：

	module PrioQueue = 
		struct
		...
		end;;



#### 接口、签名
通常模块中的所有定义，都可以从外部进行访问。但实际中，模块只应该提供一系列接口，隐藏一些内容，这也是面向对象语言中所提倡的。模块是定义在`.ml`文件中的，而相应的接口，则是从`.mli`文件中得到的。它包含了一个带有类型的值的表。例如，对于一个模块来说，它的接口可以这样定义：

	(*模块定义*)
	let message = "Hello"
	let hello() = print_endline message
	
	(*接口定义*)
	val hello : uint -> unit

这样，接口的定义就隐藏了`message`。这里，`.mli`文件是在`.ml`文件之前编译的。`.mli`用`ocamlc`来编译，而`.ml`则是用`ocamlopt`来编译的。`.mli`文件就是所说的“签名”。

	ocamlc -c amodule.mli
	ocamlopt -c amodule.ml
	
#### 类型
值可以通过把它们的名字和类型，放到.mli文件的方式来导出。
	
	val hello : unit -> unit

但模块经常定义新的类型。例如，

	type date = { day : int; month : int; year : int }

这里其实有几种.mli文件的写法，例如，包括：

+ 完全忽略类型
+ 把类型定义拷贝到签名
+ 把类型抽象，只给出名字`type date`
+ 把域做成只读的:`type date = private{...}`

如果采用第三种方式，那么模块的用户就只能操作date对象，使用模块提供的函数去间接进行访问。	
#### 子模块
一个给定的模块，可以在文件中显示的定义，成为当前模块的字模块。通过约束一个给定自模块的接口，是能够达到和写一对`.mli/.ml`文件一样的效果的。例如：

	module type Hello_type = sig
 	val hello : unit -> unit
	end
  
	module Hello : Hello_type = struct
  	...
	end
	
#### 仿函数（函子）
OCaml中的仿函数，定义与其他语言不太一样，它是用另一个模块，来参数化的模块。它允许传入一个类型作为参数，但这在OCaml中直接做是不可能的。个人理解，这里的函子和C++中的STL比较类似，它接受不同类型的输入作为初始化。事实上在OCaml中，map和set模块都是要通过函子来使用的。  
例如，标准库定义的`Set`模块，就提供了一个`Make`函子。假如要使用不同类型的集合，可以这样这样利用函子：
	
	# module Int_set = Set.Make (struct type t = int let compare = compare end)
    # module String_set = Set.Make (String);;

至于函子的定义，则是这样：

	module F(X : X_type) = struct
		...
	end

`X`是作为参数被传递的模块，而`X_type`是它的签名，这种写法是强制。  

	module F(X:X_type) : Y_type = 
	struct
		...
	end

这种写法对于返回模块的签名，也能够进行约束。函子的操作也是比较难理解的，多使用set/map，并且阅读这两个库中的源码，是能够帮助理解和记忆的。  

#### 模式匹配
OCaml能够把数据结构分开，并对其做模式匹配。这里举一个例子，表示一个数学表达式`n * (x + y)`，并且分解公因式为`n * x + n * y`  
首先定义一个表达式类型：

	# type expr =
    | Plus of expr * expr        (* means a + b *)
    | Minus of expr * expr       (* means a - b *)
    | Times of expr * expr       (* means a * b *)
    | Divide of expr * expr      (* means a / b *)
    | Value of string            (* "x", "y", "n", etc. *);;
    
那么，对于一个表达式，用模式匹配的方式，可以将其打印成对应的数学表达式：  
	
	# let rec to_string e =
    match e with
    | Plus (left, right) ->
       "(" ^ to_string left ^ " + " ^ to_string right ^ ")"
    | Minus (left, right) ->
       "(" ^ to_string left ^ " - " ^ to_string right ^ ")"
    | Times (left, right) ->
       "(" ^ to_string left ^ " * " ^ to_string right ^ ")"
    | Divide (left, right) ->
       "(" ^ to_string left ^ " / " ^ to_string right ^ ")"
    | Value v -> v;;
    
	val to_string : expr -> string = <fun>
	# let print_expr e =
    	print_endline (to_string e);;
	val print_expr : expr -> unit = <fun>
	
这样，使用print_expr，就能够把一个表达式打印成一个数学表达式。那么，模式匹配的通用形式是：

	match value with
	| pattern    ->  result
	| pattern    ->  result
	  ...
	  
或者对条件进行进一步的约束

	match value with
	| pattern  [ when condition ] ->  result
	| pattern  [ when condition ] ->  result
	  ...
	  
注意，这里还有一种特殊的模式匹配，`| _`，它用来匹配剩下的任意情况。

#### 奇奇怪怪的操作符
OCaml中，还有许多有趣的操作符和表达式。在SO上，我也看到了类似的提问： 
 
	let m = PairsMap.(empty |> add (0,1) "hello" |> add (1,0) "world") 
	
这里有两个问题。第一个，`module.(e)`是啥意思？它其实等价于`let open Module in e`，它相当于一种简写的形式，同样是把module引入当前模块的方式。  
第二个`|>`表达式是什么意思？其实它是`Pervasives`中定义的一个操作符，其定义为`let (|>) x f = f x`。它被称为"reverse application function"（我不知道应该如何翻译），但它的作用，是把连续的调用去有效的串联起来（可以把函数放在参数之后，从而保证一个调用顺序，有一点类似管道的意思）。如果不使用`|>`符号，那么就必须写成：

	let m = PairsMap.(add(1,0) "world" (add(0,1) "hello" empty))
	
在Uroboros当中，还看到有一个奇怪的操作符，那便是`@`，从manual上来看，这个操作符的意思是“串联List”。有这样的例子：

	# List.append [1;2;3] [4;5;6];;
	- : int list = [1; 2; 3; 4; 5; 6]
	# [1;2;3] @ [4;5;6];;
	- : int list = [1; 2; 3; 4; 5; 6]
