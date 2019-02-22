### 序列化方案需要考虑的几个要素
- 是否支持 循环检测/对象共享
- 是否需要包含元数据 schema
- 是否支持跨平台
- 文本/二进制
- 是否支持向前、向后兼容性


# java原生序列化（java-build-in）

### 简介
>Java 对象序列化是 JDK 1.1 中引入的一组开创性特性之一，用于作为一种将 Java 对象的状态转换为字节数组，以便存储或传输的机制，以后，仍可以将字节数组转换回 Java 对象原有的状态。  
序列化的思想是 “冻结”对象状态，传输对象状态（写到磁盘、通过网络传输等等），然后 “解冻” 状态，重新获得可用的 Java 对象。   
这要归功于 **ObjectInputStream(反序列化)/ObjectOutputStream(序列化)** 类、完全保真的元数据以及用Serializable 标识接口标记object类，从而 “参与” 这个过程。  
实际也即是：jdk内部程序包，提供一套数据存储与标记协议，并实现根据此协议来进行对象到byte[]以及byte[]到对象的相互转换。  
http://developer.51cto.com/art/201506/479979.htm

#### 特性:  
- JDK自行实现，无需manual控制  
- 对象类需要实现serializable接口，并给出serialVersionUID.serialVersionUID不变，才能保证反序列化的成功  
- 不支持static以及transient类型的序列化  
- 不支持跨平台
- 需要知道元数据(类文件)
- 数据域不受保护，私有域也能操作
- 数据传输中安全性需要自行实现，重写readObject/writeObject方法
- 空间开销大  
- 效率并非理想
#### 序列化步骤:  


1. 将对象实例相关的类元数据输出  
2. 递归地输出类的超类描述直到不再有超类(超类也需要实现serializable接口)  
3. 类元数据完了以后，开始从最顶层的超类开始输出对象实例的实际数据值    
4. 从上至下递归输出实例的数据

#### 序列化源码简读（重要方法）:  
1. ObjectOutputStream() 初始化 ObjectOutputStream
2. writeStreamHeader() 输出标记-使用序列化协议、序列化版本  
3. writeObject() 开始序列化 
4. writeClassDesc() 递归读写类的元数据信息、变量定义信息
5. writeSerialData() 循环调用，写如序列化的数据
6. 结束writeObject();

#### 序列化过程解析:
```
package test;
import java.io.*;
public class SerializableTest {
    public static void main(String[] args) throws Exception {
        FileOutputStream fos = new FileOutputStream("temp.out");
        ObjectOutputStream oos = new ObjectOutputStream(fos);
        TestObject testObject = new TestObject();
        oos.writeObject(testObject);
        oos.flush();
        oos.close();

        FileInputStream fis = new FileInputStream("temp.out");
        ObjectInputStream ois = new ObjectInputStream(fis);
        TestObject deTest = (TestObject) ois.readObject();
        System.out.println(deTest.testValue);
        System.out.println(deTest.parentValue);
        System.out.println(deTest.innerObject.innerValue);
    }
}
class Parent implements Serializable {
    private static final long serialVersionUID = -4963266899668807475L;
    public int parentValue = 100;
}
class InnerObject implements Serializable {
    private static final long serialVersionUID = 5704957411985783570L;
    public int innerValue = 200;
}
class TestObject extends Parent implements Serializable {
    private static final long serialVersionUID = -3186721026267206914L;
    public int testValue = 300;
    public InnerObject innerObject = new InnerObject();
}
```
```
序列化之后的内容为：
aced 0005 7372 000f 7465 7374 2e54 6573
744f 626a 6563 74d3 c67e 1c4f 132a fe02
0002 4900 0974 6573 7456 616c 7565 4c00
0b69 6e6e 6572 4f62 6a65 6374 7400 124c
7465 7374 2f49 6e6e 6572 4f62 6a65 6374
3b78 7200 0b74 6573 742e 5061 7265 6e74
bb1e ef0d 1fc9 50cd 0200 0149 000b 7061
7265 6e74 5661 6c75 6578 7000 0000 6400
0001 2c73 7200 1074 6573 742e 496e 6e65
724f 626a 6563 744f 2c14 8a40 24fb 1202
0001 4900 0a69 6e6e 6572 5661 6c75 6578
7000 0000 c8

按段格式化之后为：
aced 0005 7372 000f 
7465 7374 2e54 6573 744f 626a 6563 74
d3   c67e 1c4f 132a fe
02   0002 49 0009 
74 6573 7456 616c 7565 
4c 000b 69 6e6e 6572 4f62 6a65 6374 
74 0012 
4c 7465 7374 2f 49 6e6e 6572 4f62 6a65 6374 3b
78 
72 
000b 
74 6573 742e 5061 7265 6e74
bb1e ef0d 1fc9 50cd 
02
0001
49 
000b 
7061 7265 6e74 5661 6c75 65  
78 
70
0000 0064 
0000 012c 
73 
72
0010 
7465 7374 2e49 6e6e 6572 4f62 6a65 6374 
4f2c 148a 4024 fb12
02
0001 
49
000a
69 6e6e 6572 5661 6c75 65
78
70
0000 00c8

```
解析：
```
aced：STREAM_MAGIC，声明使用了序列化协议
0005：STREAM_VERSION，序列化协议版本
73：TC_OBJECT，声明这是一个新的对象
72：TC_CLASSDESC，声明这里开始一个新Class描述
000f：Class名字的长度 15字节 
7465 7374 2e54 6573 744f 626a 6563 74：类名，test.TestObject
d3   c67e 1c4f 132a fe：SerialVersionUID, 序列化ID。Long.toHexString(-3186721026267206914L);如果没有指定,则会由算法随机生成一个8byte的ID
02：标记号，该值声明该对象支持序列化
0002：该类所包含的域个数
49：I 域的类型,int
00 09：域名字的长度
74 6573 7456 616c 7565：域名 testValue
4c：L，域的类型，代表一个class或者interface
000b：域名的长度15
69 6e6e 6572 4f62 6a65 6374 : 域名 innerObject
74:  标识 TC_STRING，标识后面的是个string值
0012 : 后面的string长度
4c 7465 7374 2f49 6e6e 6572 4f62 6a65 6374 3b: test/InnerObject;域类型的名
78:  块结束标记
72：TC_CLASSDESC，声明这里开始一个新Class描述
000b: Class名字的长度 11字节
74 6573 742e 5061 7265 6e74: 类名，test.Parent
bb1e ef0d 1fc9 50cd： SerialVersionUID（test.Parent）Long.toHexString(-4963266899668807475L)
02：标记号，该值声明该对象支持序列化
0001：该类所包含的域个数
49：I 域的类型,int
000b：域名字的长度，11字节
7061 7265 6e74 5661 6c75 65: 域名 parentValue
78: 块结束标记
70: 没有其他超类标记
0000 0064: 100 父类中域 parentValue的值
0000 012c: 300 自身的域 testValue值
73：TC_OBJECT，声明这是一个新的对象（ref）
72：TC_CLASSDESC，声明这里开始一个新Class描述
0010：Class名字的长度 16字节
7465 7374 2e49 6e6e 6572 4f62 6a65 6374：类名称 test.InnerObject
4f2c 148a 4024 fb12: SerialVersionUID（test.InnerObject）Long.toHexString(5704957411985783570L)
02：标记号，该值声明该对象支持序列化
0001：该类所包含的域个数
49：I 域的类型,int
000a：域名字的长度，10字节
69 6e6e 6572 5661 6c75 65: 域名 innerValue
78: 标志位:TC_ENDBLOCKDATA,对象的数据块描述的结束
70: 标志位:TC_NULL,Null object reference.
0000 00c8:  innervalue的值:200
```

