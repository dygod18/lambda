本文简要概述了将java 源代码中的额lambda表达式和方法引用转化为字节码的策略，lambda 表达式在 jsr335 中已经有具体说明和实现。

本文概述了在lambda 表达式被转化为字节码时的必要转变以及语言如何在运行时参与评估lambda 表达式。本文档的大部分涉及的是function接口转换的机制。

函数接口是Java中lambda表达式的一个重要方面。 一个功能接口是一个接口，它有一个非Object方法，比如Runnable，Comparator等（Java库已经使用了这样的接口来表示回调多年）。
Lambda表达式只能出现在将其分配给类型为function接口的变量的地方。 例如：
	Runnable r = () -> { System.out.println("hello"); };
或者
	Collections.sort(strings, (String a, String b) -> -(a.compareTo(b)));

编译器生成的捕获这些lambda表达式的代码依赖于lambda表达式本身以及它所分配的功能接口类型。

翻译方案依赖于JSR 292的几个特性，包括方法句柄和方法类型的invokedynamic，方法句柄和增强的LDC字节码表单。 由于这些代码不能在Java源代码中表示，因此我们的示例将对这些功能使用伪语法：

对于方法句柄常量：MH([refKind] class-name.method-name)
对于方法类型常量：MT(method-signature)
对于动态调用：INDY((bootstrap, static args...)(dynamic args...))

读者应该具有jsr292 的工作经验
翻译方案还假定正在由JSR-292专家组为Java SE 8指定的新功能：用于反映常量方法句柄的API。

Translation strategy

有很多方法可以表示字节码中的lambda表达式，比如内部类，方法句柄，动态代理等等。每种方法都有优点和缺点。 在选择策略时，有两个相互竞争的目标：通过不采取特定的策略来提高未来优化的灵活性，同时提供类文件表示的稳定性。通过使用JSR 292中的invokedynamic特性，我们可以实现这两个目标，将字节码中的lambda创建的二进制表示与运行时评估lambda表达式的机制分开。 我们不用生成字节码来创建实现lambda表达式的对象（例如调用内部类的构造函数），而是描述构造lambda的配方，并将实际构造委托给语言运行库。 该配方被编码在invokedynamic指令的静态和动态参数列表中。

invokedynamic的使用让我们推迟翻译策略的选择，直到运行时间。 运行时实现可以动态地选择一个策略来评估lambda表达式。 运行时实现选择隐藏在用于lambda构建的标准化（即平台规范的一部分）API之后，以便静态编译器可以发出对该API的调用，并且JRE实现可以选择它们的优选实现策略。 invokedynamic机制允许这样做，而没有性能成本，这个后期绑定方法可能会强加。

当编译器遇到一个lambda表达式时，它首先将lambda体降低（去除）到一个方法中，该方法的参数列表和返回类型与lambda表达式匹配，可能带有一些附加参数（对于从词法作用域捕获的值，如果有的话）。 ）在捕获lambda表达式的时候，它会生成一个invokedynamic调用位置，当调用它时，将返回lambda转换到的函数接口的一个实例。这个调用站点被称为给定lambda的lambda工厂。 lambda工厂的动态参数是从词法作用域捕获的值。 lambda工厂的引导方法是Java语言运行库中的标准化方法，称为lambda元数据集。静态引导参数在编译时捕获有关lambda的已知信息（它将被转换到的函数接口，desugared lambda体的方法句柄，关于SAM类型是否可序列化的信息等）

方法引用与lambda表达式的处理方式相同，除了大多数方法引用不需要被解析成新的方法; 我们可以简单地为引用的方法加载一个常量方法句柄，并将其传递给元函数。

Lambda body desugaring

将lambda转换成字节码的第一步是将lambda体转换成一个方法。

有几个选择，必须围绕desugaring：
- 我们是剖析一个静态方法或实例方法吗？
- 被剖析的方法会落在哪个类上？
- 被剖析方法的可访问性是怎样的？
- 被剖析方法的方法名是？
- 如果需要适配lambda body签名和函数接口方法签名（如装箱，拆箱，原始扩大或缩小转换，可变参数转换等）之间的差异，那么desugared方法应遵循lambda体的签名， 函数接口的方法，还是在两者之间？ 谁负责所需的适应？
- 如果lambda捕获封闭范围的参数，应该如何在desugared方法签名中表示这些参数？ （它们可以是在参数列表的开头或结尾添加的单个参数，或者编译器可以将它们收集到一个“框架”参数中。）

与desugaring lambda体相关的问题是方法引用是否需要生成适配器或“桥”方法。

编译器将推断lambda表达式的方法签名，包括参数类型，返回类型和抛出的异常; 我们将这称为自然签名。 Lambda表达式也有一个目标类型，这将是一个功能接口; 我们将调用lambda描述符作为目标类型的擦除描述符的方法签名。 从实现功能接口并捕获lambda行为的lambda工厂返回的值称为lambda对象。

所有的东西都是平等的，私有方法比非私有方法更适合，实例方法比静态方法更适合，最好是在lambda表达式出现在最内层的类中，最好是签名应该匹配lambda的body签名，额外参数应该被添加到捕获值的参数列表的前面，并且根本不会抛出方法引用。 但是，也有例外的情况下，我们可能不得不偏离这个基准线策略。

Desugaring example -- "stateless" lambdas

要翻译的最简单的lambda表达式就是从其封闭范围（无状态的lambda）中捕获的状态：

class A {
    public void foo() {
        List<String> list = ...
        list.forEach( s -> { System.out.println(s); } );
    }
}

