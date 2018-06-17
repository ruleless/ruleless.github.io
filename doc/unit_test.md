## xtest使用介绍

### 目录结构

``` shell
|----.
|    |----./jni
|    |    |----./jni/Android.mk
|    |    |----./jni/Application.mk
|    |    |----./jni/xtest_demo.cpp
|    |----./libxtest
|    |    |----./libxtest/Android.mk
|    |    |----./libxtest/Makefile
|    |    |----./libxtest/xtest.c
|    |    |----./libxtest/xtest.h
|    |----./Makefile
|    |----./vcproject
|    |    |----./vcproject/libxtest
|    |    |    |----./vcproject/libxtest/libxtest.sln
|    |    |    |----./vcproject/libxtest/libxtesttest
|    |    |    |    |----./vcproject/libxtest/libxtesttest/dm.cpp
|    |    |    |    |----./vcproject/libxtest/libxtesttest/libxtesttest.vcxproj
|    |    |    |----./vcproject/libxtest/libxtest.vcxproj
|    |    |    |----./vcproject/libxtest/testxtest
|    |    |    |    |----./vcproject/libxtest/testxtest/c.c
|    |    |    |    |----./vcproject/libxtest/testxtest/cpp.cpp
|    |    |    |    |----./vcproject/libxtest/testxtest/testxtest.vcxproj
|    |----./xtest_app.c
```

  * libxtest: 单测核心文件(主要就是xtest.c和xtest.h)
  * jni目录: android平台需要关注的demo之一
  * vcproject: 里面包含了windows的工程demo
  * Makefile: linux的编译文件

### xtest使用步骤

**1. 添加main函数**

``` c++
int main(int argc, char ** argv)
{
    return xtest_start_test(argc, argv);
}
```

**2. 注册单测用例**

在各代码文件中，调用`TEST_F(suite_name, case_name, setup_func, teardown_func)`或
`TEST(suite_name, case_name)`注册用例，如：

``` c++
TEST(test_some_api, max)
{
    EXPECT_TRUE(max(1, 2) == 2 && "max wrong 1,2");
    EXPECT_TRUE(max(-1, 0) == 0 && "max wrong -1, 0");
    EXPECT_TRUE(max(-1, 1) == 1 && "max wrong -1, 1");
}
```

`TEST_F` 相比 `TEST` 多了 `setup_func` 和`teardown_func`。
这两个函数前者用于测试用例执行之前的初始化和执行之后的去初始化操作。

### xtest单元测试宏

|--|--|--|
|致命断言|非致命断言|说明|
|**ASSERT_TRUE(condition)**|**EXPECT_TRUE(condition)**|condition is true|
|**ASSERT_FALSE(condition)**|**EXPECT_FALSE(condition)**|condition is false|
|**ASSERT_EQ(val1, val2)**|**EXPECT_EQ(val1, val2)**|val1 == val2|
|**ASSERT_NE(val1, val2)**|**EXPECT_(val1, val2)NE**|val1 != val2|
|**ASSERT_LT(val1, val2)**|**EXPECT_LT(val1, val2)**|val1 < val2|
|**ASSERT_LE(val1, val2)**|**EXPECT_LE(val1, val2)**|val1 <= val2|
|**ASSERT_GT(val1, val2)**|**EXPECT_GT(val1, val2)**|val1 > val2|
|**ASSERT_GE(val1, val2)**|**EXPECT_GE(val1, val2)**|val1 >= val2|
|**ASSERT_STREQ(str1, str2)**|**EXPECT_STREQ(str1, str2)**|str1 == str2|
|**ASSERT_STRNE(str1, str2)**|**EXPECT_STRNE(str1, str2)**|str1 != str2|
|**ASSERT_STRCASEEQ(str1, str2)**|**EXPECT_STRCASEEQ(str1, str2)**|str1 == str2(忽略大小写)|
|**ASSERT_STRCASENE(str1, str2)**|**EXPECT_STRCASENE(str1, str2)**|str1 != str2(忽略大小写)|

### xtest原理