##### 序列化过程:

```
1. 协议标记
2. 开始读目标类元数据信息：类名、字段
3. 递归从目标类开始向上，读取可以序列化的超类的元数据信息
4. 类递归结束，从超类开始，按照读取顺序写入 各个字段的值
5. 如果字段为引用（另一个类），则从1 开始重复，进行递归序列化
6. 协议结束标记
```
对于以上一个简单的java对象，通过其序列化之后的内容以及解析，我们可以知道java-build-in 在序列化一个对象时，存储空间效率不高的原因：
1.  会额外的添加很多标记性的字节
2.  没有采用压缩算法，直接进行16进制转化  
3.  数据存储采用固定大小的方式，会产生额外的空间开销  

#### 效率低下原因：  
1.  build-in 力求详尽描述一个对象的数据结构
2.  build-in 内部包含序列化类的校验
3.  为了追求通用性，内部用了大量的反射操作获取类相关信息

#### 反序列化  
java-build-in 的反序列化，也即是对上述序列化算法的逆向解析过程，根据设定的标识符解析。
重要的标记符为：  
- 0x70 - TC_NULL  
- 0x71 - TC_REFERENCE
- 0x72 - TC_CLASSDESC
- 0x73 - TC_OBJECT
- 0x74 - TC_STRING
- 0x75 - TC_ARRAY
- 0x76 - TC_CLASS
- 0x7B - TC_EXCEPTION
- 0x7C - TC_LONGSTRING
- 0x7D - TC_PROXYCLASSDESC
- 0x7E - TC_ENUM  
> 查看所有的标记定义，请阅读 **java.io.ObjectStreamConstants**类  

