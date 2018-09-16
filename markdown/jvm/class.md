# Class类文件结构(GO语言描述)
---
## 一 Class文件
使用classpy-0.4.jar 打开文件如下图
![class](../../picture/jvm/class.PNG)
### 1.1 魔法数字
所有class文件的版本号都必须是0xCAFEBABE

    /**
     * 读取字节流文件且校验文件格式(magic number) class文件的magic number是0xCAFEBABE
     */
    func (self *ClassFile) readAndCheckMagic(reader *ClassReader) {
    	magic := reader.readUnit32()
    	if magic != 0xCAFEBABE {
    		panic("java.lang.ClassFormatError: magic!")
    	}	
    }

### 1.2 版本号
特定的Java虚拟机只支持版本号在某个范围内的class文件 oracle实现的是完全向后兼容的

    /**
     * 版本号校验 
     *     Java版本                     class文件版本号
     *     jdk1.0.2                     45.0 - 45.3
     *     jdk1.1                       45.0 - 45.65535
     *     J2SE1.2                      46.0
     *     J2SE1.3                      47.0
     *     J2SE1.4                      48.0
     *     JavaSE5.0                    49.0
     *     JavaSE6.0                    50.0
     *     JavaSE7.0                    51.0
     *     JavaSE8.0                    52.0
     */
    
    func (self *ClassFile) readAndCheckVersion(reader *ClassReader) {
        self.minorVersion = reader.readUnit16()
        self.majorVersion = reader.readUnit16()
        switch self.majorVersion {
        case 45:
            return
        case 46, 47, 48, 49, 50, 51, 52:
            if self.minorVersion == 0 {
                return
            }
        }
        panic("java.lang.UnsupportedClassVersionError!")
    }    

### 1.3 常量池
    
Java虚拟机规范定义了十四种常量
   
    //结构 
    cp_info {
      u1 tag;
       u1 info[];
    }
    
    const (        
        CONSTANT_Utf8 = 1
        CONSTANT_Integer = 3
        CONSTANT_Float = 4
        CONSTANT_Long = 5
        CONSTANT_Double = 6
        CONSTANT_Class = 7
        CONSTANT_String = 8
        CONSTANT_Fieldref = 9                     字段符号引用
        CONSTANT_Methodref = 10                   方法符号引用
        CONSTANT_InterfaceMethodref = 11          接口方法符号引用
        CONSTANT_NameAndType = 12
        CONSTANT_MethodHandle = 15
        CONSTANT_MethodType = 16
        CONSTANT_InvokeDynamic = 18
    )
    
### 1.4 访问标识(access_flags)用于识别一些类或者接口层面的信息
|标识名称|标志值|含义|针对对象|
|:-|:-|:-|:-|
|ACC_PUBLIC|0x0001|public类型|所有类型|
|ACC_FINAL|0x0010|final类型|类|
|ACC_SUPER|0x0020|使用新的invokespecial语义|类和接口|
|ACC_INTERFACE|0x0200|接口类型|接口|
|ACC_ABSTRACT|0x0400|抽象类型|类和接口|
|ACC_SYNTHETIC|0x1000|该类不由用户代码生成|所有类型|
|ACC_ANNOTATION |0x2000|注解类型|注解|
|ACC_ENUM  |0x4000|枚举类型|枚举|

### 1.5 其余部分
依次是 类索引、超类索引与接口索引 字段集合 方法集合 属性集合 