单元测试框架最重要的就两点：单元测试断言、用例执行函数自注册。

  1. 测试断言用于判断用例是否通过，如果不通过则给出提示，上面列出了xtest支持的测试断言
  1. 用例执行函数自注册用于在添加新的用例执行函数时不必与现有单测代码产生耦合
  
测试断言比较简单，我们这里主要来看下用例执行函数自注册。
C/C++中，在不破坏现有代码的条件下，让新代码执行的唯一办法是将新代码提前到main函数前执行。
在C++中静态或全局对象的构造函数可在main函数前执行；C语言在语法上没提供相应的机制。

xtest是用C语言写的，所以得另辟蹊径。
ELF格式的二进制文件中有.ctors和.dtors段，
.ctors段中存有在main函数之前执行的函数的指针，.dtors段中存有在main函数之后执行的函数的指针。

``` c++
void init(void)
{
    printf("enter init\n");
}

__attribute__((section(".ctors"))) void (*funcptr)(void) = init;

int main()
{
    printf("enter main\n");
    return;
}
```

上面的方式就可以把init函数放在main函数之前执行，为方便起见，GCC还提供如下的方式来达到同样的效果：

``` c++
__attribute__((constructor)) void init(void)
{
    printf("enter init\n");
}
```

有了上面的机制，我们就可以仅通过重新编译把新代码入口函数指针编译进.ctors段。
这样就可以不用改动现有代码仅通过重新编译就实现用例执行函数的自注册。
我们具体看下xtest中的做法。

``` c++
#define TEST(catagory, name)                                            \
    static void _X_TEST_##catagory##_##name##_FUNC(void);               \
    __attribute__((constructor))                                        \
    static void _X_TEST_##catagory##_##name##_CONSTRUCT(void) {         \
        xtest_register(#catagory, #name, __FILE__, __LINE__,            \
                       NULL, _X_TEST_##catagory##_##name##_FUNC, NULL); \
    }                                                                   \
    void _X_TEST_##catagory##_##name##_FUNC(void)
```

上面是xtest.h中对TEST的定义，`_X_TEST_##catagory##_##name##_CONSTRUCT`会在main函数之前自动执行。
该函数通过`xtest_register`注册了用例执行函数`_X_TEST_##catagory##_##name##_FUNC(void)`。
我们再来看看`xtest_register`的定义：

``` c++
XTESTAPI void xtest_register(const char *catagory, const char *name, 
                    const char *file, int lineno,
                    xtest_entry_func_t init, 
                    xtest_entry_func_t entry, 
                    xtest_entry_func_t fini)
{
    char buf[1024];
    
    if (s_infos.ninfo >= s_infos.sizeinfo) {
        s_infos.sizeinfo = s_infos.sizeinfo ? s_infos.sizeinfo * 2 : 32;
        s_infos.infos = realloc(s_infos.infos, s_infos.sizeinfo * sizeof(xtest_entry_info_st *));
        assert(s_infos.infos);
    }
    
    snprintf(buf, sizeof(buf), "%s.%s", catagory, name);
    s_infos.infos[s_infos.ninfo++] = make_entry(buf, file, lineno, init, entry, fini);
}
```

用例执行函数被注册到了`s_infos`变量中，`s_infos`定义如下：

``` c++
/* test case info struct */
typedef struct xtest_entry_info_st {
    const char          *name;
    const char          *file;
    int                 lineno;
    xtest_entry_func_t  init;
    xtest_entry_func_t  entry;
    xtest_entry_func_t  fini;
} xtest_entry_info_st;

/* test case entry set for collecting entries */
typedef struct entry_infos_st {
    xtest_entry_info_st **infos;
    int                 ninfo;
    int                 sizeinfo;
} entry_infos_st;

/* test cases info */
static entry_infos_st s_infos;
```

我们在最开始讲xtest的用法时，就说过在main函数用要调用`xtest_start_test`：

``` c++
int main(int argc, char ** argv)
{
    return xtest_start_test(argc, argv);
}
```

`xtest_start_test`通过遍历调用注册到s_infos.infos中的函数来完成所有的单测用例执行。

## 单元测试要点

### 单测代码组织形式

**1.单测代码放在独立源文件中。如xxx_test.cpp**

如：

