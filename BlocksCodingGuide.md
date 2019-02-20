# BlocksCodingGuide

翻译自文档：[BlocksCodingGuide](https://wiki.gnuradio.org/index.php/BlocksCodingGuide)

## 名词术语

|名词|解释|
|----|---|
|Block|一种具有输入和输出的功能处理单元|
|Port|块的单个输入或输出|
|Source|数据的生产者|
|Sink|数据的消费者|
|Connection|从输出端口到输入端口的数据流|
|Flow graph|模块和链接得集合|
|Item|数据单位。例如：fft向量、矩阵...|
|Stream|连续的数据流|
|IO signature|模块输入输出端口得描述|

## 代码结构

### 公共头文件

公共头文件在`include/foo`中定义，并安装在`$prefix/include/foo`。  
要导出的外部方法(set/get)在头文件中被定义为纯虚函数。  
公共头文件的框架如下：  

```cpp
#ifndef INCLUDED_FOO_BAR_H
#define INCLUDED_FOO_BAR_H

#include <foo/api.h>
#include <gr_sync_block.h>

namespace gr {
  namespace foo {

    class FOO_API bar : virtual public gr_sync_block
    {
    public:

      // gr::foo::bar::sptr
      typedef boost::shared_ptr sptr;

      /*!
       * \class bar
       * \brief A brief description of what foo::bar does
       *
       * \ingroup _blk
       *
       * A more detailed description of the block.
       * 
       * \param var explanation of argument var.
       */
      static sptr make(dtype var);

      virtual void set_var(dtype var) = 0;
      virtual dtype var() = 0;
    };

  } /* namespace foo */
} /* namespace gr */

#endif /* INCLUDED_FOO_BAR_H */
```

