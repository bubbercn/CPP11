# 深入浅出——C++11中的移动语义(move semantics)

## 背景和问题

### RAII

RAII是 *Resource acquisition is initialization* 的简称，是面向对象编程中常用的一种模式。总结起来，RAII包括：

* 把资源的使用和维护封装在类( *class* )中
  * 在构造函数中获得资源并且初始化维护资源需要用到的辅助结构。如果获得资源失败，则抛出异常( *exception* )。
  * 通过析构函数来释放资源。
* 使用资源时，通过类的接口来获得资源。

可以看出RAII的主要思想就是把程序中用到的资源的生命周期跟对象的生命周期绑定起来，利用编程语言的特性来防止资源泄漏。因此，RAII也称为 *Scope-Bound Resource Management* 。

### 一个C++的例子

	#include <iostream>
	using namespace std;
	
	class MyString
	{
	public:
		MyString(const char* string = nullptr)
		{
			if (string != nullptr)
			{
				size_t length = strlen(string);
				m_data = new char[length + 1];
				strcpy(m_data, string);
				cout << "memory allocated" << endl;
			}
		}
		~MyString()
		{
			delete[] m_data;
			cout << "memory released" << endl;
		}
		const char* c_str()
		{
			return m_data;
		}
	private:
		char* m_data = nullptr;
	};

	int main()
	{
		MyString string = "Hello, RAII!";
		cout << string.c_str() << endl;
		return 0;
	}

上面是一个非常简单的一个RAII的例子。我们把字符串的内存申请和释放封装在了MyString类的构造函数和析构函数中。在main函数中，我们创建了一个MyString的实例并将它打印出来。当我们运行这个程序时，我们会得到如下输出:

> memory allocated  
Hello, RAII!  
memory released

可以看到，MyString类的使用者不需要去担心其背后内存的申请和释放。

### 浅拷贝和深拷贝

现在让我们在main函数里面加一些字符串拷贝的操作——构建两个字符串stringA和stringB，stringB是stringA的拷贝：

	int main()
	{
		MyString stringA = "Hello, RAII!";
		MyString stringB = stringA;
		cout << stringB.c_str() << endl;
		return 0;
	}

这段程序在我的MacBook上用Xcode编译运行时的结果如下：

> memory allocated  
Hello, RAII!  
memory released  
Test(27525,0x1000ae5c0) malloc: *** error for object 0x10070c690: pointer being freed was not allocated  
Test(27525,0x1000ae5c0) malloc: *** set a breakpoint in malloc_error_break to debug

可以看到，我们遇到了一个运行时的错误。为了理解这里发生了什么，不得不提一下 ***拷贝构造函数*** 这个概念。对于任何类来说，如果我们不定义拷贝构造函数，编译器会自动帮我们生成一个默认的拷贝构造函数。在这里，这个拷贝构造函数，只是简单的值拷贝。因此，拷贝的结果是stringA和stringB有一样的m_data，它们指向同一个字符串。对于这类拷贝——只拷贝了指针的值，而并没有拷贝指针指向的内容，我们称之为 ***浅拷贝*** 。显然，浅拷贝并不是我们想要的结果，而且它导致了stringA和srtingB在析构的时候去释放同一片内存，这在程序运行时是可能导致程序崩溃的。  

为了解决这个问题，我们需要定义自己的拷贝构造函数来实现我们需要的 ***深拷贝*** 。一般来说，我们也需要同时定义对应的 ***拷贝赋值函数*** (拷贝构造函数和拷贝赋值函数并称C++的 ***拷贝语义***)：

	class MyString
	{
	public:
		MyString(const char* string = nullptr)
		{
			init(string, "constructor");
		}
		MyString(const MyString& myString)
		{
			init(myString.m_data, "copy constructor");
		}
		MyString& operator=(const MyString& myString)
		{
			if (this == &myString)
				return *this;

			cleanUp("assignment operator");
			init(myString.m_data, "assignment operator");
			return *this;
		}
		~MyString()
		{
			cleanUp("destructor");
		}
		const char* c_str()
		{
			return m_data;
		}
	private:
		void init(const char* string, const char* where)
		{
			if (string != nullptr)
			{
				size_t length = strlen(string);
				m_data = new char[++length];
				strcpy(m_data, string);
				cout << "memory allocated in " << where << endl;
			}
		}
		void cleanUp(const char* where)
		{
			if (m_data)
			{
				delete[] m_data;
				m_data == nullptr;
				cout << "memory released in " << where << endl;
			}
		}
		char* m_data = nullptr;
	};

