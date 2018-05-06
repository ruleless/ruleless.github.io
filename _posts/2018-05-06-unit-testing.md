---
layout: post
title: "单元测试"
description: ""
category:
tags: []
---
{% include JB/setup %}

## 单元测试框架cppunit

使用cppunit做单元测试时，需要自己写main函数，不过一般来讲，main函数的内容都是固定的，如下：

``` c++
#include "cppunit/extensions/TestFactoryRegistry.h"
#include "cppunit/ui/text/TestRunner.h"

int main(int argc, char *argv[])
{
    CppUnit::TextUi::TestRunner runner;
    runner.addTest(CppUnit::TestFactoryRegistry::getRegistry().makeTest());
    runner.run();

    return 0;
}
```

然后，另外创建一个文件来包裹测试用例，如foo.cpp:

``` c++
#include "cppunit/TestFixture.h"
#include "cppunit/extensions/HelperMacros.h"

class FooTest : public CppUnit::TestFixture
{
    CPPUNIT_TEST_SUITE(FooTest);
    CPPUNIT_TEST(utest1);
    CPPUNIT_TEST(utest2);
    CPPUNIT_TEST_SUITE_END();
  public:
    FooTest() {}
    virtual ~FooTest() {}

    virtual void setUp() {}

    virtual void tearDown() {}

    void utest1()
    {
        CPPUNIT_ASSERT(false);
    }
    void utest2()
    {
        CPPUNIT_ASSERT(false);
    }
};

CPPUNIT_TEST_SUITE_REGISTRATION(FooTest);
```

`test1`, `test2`就是我们写测试用例的地方，习惯上一般把对一个函数的测试用例写在同一个函数中。
假如，被测试函数为foo，那么我们可以定义`void foo_test`来测试foo。

有了上面的两个源文件，我们就可以编译了：`gcc -o a.out foo.cpp main.cpp  -lcppunit`。
对于上面的示例，我们执行后的输出结果是：

``` shell
~/src/test $ ./a.out
.F.F


!!!FAILURES!!!
Test Results:
Run:  2   Failures: 2   Errors: 0


1) test: FooTest::utest1 (F) line: 20 foo.cpp
assertion failed
- Expression: false


2) test: FooTest::utest2 (F) line: 24 foo.cpp
assertion failed
- Expression: false


~/src/test $ ls
a.out  foo.cpp  main.cpp
~/src/test $ ls
a.out  foo.cpp  main.cpp
```

我们知道怎么用cppunit来做单元测试了。现在，让我们更进一步，去探索cppunit的内部设计。
首先，我们来看下，作为单元测试框架，cppunit为我们提供了什么？

  1. 各种测试断言宏，如：`CPPUNIT_ASSERT`, `CPPUNIT_ASSERT_MESSAGE`, `CPPUNIT_FAIL`
  2. 测试用例自注册功能

测试断言是任何一款单元测试框架都必须提供的，因为它是我们进行用例测试的基础设施。
相较用例自注册功能而言，测试断言的实现比较简单，也无多少技巧。
所以，我们着重讨论下用例自注册功能在cppunit中是如何做到的。

通过上面的示例，我们可以看到，使用cppunit，我们的测试程序入口是固定的。
**当我们添加测试源文件时，无需修改现有测试程序入口或其他源文件，只要重新编译一下，新加的测试用例就会被自动执行了。**

为表达方便，我们用**自注册设计**来表示该特性。
这条特性不仅对单元测试有意义，在我们平常的编程工作中，我们如果能善用这个特性，就能够事半功倍的写模块化良好的代码。
**自注册设计**是解耦的利器，是松耦合的终极形式。
下面，让我们具体来看下cppuinit是如何实现这个特性的。

cppunit的类结构图如下：

![](/images/cppunit/structure.png)

我们可以将上面的结构图拆解成三部分：

  1. TestRunner及其子类。该部分为单元测试调用入口。
  2. TestFactory及其子类。Test构建工厂，负责构建Test及其子类TestSuite。
  3. Test及其子类。

TestRunner略过不表，因为它主要就是为我们封装了构建和运行测试对象的细节。
我们主要讨论TestFactory和Test部分。

从上面的结构图来看，TestFactory采用了使用模板的工厂模式和组合模式，Test部分都采用了组合模式。

main函数共有三句：

``` c++
CppUnit::TextUi::TestRunner runner;
runner.addTest(CppUnit::TestFactoryRegistry::getRegistry().makeTest());
runner.run();
```

第一句定义了一个runner对象；
第二句调用了makeTest并将结果添加到了runner中；
第三句调用runner的run方法。

很明显，这里关键的就是makeTest方法，我们先来看看其定义：

``` c++
Test *
TestFactoryRegistry::makeTest()
{
  TestSuite *suite = new TestSuite( m_name );
  addTestToSuite( suite );
  return suite;
}

void
TestFactoryRegistry::addTestToSuite( TestSuite *suite )
{
  for ( Factories::iterator it = m_factories.begin();
        it != m_factories.end();
        ++it )
  {
    TestFactory *factory = *it;
    suite->addTest( factory->makeTest() );
  }
}
```

makeTest首先创建了一个TestSuite对象，然后遍历所有工厂，依次再调用工厂的makeTest方法构建Test，再将构建的Test对象添加到了TestSuite中。
这里面用到了组合模式，我们先忽略它，在TestSuite里再讨论该模式。