## 二 Class文文件加载过程

    /**
     * 1.校验魔法数字和版本号
     */
    func (self *ClassFile) read(reader *ClassReader) {
        //校验魔法数字
        self.readAndCheckMagic(reader)
        //校验版本号
        self.readAndCheckVersion(reader)
        //加载常量池
        self.constantPool = readConstantPool(reader)
        //加载访问标识
        self.accessFlags = reader.readUnit16();
        //加载当前类
        self.thisClass = reader.readUnit16();
        //加载父类信息
        self.superClass = reader.readUnit16();
        //加载接口
        self.interfaces = reader.readUnit16s();
        //加载字段表
        self.fields = readMembers(reader, self.constantPool)
        //加载方法表
        self.methods = readMembers(reader, self.constantPool)
        //加载属性信息
        self.attributes = readAttributes(reader, self.constantPool)
    }
    
    /**
     * 2.读取常量池
     */
    func readConstantPool(reader *ClassReader) ConstantPool {
        cpCount := int(reader.readUnit16())
        cp := make([]ConstantInfo, cpCount)
        //常量池的索引是从1开始的 
        for i :=1; i < cpCount; i++ {
            cp[i] = readConstantInfo(reader, cp)
            //Long和double占两个索引位
            switch cp[i].(type) {
            case *ConstantLongInfo, *ConstantDoubleInfo:
                i++
            }
        }
        return cp
    }
    
    
    /**
     * 3.读取常量池信息
     */
    func readConstantInfo(reader *ClassReader, cp ConstantPool) ConstantInfo {
        tag := reader.readUnit8()
        c := newConstantInfo(tag, cp)
        c.readInfo(reader)
        return c
    }
    
    
    /**
     * 4.根据访问标识 构建对应的常量池信息
     */
    func newConstantInfo(tag uint8, cp ConstantPool) ConstantInfo {
        switch tag {
        case CONSTANT_Integer:
            return &ConstantIntegerInfo{}
        case CONSTANT_Float:
            return &ConstantFloatInfo{}
        case CONSTANT_Long:
            return &ConstantLongInfo{}
        case CONSTANT_Double:
            return &ConstantDoubleInfo{}
        case CONSTANT_Utf8:
            return &ConstantUtf8Info{}
        case CONSTANT_String:
            return &ConstantStringInfo{cp: cp}
        case CONSTANT_Class:
            return &ConstantClassInfo{cp: cp}
        case CONSTANT_Fieldref:
            return &ConstantFieldrefInfo{ConstantMemberrefInfo{cp: cp}}
        case CONSTANT_Methodref:
            return &ConstantMethodrefInfo{ConstantMemberrefInfo{cp: cp}}
        case CONSTANT_InterfaceMethodref:
            return &ConstantInterfaceMethodrefInfo{ConstantMemberrefInfo{cp: cp}}
        case CONSTANT_NameAndType:
            return &ConstantNameAndTypeInfo{}
        case CONSTANT_MethodType:
            return &ConstantMethodTypeInfo{}
        case CONSTANT_MethodHandle:
            return &ConstantMethodHandleInfo{}
        case CONSTANT_InvokeDynamic:
            return &ConstantInvokeDynamicInfo{}
        default:
            panic("java.lang.ClassFormatError: constant pool tag!")
        }
    }
    
    /**
     * 5.常量池中读取字段表或方法表
     */
    func readMembers(reader *ClassReader, cp ConstantPool) []*MemberInfo {
        memberCount := reader.readUnit16()
        members := make([]*MemberInfo, memberCount)
        for i := range members {
            members[i] = readMember(reader, cp)
        }
        return members
    }
    
    /**
     * 6.读取字段或方法数据
     */
    func readMember(reader *ClassReader, cp ConstantPool) *MemberInfo {
        return &MemberInfo {
            cp: cp,
            accessFlags: reader.readUnit16(),
            nameIndex: reader.readUnit16(),
            descriptorIndex: reader.readUnit16(),
            attributes: readAttributes(reader, cp), 
        }
    }
    
    /**
     * 7.readAttributes方法读取属性表
     */
    func readAttributes(reader *ClassReader, cp ConstantPool) []AttributeInfo {
        attributesCount := reader.readUnit16()
        attributes := make([]AttributeInfo, attributesCount)
        for i := range attributes {
            attributes[i] = readAttribute(reader, cp)
        }
        return attributes
    }
    
    /**
     * 8.readAttribute方法读取单个属性
     */
    func readAttribute(reader *ClassReader, cp ConstantPool) AttributeInfo {
        attrNameIndex := reader.readUnit16()
        attrName := cp.getUtf8(attrNameIndex)
        attrLen := reader.readUnit32()
        attrInfo := newAttributeInfo(attrName, attrLen, cp)
        attrInfo.readInfo(reader)
        return attrInfo
    }
    
    /**
     * 9.创建具体的属性实例java虚拟机中定义了23种属性， 先解析其中的八种
     * 23种预定义的属性可以分成三组 
     * 1.Java虚拟机必需的 5种
     * 2.java类库所必需的 12种
     * 3.提供给工具使用的 6种 (也就是说第三种属性是可选的 比如LineNumberTableshuxin)
     */
    func newAttributeInfo (attrName string, attrLen uint32, cp ConstantPool) AttributeInfo {
        switch attrName {
        case "Code": return &CodeAttribute{cp: cp} 
        case "ConstantValue": return &ConstantValueAttribute{}
        case "Deprecated": return &DeprecatedAttribute{}
        case "Exceptions": return &ExceptionsAttribute{}
        case "LineNumberTable": return &LineNumberTableAttribute{}
        case "LocalVariableTable": return &LocalVariableTableAttribute{}
        case "SourceFile": return &SourceFileAttribute{}
        case "Synthetic": return &SyntheticAttribute{}
        default: return &UnparsedAttribute{attrName, attrLen, nil}
        }
    }




    