### 拷贝带来的性能问题

当我们定义完拷贝构造函数和拷贝赋值函数，我们的类就有了完整的拷贝语义。这个时候我们可能会碰到如下问题：

	MyString getTempString()
	{
		MyString temp = "This is a temp string";
		return temp;
	}

	int main()
	{
		MyString temp;
		temp = getTempString();
		return 0;
	}

在这段程序中可能发生拷贝的地方有两处：

1. 在getTempString返回的时候，从temp拷贝构造返回值。
2. 把getTempString的返回值赋值给temp。

程序在Xcode中的运行结果如下：

> memory allocated in constructor  
memory allocated in assignment operator  
memory released in destructor  
memory released in destructor

可以看到，这里只发生了拷贝#2，拷贝#1应该是被编译器优化掉了。大家有兴趣的话可以在Visual Studio中试一下，有可能会看到拷贝#1(写到这里的时候，我把代码贴到VS2019里，在debug模式下，可以看到拷贝#1)。
  
总结起来，问题就是从函数返回大型的对象时，程序会因为不必要的拷贝而变得低效。

在传统C++俩面解决这个问题，主要有两个思路：

1. 返回指向对象的指针。这种方法需要调用者注意对象内存的管理——用完要记得销毁对象。
2. 返回对象的引用。这种方法不是通过返回值而是通过参数来返回对象。因此需要调用者事先创建好对象，然后通过引用参数将对象传给函数。

接下来让我们看看C++11是怎么解决这个问题的。

## C++11中的移动语义

### 进一步分析问题

如果我们再稍微仔细分析一下上一章最后的问题，不难发现这两处不必要的拷贝都是从临时对象做的拷贝。假如有一种新的参数类型能够区别临时对象和非临时对象，那么我们就有可能用这种新的类型来重载拷贝构造函数和拷贝赋值函数来达到我们的目的——将资源的所有权从临时变量移动到拷贝的目标身上。

### 左值，右值和右值引用

为了理解C++中的移动语义，我先介绍一下左值和右值的概念。

* ***左值*** ，就是指可以被取地址的表达式。简单的说，可以出现在等号左边的就是左值。比如：

		int a;
		a = 1;	// 这里的a是左值

	另外也可以有不是变量的左值

		int x;
		int& getRef()
		{
			return x;
		}
		getRef() = 4;

	这里，getRef()返回的是一个全局变量的引用，它的值存在固定的位置，因此是一个左值。
* ***右值*** ，则指的是没有名字的值，它们只出现表达式的计算过程中，也就是等号的右边。例如：

		string getName()
		{
			return "Baosong";
		}
		string name = getName();

	getName()返回一个在函数中构造的字符串。你可以把它的值赋给一个变量，但是它是一个临时对象，我们并不知道它的值放在哪里。所以，getName()是一个右值。

说清楚了什么是左值和右值，那么什么是右值引用呢？***右值引用*** 是C++11中新引入，是一种只绑定与右值的引用。区别与左值引用(&)，它用&&来表示。与左值引用一样，它也可以是const或者是非const的，但是我们基本不会在实际应用中用到const的右值引用(这个大家可以思考一下为什么)。让我们来看一些例子：

	const string& name = getName();	// OK
	string& name = getName();		// NOT OK
	string&& name = getName();		// OK - YEAH!

