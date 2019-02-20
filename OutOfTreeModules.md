# 拓展GNU Radio的功能和模块

翻译自文档：[OutOfTreeModules](https://wiki.gnuradio.org/index.php/OutOfTreeModules)

## 什么是树外模块

树外模块(out-of-tree module)是指不在GNU Radio源代码树中的代码组件。通常情况下，如果你想用自己的功能和模块来拓展GNU Radio，那么这个模块就是你创建的（即你通常不会向实际的GNU Radio源代码树添加内容，除非你打算将它提交给开发者用于上游整合）。这样使得用户可以自主维护代码，并且允许用户使用自定义的功能。  

## 教程1：创建一个树外模块

在下面得教程中，我们将使用名为`howto`的树外模块。第一步是创建词模块。  
使用`gr_modtool`会将这个操作变得很容易。只需要用命令行指定你想要创建新模块的目录（应该在GNU Radio的源码树外），然后执行如下：

```shell
% gr_modtool newmod howto
Creating out-of-tree module in ./gr-howto... Done.
Use 'gr_modtool add' to add a new block to this currently empty module.
```

如果一切顺利，你会看到一个新目录叫做`gr-howto`，后续的工作将在这个目录下进行。

## 模块的结构

我们直接进入`gr-howto`模块，看看它的构成：

```shell
gr-howto % ls
apps  cmake  CMakeLists.txt  docs  examples  grc  include  lib  python  swig
```

它由几个子目录组成。任何将用C++（或C，或任何非Python语言）编写的东西都会被放入`lib/`。对于C++文件，我们通常会将一些头文件放在`include/`下（如果它们要导出）或者放在`lib/`下（如果它们仅在编译期间相关，但以后不再安装，例如`_impl.h`文件。你会在下一个教程中看到这些）。  
当然，Python的东西会放在`python/`目录下。这包括单元测试（未安装）和部分已安装的Python模块。  
用C++编写的GNU Radio的模块，通过SWIG（简化的包装器和接口生成器）自动创建的粘合代码，在Python中依然是可用的。在`swig/`子目录下会存放一些关于如何生成粘合代码的说明。`gr_modtool`会帮助我们处理这些过程。  
如果希望能在GNU Radio图形界面中调用自定义的模块，则需要添加模块得XML描述并将其放入`grc/`。  
建立好的系统会带来一些开销：`cmake/`文件夹以及每个子目录下都存在的`CMakeLists.txt`文件。暂时可以忽略前者，因为它主要提供了CMake关于如何查找GNU Radio库的说明。为了确保正确构建模块，需要对CMakeLists.txt文件进行大量编辑。

## 教程2：用C++编写一个快(square_ff)

我们将创建一个用来计算浮点输入的平方值的模块来做作为我们的第一个例子。这个模块将接收单个float输入流并生成单个float输出流，即对于每个传入的浮点值，都会输出一个浮点值，且输出值为输入值的平方。  
遵循命名规则，因为模块具有浮点输入和浮点输出，所以模块取名`square_ff`。  
我们将把这个模块以及本文中编写的其他模块都放到howto的Python模块中。这样我们在Python中也可以调用模块：

```python
import howto
sqr=howto.square_ff()
```

### 创建文件

第一步是为模块创建空文件并编辑CMakeLists.txt文件。  
`gr_modtool`再次帮你做好了这部分工作。在命令行中进入`gr-howto`目录，并输入以下指令：

```shell
gr-howto % gr_modtool add -t general -l cpp square_ff
GNU Radio module name identified: howto
Block/code identifier: square_ff
Language: C++
Please specify the copyright holder:
Enter valid argument list, including default arguments: 
Add Python QA code? [Y/n] 
Add C++ QA code? [y/N] 
Adding file 'square_ff_impl.h'...
Adding file 'square_ff_impl.cc'...
Adding file 'square_ff.h'...
Editing swig/howto_swig.i...
Adding file 'qa_square_ff.py'...
Editing python/CMakeLists.txt...
Adding file 'howto_square_ff.xml'...
Editing grc/CMakeLists.txt...
```

在输入的指令中，我们指定要添加一个新模块，它的类型是"general"（因为我们还没有确定它是什么模块类型），它的名称叫做`square-ff`。这个模块是用C++创建的，它目前没有指定的版权所有者。`gr-modtool`然后询问你的模块是否接受任何参数（它不接受，所以我们将其留空），是否需要Python的QA（质量保证）代码（是的我们需要），是否需要C++的QA代码（不，暂时不需要）。  
现在，比较一下变化前后的CMakeLists.txt文件，你就能看到`gr-modtool`做了什么工作。你还可以看到很多新的文件，现在我们要对它们进行编辑来让模块可以工作。

### 测试驱动编程

我们现在可以编写C++代码，但是作为高度发展的现代程序员，我们应该首先编写测试代码。毕竟我们确实有一个很好的编码习惯：将一串浮点数据流作为输入，并产生一个浮点数据流作为输出，输出应为输入的平方。  
打开`python/qa_square_ff.py`，对它进行编辑：

```py
from gnuradio import gr,gr_unittest
from gnuradio import blocks
import howto_swig as howto

class qa_square_ff (gr_unittest.TestCase):

    def setUp (self):
        self.tb = gr.top_block ()

    def tearDown (self):
        self.tb = None

    def test_001_t(self):
        src_data = (-3, 4, -5.5, 2, 3)
        expected_result = (9, 16, 30.25, 4, 9)
        src = blocks.vector_source_f(src_data)
        sqr = howto.square_ff()
        dst = blocks.vector_sink_f()
        self.tb.connect(src, sqr)
        self.tb.connect(sqr, dst)
        self.tb.run()
        result_data = dst.data()
        self.assertFloatTuplesAlmostEqual(expected_result, result_data, 6)

if __name__ == '__main__':
    gr_unittest.run(qa_square_ff, "qa_square_ff.xml")
```

gr_unittest是标准Python模块unittest的扩展。gr_unittest添加了对于浮点数和复数元祖的近似相等性的检测。Unittest使用了Python的反射机制来查找以`test_`开头的所有方法并执行它们。Unittest将对于`test_*`的每一个调用都匹配成对于`setUp`和`tearDown`的调用。  
当我们运行测试的时候，`gr_unittest.main`按照`setUp`，`test_001_t`，`tearDown`的顺序分别调用这些方法。  

`test_001_t`创建了一个包含三个节点的流程图。  
`blocks.vector_source_f(src_data)`获取源数据并返回完成的状态。  
`howto.square_ff`则是我们正在测试得模块。  
`blocks.vector_sink_f`则负责收集`howto.square_ff`的输出。  
`run()`方法负责执行流程图，直到所有的模块都被执行完毕。最后我们检查`square_ff`的执行结果是否和正确结果一致。  
注意，这样的测试模块一般在安装模块之前调用。这就意味着我们需要一些技巧使得模块在测试的时候被加载。CMake通过适当改变PYTHONPATH来处理大多数的问题。此外，在这个文件中我们导入的是`howto_swig`而不是`howto`。  
为了让CMake知道这个测试的存在，gr_modtool修改了CMakeLists.txt的以下代码：

```txt
########################################################################
# Handle the unit tests
########################################################################
include(GrTest)

set(GR_TEST_TARGET_DEPS gnuradio-howto)
set(GR_TEST_PYTHON_DIRS ${CMAKE_BINARY_DIR}/swig)
GR_ADD_TEST(qa_square_ff ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/qa_square_ff.py)
```

### C++代码（第一部分）

现在我们已经有了测试用例，让我们编写C++代码。所有的信号处理模块都是`gr::block`和它的子类派生出来的。  
`gr_modtool`已经为我们提供了三个文件来定义一个新模块：`lib/square_ff_impl.h`，`lib/square_ff_impl.cc`和`include/howto/square_ff.h`。我们所需要做的就是修改它们来达成我们的目标。完成这个教程后请阅读和理解[模块编码指南](./BlocksCodingGuide.md)，了解这些文件的结构和原因！  
首先，我们要关注头文件。因为我们编写的模块非常简单，所以我们实际上不需要更改它们（在运行`gr_modtool`之后`include/`中的头文件通常都是相对完整的，除非我们需要添加一些公共的方法，例如一些get和set方法）。所有我们只需要关注剩下的`lib/square_ff_impl.cc`。  
`gr_modtool`通过`<++>`来提示你需要进行修改的代码的位置。

```cpp
square_ff_impl::square_ff_impl()
      : gr::block("square_ff",
                  gr::io_signature::make(1, 1, sizeof (float)), // input signature
                  gr::io_signature::make(1, 1, sizeof (float))) // output signature
    {
        // empty constructor
    }
```

因为平方模块不需要设置任何东西，所以构造函数本身是空的。  
值得关注的部分是输入和输出信号的定义：在输入端，我们有一个只允许浮点输入的端口。输出端是相同的定义。  

```cpp
void square_ff_impl::forecast (int noutput_items, gr_vector_int &ninput_items_required)
    {
      ninput_items_required[0] = noutput_items;
    }
```

`forecast()`是一个函数，它的职能是告诉调度程序生成`noutput_items`输出项需要多少个输入项。在这个例子里，它们的数目是相等的。下标0表示这是第一个端口，当然我们也只有一个端口。`forecast`在许多模块中都是这种情况，例如`gr::block`，`gr::sync_block`，`gr::sync_decimator`都是采用的默认定义方法来考虑速率的变化。  
最后，`general_work()`是`gr::block`中的纯虚函数，所以我们需要重载这个方法。`general_work()`是进行实际信号处理的方法。  

```c++
int square_ff_impl::general_work (int noutput_items,
                                  gr_vector_int &ninput_items,
                                  gr_vector_const_void_star &input_items,
                                  gr_vector_void_star &output_items)
    {
      const float *in = (const float *) input_items[0];
      float *out = (float *) output_items[0];

      for(int i = 0; i < noutput_items; i++) {
        out[i] = in[i] * in[i];
      }

      // Tell runtime system how many input items we consumed on each input stream.
      consume_each (noutput_items);

      // Tell runtime system how many output items we produced.
      return noutput_items;
    }
```

有一个指向输入缓冲区的指针和一个指向输出缓冲区的指针，以及一个for循环来将输入数据平方值赋值给输出缓存。  

### 使用CMake