注意到：我们没有手工调用注册工厂之类的方法，整个main函数都只有三行，按道理说，`m_factories`应为空。
但实际执行时，`m_factories`却有内容，那解释只有一个，`m_factories`的注册是在main函数执行之前。
我们之前写了个cppunit的使用示例，在foo.cpp的最后有一句：

``` c++
CPPUNIT_TEST_SUITE_REGISTRATION(FooTest);
```

从名字来看，这个语句就是用来注册的。我们依据该线索，再往下看：

``` c++
#define CPPUNIT_TEST_SUITE_REGISTRATION( ATestFixtureType )      \
  static CPPUNIT_NS::AutoRegisterSuite< ATestFixtureType >       \
             CPPUNIT_MAKE_UNIQUE_NAME(autoRegisterRegistry__ )

template<class TestCaseType>
class AutoRegisterSuite
{
public:
  /** Auto-register the suite factory in the global registry.
   */
  AutoRegisterSuite()
      : m_registry( &TestFactoryRegistry::getRegistry() )
  {
    m_registry->registerFactory( &m_factory );
  }

  /** Auto-register the suite factory in the specified registry.
   * \param name Name of the registry.
   */
  AutoRegisterSuite( const std::string &name )
      : m_registry( &TestFactoryRegistry::getRegistry( name ) )
  {
    m_registry->registerFactory( &m_factory );
  }

  ~AutoRegisterSuite()
  {
    if ( TestFactoryRegistry::isValid() )
      m_registry->unregisterFactory( &m_factory );
  }

private:
  TestFactoryRegistry *m_registry;
  TestSuiteFactory<TestCaseType> m_factory;
};

template<class TestCaseType>
class TestSuiteFactory : public TestFactory
{
public:
  virtual Test *makeTest()
  {
    return TestCaseType::suite();
  }
};
```

如上，我们把`CPPUNIT_TEST_SUITE_REGISTRATION`宏及其关联的类的定义都找出来了。
在foo.cpp的示例中，我们可以把`CPPUNIT_TEST_SUITE_REGISTRATION(FooTest)`展开为：

``` c++
static AutoRegisterSuite<FooTest> autoRegisterRegistry__foo;

class TestSuiteFactory<FooTest> : public TestFactory
{
public:
  virtual Test *makeTest()
  {
    return FooTest::suite();
  }
};
```

注意，变量名不一定为`autoRegisterRegistry__foo`，但这跟我们讨论的主题关系不大，暂且用它。
利用C++模板，我们在编译期产生了一个可调用`FooTest::suite`方法并继承自`TestFactory`的类。
而依赖静态变量和C++构造函数的特性，我们得以在main函数执行前将`TestSuiteFactory<FooTest>`注册到了`TestFactoryRegistry`的`m_factories`中。

经过上面的讨论，我们的**自注册设计**已经可以在main函数之前自动注册工厂了，
并且，利用C++模板，该工厂可调用我们自定义类中的静态函数而无需另外实现一个工厂类。

到了这一步，我们就可以实现简单场景下的自注册机制了。
亦即：**无需改动现有源文件，仅通过新增源文件的方式就可以将新模块新特性添加到现有程序。**
典型场景如**命令行程序**。

使用C++模板的工厂模式实现的**自注册设计**仅是我们在松耦合编程上迈出的第一步。
有了该设计，在已经构建好的系统上，我们无需改动现有代码就可以增添新特性。
但，对于不适用线性模型的问题域或者稍微复杂的场景，则还不够。

比如，**逻辑表达式解析**就不是线性模型可解决的问题，单元测试框架如果仅使用带C++模板的工厂方法来实现也会比较简陋。

下面我们来看cppunit中的Test部分，该部分用到了组合模式。
什么是组合模式？

![](/images/cppunit/composite.png)

拥有如上结构的类结构设计模型，我们称其为组合模式。组合模式适用于抽象拥有树形结构的问题域。
单元测试框架不属于天然拥有树形结构的问题域，我们来看看cppunit是怎么将Test抽象为树形结构模型的。

![](/images/cppunit/test.png)

  + Test为接口
  + TestLeaf是对树形结构中叶子节点的抽象
  + TestComposite是对树形结构中非叶子节点的抽象
  + TestSuite是测试套件，我们可用该类实例化一个根节点

还记得makeTest的定义吗？

``` c++
Test *
TestFactoryRegistry::makeTest()
{
  TestSuite *suite = new TestSuite( m_name );
  addTestToSuite( suite );
  return suite;
}

void
TestFactoryRegistry::addTestToSuite( TestSuite *suite )
{
  for ( Factories::iterator it = m_factories.begin();
        it != m_factories.end();
        ++it )
  {
    TestFactory *factory = *it;
    suite->addTest( factory->makeTest() );
  }
}
```

TestFactoryRegistry是全局唯一的，而makeTest也只调用了一次。
所以，我们在main函数中创建了一个以TestSuite为根节点的树形对象结构。
以foo.cpp为例，其测试套件结构图如下：

![](/images/cppunit/TestSuite_foo.png)

这是比较简单的一种情况，更复杂的如：

![](/images/cppunit/TestSuite_sample.png)

到了这一步，从根节点的测试套件逐步执行到所有叶子节点的实际测试用例就比较清晰了。