核心代码：
```
        try {
            switch (tc) {
                case TC_NULL:
                    return readNull();
                case TC_REFERENCE:
                    return readHandle(unshared);
                case TC_CLASS:
                    return readClass(unshared);
                case TC_CLASSDESC:
                case TC_PROXYCLASSDESC:
                    return readClassDesc(unshared);
                case TC_STRING:
                case TC_LONGSTRING:
                    return checkResolve(readString(unshared));
                case TC_ARRAY:
                    return checkResolve(readArray(unshared));
                case TC_ENUM:
                    return checkResolve(readEnum(unshared));
                case TC_OBJECT:
                    return checkResolve(readOrdinaryObject(unshared));
                case TC_EXCEPTION:
                    IOException ex = readFatalException();
                    throw new WriteAbortedException("writing aborted", ex);
                case TC_BLOCKDATA:
                case TC_BLOCKDATALONG:
                   ...
                case TC_ENDBLOCKDATA:
                    ...
                default:
                    throw new StreamCorruptedException(
                        String.format("invalid type code: %02X", tc));
            }
        }
```
根据上述的序列化之后的数据，结合代码：  
```
默认在读取一个ObjectInputStream对象时，会按照写入的顺序，预先读取出序列化标记和协议等字节接下来就是根据【73：TC_OBJECT】这个协议标记字节，执行readOrdinaryObject方法

private Object readOrdinaryObject(boolean unshared) throws IOException{
    ...
    ObjectStreamClass desc = readClassDesc(false);  //SectionA
    ...
    ...
    Object obj;
    try {
        obj = desc.isInstantiable() ? desc.newInstance() : null; //SectionB
    }
    ...
    readSerialData(obj, desc);//SectionC
    ...
    if (obj != null && handles.lookupException(passHandle) == null && desc.hasReadResolveMethod()){
            Object rep = desc.invokeReadResolve(obj);////SectionD
            if (unshared && rep.getClass().isArray()) {
                rep = cloneArray(rep);
            }
            if (rep != obj) {
                handles.setObject(passHandle, obj = rep);
            }
        }
    ...
    return obj;
}
```
结合代码段，我们按照section*顺序来分析：  
```
sectionA:
    [Func:] ObjectStreamClass desc = readClassDesc(false);
    读取并返回一个【class描述器】对象
    
    private ObjectStreamClass readClassDesc(boolean unshared) throws IOException{
            byte tc = bin.peekByte();
            switch (tc) {
            ...
                case TC_CLASSDESC:
                    return readNonProxyDesc(unshared);//case里面有其他判断，我们只考虑简单对象（非代理）的执行逻辑
            ...
            }
    }
    [Func:] readNonProxyDesc(unshared)：
    返回ObjectStreamClass
     private ObjectStreamClass readNonProxyDesc(boolean unshared) throws IOException{
        ...
        ObjectStreamClass desc = new ObjectStreamClass();
        ...
        ObjectStreamClass readDesc = null;
        try {
            readDesc = readClassDescriptor();//PRODUCE-1
        } ...
        Class<?> cl = null;
        
        bin.setBlockDataMode(true);
        final boolean checksRequired = isCustomSubclass();
        try {
            if ((cl = resolveClass(readDesc)) == null) {//PRODUCE-2
                resolveEx = new ClassNotFoundException("null class");
            } else if (checksRequired) {
                ReflectUtil.checkPackageAccess(cl);
            }
        } ...
        skipCustomData();
        desc.initNonProxy(readDesc, cl, resolveEx, readClassDesc(false));
        ...
        return desc;
    }
    [Func-PRODUCE-1] readClassDescriptor();
    读取一个非代理类型的class描述
    跟踪该方法会找到void readNonProxy(ObjectInputStream in)该方法
    void readNonProxy(ObjectInputStream in)
        throws IOException, ClassNotFoundException
    {
        name = in.readUTF();// 首先读长度，再接着根据长度读数据
        suid = Long.valueOf(in.readLong());
        isProxy = false;
        byte flags = in.readByte();
        ...
        int numFields = in.readShort();
       
        fields = (numFields > 0) ?
            new ObjectStreamField[numFields] : NO_FIELDS;
        for (int i = 0; i < numFields; i++) {
            char tcode = (char) in.readByte();
            String fname = in.readUTF();
            String signature = ((tcode == 'L') || (tcode == '[')) ?
                in.readTypeString() : new String(new char[] { tcode });
            try {
                fields[i] = new ObjectStreamField(fname, signature, false);
            } ...
        }
        computeFieldOffsets();
    }
    通过该方法可以清晰的了解到，按照我们写入的协议标记顺序，来读取出class的name以及序列ID，接着读取域的个数，按照个数循环读取每个域，并实例化成ObjectStreamField对象。需要注意的是，该方法会使ObjectInputStream流前进。
    [Func-PRODUCE-2] cl = resolveClass(readDesc)
    跟进该方法就会得到重点:
    return Class.forName(name, false, latestUserDefinedLoader());
    通过类加载器根据类名加载该类。（默认使用的是public 无参构造函数）
    接下来是desc.initNonProxy(readDesc, cl, resolveEx, readClassDesc(false));
    这个方法内部就包含了一个循环，可跟踪readClassDesc方法看见，将会循环读取，直至取出了所有的类描述器并初始化了类描述器，同时加载类描述器代表的类（并没有初始化）（此方法涉及缓存类描述器对象的技巧）。
    
SectionB:
    obj = desc.isInstantiable() ? desc.newInstance() : null; 
    创建反序列化对象并返回
    这里面涉及到使用类构造器对象来创建类对象。
SectionC:
    readSerialData(obj, desc);
    从超类开始，到子类；读取每个可序列化类的实例数据；这里面包含大量的反射操作。
SectionD:
    Object rep = desc.invokeReadResolve(obj);
    当类重写了writeObject和readObject方法时候，会调用此方法来进行实例的进一步加工。 
    

```



---