``` shell
|----.
|    |----Makefile
|    |----readme.md
|    |----src
|    |    |----src/condition.cpp
|    |    |----src/condition_factory.cpp
|    |    |----src/condition_factory.hpp
|    |    |----src/condition.hpp
|    |    |----src/iterator_wrapper.hpp
|    |    |----src/Makefile
|    |----./test
|    |    |----test/condition_test.cpp
|    |    |----test/Makefile
|    |    |----test/test_main.cpp
```

**2.单元测试直接集成在代码文件中**

如：

``` c++
static void foo()
{
    ...
}

#ifdef UNIT_TEST
int main(int argc, char *argv[])
{
    ...
    exit(0);
}
#else
TEST(test_some_api, foo)
{
    ...
}

int main(int argc, char ** argv)
{
    return xtest_start_test(argc, argv);
}
#endif
```

**3.独立源文件 VS 集成在源文件中**

独立源文件更优雅，但是需要将所有待测函数都暴露出接口到.h文件中。
有些直接写在.c/.cpp文件中的结构体和函数都不方便测试。
集成在源文件中不优雅，但是易测试。
特别是有部分static函数是只允许同文件访问的，以及有些结构体，可能都直接定义在.c/.cpp文件中。
此时，采用集成到源文件中的方式更易于测试。

### 单元测试注意事项

**1.针对同一函数如果测试的点位太多，则拆分为多个测试用例**

``` c++
void demo_test_setup(void){}
void demo_test_teardown(void){}

// 输入为null的测试
TEST_F(demotest, match_input_null, demo_test_setup, demo_test_teardown)
{ 
    // 输入为null 
}
// 完全匹配的测试，如aBc匹配aBc
TEST_F(demotest, match_all, demo_test_setup, demo_test_teardown)
{ 
    // 完全匹配 
}
// 通配符*匹配测试
TEST_F(demotest, match_start_symbol, demo_test_setup, demo_test_teardown)
{ 
    // 通配符*匹配
}
//  通配符 ？ 匹配测试
TEST_F(demotest, match_qusttion_symbol, demo_test_setup, demo_test_teardown)
{ 
    //  通配符 ？ 匹配
}
// 复杂匹配测试
TEST_F(demotest, match_complex, demo_test_setup, demo_test_teardown)
{ 
    // 复杂匹配 aB?d*
    // 其它较为复杂的匹配
}
```

**2.用例的检查点要明确**

``` c++
TEST(demotest, foo)
{
    foo();
}
```

该用例没有任何检查点，体现不出测试的价值，它只能给测试者一个"哦，它没有抛异常"的结论。
我们应该仔细挖掘其内的实现，找到检查点进行检查，比如它会改变某个全局的状态变量：

``` c++
TEST(demotest, foo)
{
    foo();
    EXPECT_STREQ("OK", g_status);
}
```

**3.用例失败时能精确的定位问题**

**4.用例执行结果稳定**

对于同一个用例，每次执行都要能获得相同的结果。
而不是通过随机数或根据执行次数、编译Debug或Release都会有所变化。

**5.用例执行时间不要太久**

**6.用例对测试环境无破坏性**

每个用例执行完成后，都要恢复全局的状态、销毁创建的资源。
不要使后面的用例依赖于前面一个用例的结果。
否则可能会出现调整用例后，结果不一致的情况。

**7.用例的命名清晰易理解**

使用有意义的测试套名称和testcase名称，请勿使用`foo_test1`，`foo_test2`，`foo_case1`之类的名称，
正确的命名类似于`foo_input_null`, `foo_compare_ok`，望文生义。

**8.测试函数小而精**

单元测试的粒度始终保持在函数级别，不要试图去做功能级或场景级的测试，
那些测试虽然很重要，那但是接口测试或自动化测试该做的事。

**9.测试用例要持续更新**

**10.自动生成配置文件，而不要引用配置文件**

单测中涉及到外部配置文件，不能通过相对路径去引用现成的文件，最好自己**在单测代码中生成配置文件**。

**11.单测用例的期望结果或配置构建等关键地方需有注释说明**

添加注释**把你测试的意图**说出来 ，后续其他人员可以进行维护和修订，也可以检查单测是否合理。

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
