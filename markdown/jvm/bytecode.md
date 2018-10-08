# 字节码操作
---
## 一 JAVA AGENT
### 1.1介绍
在 JDK 1.5 中，Java 引入了 java.lang.Instrument 包，该包提供了一些工具帮助开发人员在 Java 程序运行时，动态修改系统中的 Class 类型。

### 1.2 premain
对于使用命令行接口的实现，可以将以下选项添加到命令行来启动代理 

    -javaagent:jarpath[=options]
JVM 首先尝试对代理类调用以下方法

    public static void premain(String agentArgs, Instrumentation inst)
如果不存在则尝试调用    

    public static void premain(String agentArgs)  
### 1.3 VM 启动后启动代理  
#### 1.3.1 代理 JAR 的清单必须包含属性 Agent-Class。此属性的值是代理类 的名称。
    Manifest-Version: 1.0
    Agent-Class:com.mumu.AgentDemo
    Created-By: Apache Maven 3.3.9
    Build-Jdk: 1.8.0_101
#### 1.3.2 代理类必须实现公共静态 agentmain 方法。
JVM 首先尝试对代理类调用以下方法

    public static void agentmain(String agentArgs, Instrumentation inst)
如果不存在则尝试调用    

    public static void agentmain(String agentArgs)  
### 1.4 自定义Agent(命令行启动方式)    
#### 1.4.1 实现公共静态premain方法
    public class AgentDemo {
        public static void premain(String agentArgs, Instrumentation inst) throws UnmodifiableClassException {
            inst.addTransformer(new ClassFileTransformer() {
                @Override
                public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
                    return getBytesByClassPath(Application.class.getSimpleName().concat(".class"));
                }
            });
            // todo 对字节码进行修改操作
            inst.retransformClasses(Application.class);
        }    
    }
#### 1.4.2 配置MAINIFEST.MF文件
生成的jar包中的META-INF/MAINIFEST.MF必须包含Premain-Class属性，如下

    Manifest-Version: 1.0
    Premain-Class:com.mumu.AgentDemo
    Created-By: Apache Maven 3.3.9
    Build-Jdk: 1.8.0_101
#### 1.4.3 使用
    -javaagent:/export/servers/mumu-agent-1.0.0.jar 
## 二 ASM
### 2.1 使用
生成一个类 包含打印hello word方法

    public class MumuClassAdapter extends ClassLoader  {
        public static void main (final String args[]) throws Exception {
    
            // 创建一个ClassWriter
            ClassWriter classWriter = new ClassWriter(ClassWriter.COMPUTE_MAXS);
            // 声明一个类 使用JDK 1.8/public/父类是Object/不实现任何接口
            classWriter.visit(Opcodes.V1_8, Opcodes.ACC_PUBLIC,"Mumu", null,"java/lang/Object",null);
    
            // 创建一个无参数的构造函数
            MethodVisitor constructor = classWriter.visitMethod(Opcodes.ACC_PUBLIC, "<init>", "()V", null, null);
            constructor.visitCode();
            constructor.visitVarInsn(Opcodes.ALOAD, 0);
            // 父类初始化
            constructor.visitMethodInsn(Opcodes.INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
            // 返回void
            constructor.visitInsn(Opcodes.RETURN);
            // 这段代码使用最多一个栈元素和一个本地变量
            constructor.visitMaxs(1, 1);
            constructor.visitEnd();
    
            // 创建一个输出 SAM hello world 方法
            MethodVisitor print = classWriter.visitMethod(Opcodes.ACC_PUBLIC, "run", "()V", null, null);
            print.visitCode();
            // 获取java.io.PrintStream
            print.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            print.visitLdcInsn("SAM hello world");
            print.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
            print.visitInsn(Opcodes.RETURN);
            print.visitMaxs(1, 1);
            print.visitEnd();
    
            // 生成字节码
             byte[] code = classWriter.toByteArray();
            // 实例化刚刚生成的类
            MumuClassAdapter loader = new MumuClassAdapter();
            Class mumu = loader.defineClass("Mumu", code, 0, code.length);
            // 调用run()方法
            Method method = mumu.getMethods()[0];
            method.invoke(mumu.newInstance(), null);
    
        }
    }
输出结果

    Connected to the target VM, address: '127.0.0.1:50290', transport: 'socket'
    SAM hello world
    Disconnected from the target VM, address: '127.0.0.1:50290', transport: 'socket'

### 2.2 优点
以二进制的形式修改已有类或动态生成类，小巧、灵活、性能好，
### 2.3 缺点
要很熟悉class文件结构及熟悉字节码指令，上手难度大

## 三 JAVASSIST 
### 3.1 使用
修改类 添加get方法
     
    public class People {
        private String name;
        private Integer age;
    }

    public static void main(String[] args) throws NotFoundException, CannotCompileException, IOException {
        Field[] fields = People.class.getDeclaredFields();
        StringBuilder method = new StringBuilder();
        ClassPool pool = ClassPool.getDefault();
        CtClass ctClazz = pool.get(People.class.getName());
        for (Field field : fields) {
            if (method.length() != 0) {
                method.setLength(0);
            }
            field.getType();
            method.append("public ")
                    .append(field.getType())
                    .append("get").append(field.getName().substring(0, 1).toUpperCase())
                    .append(field.getName().substring(1))
                    .append("() {")
                    .append(" return ")
                    .append(field.getName())
                    .append(";}").toString();
            ctClazz.addMethod(CtNewMethod.make(method.toString(), ctClazz));
        }
        ctClazz.toBytecode();
    }

### 3.2 优点
使用java编码的方式动态改变类的结构，简单，不需要了解虚拟机指令
### 3.3 缺点
不够灵活，性能比ASM差一些

           

