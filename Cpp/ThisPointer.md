# this指针的类型

​	今天看到c++primer（5th）上说一个类的指针类型是classtype* const 类型，于是就想自己验证下到底是不是下面是在VS2017上写的demo

```c++
class Cfoo
{
public:
	Cfoo() = default;
	~Cfoo() = default;

	void ShowN(int n)
	{
	this->m_n = n;
	cout << boost::typeindex::type_id_with_cvr<decltype(this)>().pretty_name();  //[1]== Cfoo*
		
    this = nullptr; //[2]===  error C2106: '=': left operand must be l-value
    std::cout << "n : " << m_n << "\n";
	}


	void ShowN(int n) const
	{
	this->m_n;
	cout << type_id_with_cvr<decltype(this)>().pretty_name(); //[3]==const Cfoo *
	std::cout << "n : " << m_n << "\n";
	}
private:
	int m_n = 0;
};


void TestTypecast()
{
	Cfoo foo;
	foo.ShowN(1);

	const Cfoo cfoo;
	cfoo.ShowN(1);
}
```

在vs2017上的测试结果是开始在【1】处得到的类型是 CFoo* ，既然是CFoo * 类型那么久执行【2】的代码试试，但是在编译的时候出现如下的错误：error C2106: '=': left operand must be l-value  从这个错误来说就是this是个右值，为啥会出现以上这种情况呢？

The type of this pointer is either `ClassName *` or `const ClassName *`, depending on whether it is inspected inside a non-const or const method of the class `ClassName`. Pointer `this` is not an lvalue.

```
class ClassName {
  void foo() {
    // here `this` has `ClassName *` type
  }

  void bar() const {
    // here `this` has `const ClassName *` type
  }
};
```

The observation you mentioned above is misleading. Pointer `this` is *not an lvalue*, which means that it cannot possibly have `ClassName * const` type, i.e. it cannot possible have a `const` to the right of the `*`. Non-lvalues of pointer type cannot be const or non-const. There's simply no such concept in C++ language. What you observed must be an internal quirk of the specific compiler. Formally, it is incorrect.

Here are the relevant quotes from the language specification (emphasis mine)

> **9.3.2 The this pointer**
>
> In the body of a non-static (9.3) member function, the keyword this is a prvalue expression whose value is the address of the object for which the function is called. **The type of this in a member function of a class X is X\*. If the member function is declared const, the type of this is const X*, if the member function is declared volatile, the type of this is volatile X*, and if the member function is declared const volatile, the type of this is const volatile X*.** [ Note: thus in a const member function, the object for which the function is called is accessed through a const access path. —end note ]

------

It is worth nothing that back in the C++98/C++03 times several compilers used an internal implementational trick: they interpreted their `this` pointers as constant pointers, e.g. `ClassName *const` in a non-constant method of class `ClassName`. This apparently helped them to ensure non-modifiablity of `this`. GCC and MSVC are known to have used the technique. It was a harmless trick, since at language level `this` was not an lvalue and its constness was undetectable. That extra `const` would generally reveal itself only in diagnostic messages issued by the compiler.

However, with the advent of rvalue references in C++11 it became possible to detect this extra `const` on the type of `this`. For example, the following code is valid in C++11

```
struct S
{
  void foo() { S *&&r = this; }
};
```

Yet it will typically fail to compile in implementations that still use the aforementioned trick. GCC has since abandoned the technique. MSVC++ still uses it (as of VS2017), which prevents the above perfectly valid code from compiling in MSVC++.

上面这些实际上说白了就是标准要求this的类型是 ClassType* 或者是ClassType* const 但是有些编译器根据没有遵守这个规定自己加上了实现。 比如说ClassType* const 就是为了不让改动this指针的指。