lambda的自然签名是（String）V; forEach方法采用其lambda描述符为（Object）V的Block <String>。 编译器将lambda体解析成一个静态方法，其签名是自然签名，并生成一个desugared主体的名称。

class A {
    public void foo() {
        List<String> list = ...
        list.forEach( [lambda for lambda$1 as Block] );
    }

    static void lambda$1(String s) {
        System.out.println(s);
    }
}

Desugaring example -- lambdas capturing immutable values

另一种形式的lambda表达式涉及捕获包含最终（或有效的最终）局部变量和/或来自封闭实例的字段（我们可以将其视为封装该引用的最终捕获）。
class B {
    public void foo() {
        List<Person> list = ...
        final int bottom = ..., top = ...;
        list.removeIf( p -> (p.size >= bottom && p.size <= top) );
    }
}

在这里，我们的lambda从封闭范围捕获最后的局部变量bottom和top。
desugared方法的签名将是自然签名（Person）Z，在参数列表的前面添加一些额外的参数。 编译器对于这些额外的参数如何表示有一定的自由度; 他们可以单独预先考虑，装入框架类，装入数组等。最简单的方法是单独预先安排它们：
class B {
    public void foo() {
        List<Person> list = ...
        final int bottom = ..., top = ...;
        list.removeIf( [ lambda for lambda$1 as Predicate capturing (bottom, top) ]);
    }

    static boolean lambda$1(int bottom, int top, Person p) {
        return (p.size >= bottom && p.size <= top;
    }
}

或者，捕获的值（top和bottom）可以被装箱成一个框架或一个数组; 关键是在脱钩的lambda方法的签名中出现的额外参数的类型之间的一致性，以及它们作为（动态）参数出现在lambda工厂的类型。 由于编译器在控制这两者，并且它们同时生成，所以编译器在如何打包捕获的参数方面具有一定的灵活性。

The Lambda Metafactory

Lambda捕获将由invokedynamic调用站点实现，其静态参数描述lambda体和lambda描述符的特征，其动态参数（如果有的话）是捕获的值。 当被调用时，这个调用站点返回一个对应的lambda体和描述符的lambda对象，绑定到捕获的值。 此调用站点的引导方法是一种称为lambda元数据的指定平台方法。 （我们可以为所有的lambda表单创建一个单一的元数据集，或者为常见的情况提供专门的版本）.VM只会在每个捕获站点调用一次元数据集; 然后它将连接呼叫站点并且开路。 呼叫站点是懒惰地链接的，所以永远不会被调用的工厂站点是不会被链接的。 基本的metafactory的静态参数列表如下所示：
metaFactory(MethodHandles.Lookup caller, // provided by VM
            String invokedName,          // provided by VM
            MethodType invokedType,      // provided by VM
            MethodHandle descriptor,     // lambda descriptor
            MethodHandle impl)           // lambda body

前三个参数（调用者，invokedName，invokedType）由虚拟机自动堆栈在callsite链接处。

descriptor 标识了lambda转换到的函数接口方法。（通过方法句柄的反射API，metafactory可以获得功能接口类的名称以及其主方法的名称和方法签名。）

impl 标识lambda方法，可以是desugared lambda体或方法引用中指定的方法。

功能接口方法的方法签名和实现方法之间可能存在一些差异。 实现方法可能具有与捕获的参数相对应的额外参数。 其余的论点也可能不完全一致; 允许某些适应（分类，装箱），如适应中所述。

Lambda capture

现在我们准备描述lambda表达式和方法引用的函数接口转换的翻译。 我们可以将例A翻译为：
class A {
    public void foo() {
            List<String> list = ...
            list.forEach(indy((MH(metaFactory), MH(invokeVirtual Block.apply),
                               MH(invokeStatic A.lambda$1)( )));
        }

    private static void lambda$1(String s) {
        System.out.println(s);
    }
}

因为A中的lambda是无状态的，所以lambda工厂的动态参数列表是空的。

例如B，动态参数列表不是空的，因为我们必须为lambda工厂提供bottom和top的值：
class B {
    public void foo() {
        List<Person> list = ...
        final int bottom = ..., top = ...;
        list.removeIf(indy((MH(metaFactory), MH(invokeVirtual Predicate.apply),
                            MH(invokeStatic B.lambda$1))( bottom, top ))));
    }

    private static boolean lambda$1(int bottom, int top, Person p) {
        return (p.size >= bottom && p.size <= top;
    }
}

Static vs instance methods

因为它们不以任何方式使用封闭对象实例（不要引用this，super或封闭实例的成员），所以类似上面的那些Lambdas可以转换为静态方法。总的来说，我们将参考lambda 使用this，super或捕获封装实例的成员作为实例捕获lambda表达式。

 Non-instance-capturing 的lambda被转换为私有的静态方法。Instance-capturing 的lambda被转换为私有实例方法。 这简化了实例捕获lambda的desugaring，因为lambda体中的名字与desugared方法中的名称相同，并与可用的实现技术（绑定的方法句柄）良好地啮合。捕获实例捕获lambda时，接收方 （this）被指定为第一个动态参数。

作为一个例子，考虑一个lambda捕获一个字段minSize：
list.filter(e -> e.getSize() < minSize )
我们把它作为一个实例方法来解析，并且把接收者作为第一个被捕获的参数传递给它：
list.forEach(INDY((MH(metaFactory), MH(invokeVirtual Predicate.apply),
                   MH(invokeVirtual B.lambda$1))( this ))));

private boolean lambda$1(Element e) {
    return e.getSize() < minSize;
}