从例子中，我们可以看到const的左值引用可以绑定到右值，非const的左值引用不能绑定到右值，右值引用可以绑定到右值。那么右值引用怎么帮助我们解决问题呢？让我们接着看右值引用在作为函数参数时的行为。假如我有下面两个函数：

	void printReference(const MyString& myString)
	{
		cout << "print by const lvaue reference: " << myString.c_str() << endl;
	}

	void printReference(MyString&& myString)
	{
		cout << "print by rvalue reference: " << myString.c_str() << endl;
	}

第一个PrintReference函数是用const左值引用作为参数，从前面的例子中我们知道它既可以接受左值也可以接受右值。但是当有了第二个PrintReference的用右值引用的重载之后，右值将优先绑定到第二个PrintReference。这点我们可以通过如下代码来验证：

	int main()
	{
		MyString me("Baosong");
		printReference(me);
		printReference(getTempString());
	}

输出为：

> memory allocated in constructor  
print by const lvaue reference: Baosong  
memory allocated in constructor  
print by rvalue reference: This is a temp string  
memory released in destructor  
memory released in destructor  

终于，我们可以写出专门处理临时变量的函数了！那么这个问题的最终解决方案也是呼之欲出了。

### 移动构造函数和移动赋值函数

在右值引用的帮助下，我们可以通过重载拷贝构造函数和拷贝赋值函数来定义我们想要的从临时变量拷贝和赋值时的行为。在C+11里，这两个重载函数有它们专门的名字——移动构造函数和移动赋值函数。

* ***移动构造函数*** ，和拷贝构造函数类似，接受一个对象的实例，基于这个实例创建一个新的对象实例。但是在移动构造函数里，我们知道传入的参数是一个临时变量，所以没有必要去做拷贝。高效的做法是把资源从临时变量那里“偷”过来。以MyString为例，它的移动构造函数可以这样实现：

		MyString(MyString&& myString)
		{
			std::swap(m_data, myString.m_data);
			cout << "memory moved in move constructor" << endl;
		}

* ***移动赋值函数*** ，它对应与拷贝赋值函数。用移动构造的思路，我们应该很容易写出移动赋值函数的实现。以下是我的Mystring的移动赋值函数的实现：

		MyString& operator=(MyString&& myString)
		{
			std::swap(m_data, myString.m_data);
			cout << "memory moved in move assignment operator" << endl;
			return *this;
		}

### 验证问题已解决

我们已经为Mystring实现了移动构造函数和移动赋值函数，让我们运行之前的程序：

	int main()
	{
		MyString temp;
		temp = getTempString();
		return 0;
	}

结果为：

> memory allocated in constructor  
memory moved in move assignment operator  
memory released in destructor  

从输出结果，可以看到之前的内存拷贝已经被替换成从临时变量转移内存。所以，问题圆满解决！

## 总结

C++11引入右值引用和移动语义的目的在于从语言层面上提供对深拷贝以及浅拷贝的支持。那么我们在日常编程中关于这块，需要注意什么呢？

### Keep it simple

当读到这里的时候，大家有没有觉得移动语义好像很复杂？如何才能用最小的代价来获得正确移动语义呢？其实很简单，只要记住以下两点：

1. 当类里面需要定义指针类型的成员变量的时候，不要使用裸指针，而使用智能指针——unique_ptr, shared_ptr等。
2. 优先使用C++STL里面的容器，因为这些容器已经定义了正确的移动语义。

如果你的类成员是基本类型和其他已经定义了正确的移动语义的类组合而成的，那么你完全不需要担心任何移动语义，并且你会发现你不需要手动去实现析构函数来释放资源。

还是以MyString为例子，如果它的成员变量m_data被定义为unique_ptr<char[]>，那么我们就不用去手动实现移动构造函数以及移动赋值函数了。把这些工作交给编译器就可以了。

### 按需要为类添加移动语义

在某些特定情况下，比如用RAII模式来实现的类用来封装对FILE*的操作，我们就不得不去考虑类的移动语义了。还记得吗，要为类添加移动语义，我们需要在类里面：

* 实现移动构造函数——C::C(C&&)
* 实现移动赋值函数——C& C::operator=(C&&)
