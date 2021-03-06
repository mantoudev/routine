## 1. 加载
加载阶段JVM完成3件事情：
1. 通过一个类的全限定名来获取定义此类的二进制字节流；
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构；
3. 在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问人口。

### 1.1  数组
数组类不通过类加载器创建，由JVM直接创建。但数组类的元素类型(Element Type，数组去掉所有维度的类型)要靠类加载器创建。
数组类创建遵循以下规则：
- 数组的组件类型(Component Type，数组去掉一个维度的类型)是引用类型，递归采用本节中加载过程去加载，数组C将在加载该组件类型的类加载器的类名称空间上被标识。
- 数组的组件类型不是引用类型（如int[]数组），JVM将会把数组C标记为与引导类加载器关联。
- 数组类的可见性与它的组件类型的可见性一致，如果组件类型不是引用类型，那数组类的可见性将默认为public。

 加载阶段与连接阶段的部分内容（入一部分字节码文件格式验证动作）是交叉进行的。

 ## 2.验证
 验证阶段是一个重要但不一定必要的阶段。（因为对程序运行期没有影响）

 如果所运行全部代码(包括自己编写和第三方包中的代码)都已经被反复使用和验证过，那么在实施阶段可以考虑使用 `-Xverify:none`参数关闭大部分类验证措施，已缩短虚拟机类加载的时间。

 验证是连接的第一步。这一阶段是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身安全。

 验证阶段大致会分为下面4个阶段检验动作：
 ### 2.1 文件格式验证
 验证字节流是否符合Class文件格式规范，并且能够被当前版本虚拟机处理。验证点为：
 - 是否以魔数 0xCAFEBABE开头。
 - 主、次版本号是否在当前虚拟机处理范围之内。
 - 常量池的常量中是否有不被支持的常量类型（检查常量tag标志）
 - 指向常量的各种索引值中是否有指向不存在的常量或不符合类型的常量。
 - CONSTANT_Utf8_info 型的常量中是否有不符合UTF8编码的数据。
 - Class文件中各个部分及文件本身是否有被删除的或附加的其他信息

 ### 2.2 元数据验证
对字节码描述的信息进行语义分析，以保证其描述的信息符合Java语言规范的要求。
- 这个是否有父类（出java.lang.Object之外，所有类都应该有父类）。
- 这个类的父类是否继承了不允许继承的类（被final修饰的类）。
- 如果这个类不是抽象类，是否实现了其父类或接口中要求实现的所有方法。
- 类中的字段、方法是否与父类产生矛盾（例如覆盖了父类的final字段，或者出现不符合规则的方法重载，例如方法参数都一致，但返回值类型却不同等）。

 ### 2.3 字节码验证
通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。
- 保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作，例如不会出现类似这样的情况：在操作数栈放置了一个int类型的数据，使用时却按long类型来加载入本地变量表中。
- 保证跳转指令不会跳转到方法体以外的字节码指令上。
- 保证方法体重的类型转换是有效的。

JDK1.6之后，Javac编译器和Java虚拟机中进行了一项优化，给方法体的Code属性的属性表增加了“StackMapTable”属性，避免了过多的时间消耗在字节码验证阶段。

```
-XX:-UseSplitVerifier：关闭优化
-XX:+FailOverToOldVerifier：类型校验失败的时候退回到旧的类型推到方式进行校验。

```
JDK 1.7之后，对于主版本号大于50的Class文件，使用类型检查来完成数据流分析校验是唯一选择，不允许再退回到类型推导的校验方式。

 ### 2.4 符号引用验证
 虚拟机将符号引用转化为直接引用。符号引用验证可以看做是对类自身以外(常量池中的各种符号引用)的信息进行匹配性校验。
 - 符号引用中通过字符串描述的全限定名是否能找到对应的类。
 - 在指定类中是否存在符合方法的字段描述符以及简单名称所描述的方法和字段。
 - 符号引用中的类、字段、方法的访问性(private、protected、public、default)是否可以被当前类访问。


## 3. 准备
准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些变量使用的内存都将在方法区中进行分配。

**注意**
1. 此时进行内存分配的仅包括类变量（被static修饰的变量），而不包括实例变量，实例变量将会在对象实例化时随着对象一起分配在java堆中。
2. 这里所说的初始值“通常情况”下是数据类型的零值。当变量为final类型时，类字段的字段属性表中存在ConstantValue属性，准备阶段会被初始化为指定的值。


## 4.解析
解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。

- **符号引用：** 以一组符号来描述所引用的目标，符号可以使任何形式的字面量，只要使用时能无歧义的定位到目标即可。与虚拟机内存布局无关。

- **直接引用：** 可以直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。与虚拟机实现的内存布局相关。


解析动作主要针对类、接口、字段、类方法、接口方法、方法类型、方法句柄、调用限定符7类符号引用进行。

## 5. 初始化
初始化是类加载最后一步。初始化阶段是执行类构造器<clinit>()方法的过程。

- <clinit>()方法有编译器自动收集类中的所有类变量的赋值动作和静态语句块(static{}块)中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问。

...

## 6. 双亲委派模型


## 7. 破坏双亲委派
