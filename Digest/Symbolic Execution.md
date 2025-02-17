# Intro
现在有这样一段代码
```c
void foo(int x, int y){
	int t = 0;
	if(x>y){
		t=x;
	}else{
		t=y;
	}
	if(t<x){ 
		assert false;
	}
}
```
那么在执行到`if(t<x)`时t存在两种可能

$$
(x>y) \Rightarrow t = x
$$

或

$$
(x\le y) \Rightarrow t = y
$$

由此可得无论输入怎样的$x$或$y$都无法通过断言。得到这个结论的过程在人脑中或许比较麻烦，因为我们要判断不同分支可能演化出的结果。而且示例代码是很短的，单靠思考显然无法对真实世界的海量代码进行验证。假如我们能通过代码衍生出符号系统来对代码进行验证，那意味着我们获得了对海量代码从数学上进行正确性验证的能力。
# SMT(satisfiability modulo theory)


![[Excalidraw/Drawing 2025-02-12 16.29.11.excalidraw]]


SMT求解器接收一个逻辑表达式，通过检查是否满足约束条件来决定输出哪些结果。

![[Excalidraw/Drawing 2025-02-12 17.07.09.excalidraw]]

SMT由两部分组成，SAT和Theory Solver。
## SAT
SAT负责解决布尔可满足性问题， 也就是判断一个仅包含布尔变量的命题逻辑式是否可满足。如果Theory Solver无法解时，SAT求解器会调整布尔解，并重新求解，直到找到一个可行解或证明无解。
## Theory Solver
Theory Solver负责处理特定的数学理论，比如有
* 算术求解器处理整数、实数、浮点数计算
- 数组求解器处理数组索引和更新（例如 `A[i] = 10`）
- 字符串求解器处理字符串匹配、连接、子串等操作（例如 `str.contains("abc")`）
- 位向量求解器处理低级位运算（例如 `x & y = 0b1100`）
Theory Solver会返回逻辑子式的结果，无法处理的也会告知原因。
---

比如有一个表达式`x > 5 and y < 5 and (y > x or y > 2)`
我们可以简单的把它看成是一组布尔表达式的合式即`F1 and F2 and (F3 or F4)`，现在问题变成一个纯布尔问题，我们的目标也变成了“能否找到一组满足这个合式的值？”
那SAT就把它分解成`F1∧F2∧F3`和`F1∧F2∧F4`两个式子，也就是`x>5,y<5,y>x`和`x>5,y<5,y>2`并输出给Theory Solver求解显然后一个式子可以成立。

