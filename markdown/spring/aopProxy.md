# AOP 动态代理
---
## 一 JDK代理
DK动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理

    public class JDKProxy implements InvocationHandler {
    	//需要代理的目标对象
    	private Object targetObject;
    	// 构建代理类
    	private Object newProxy(Object targetObject) {
    		this.targetObject = targetObject;
    		return Proxy.newProxyInstance(targetObject.getClass().getClassLoader(), targetObject.getClass().getInterfaces(), this);
    	}
    	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    		//模拟检查权限
    		checkPopedom();
    		//使用反射 调用代理类
    		Object value = method.invoke(proxy, args);
    		return value;
    	}
    	private void checkPopedom() {
    		System.out.println(".:检查权限 checkPopedom()!");
    	}
    }
        
## 二 CGLIB代理(底层使用ASM实现)
动态生成一个要代理类的子类，子类重写要代理的类的所有不是final的方法。在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势织入横切逻辑。它比使用java反射的JDK动态代理要快，cglib底层使用字节码处理框架ASM，来转换字节码并生成新的类。不鼓励直接使用ASM，因为它要求你必须对JVM内部结构包括class文件的格式和指令集都很熟悉，由于cglib要生成子类，所以对final类是无法进行代理的

    public class CGLIBProxy implements MethodInterceptor {
        private Object targetObject;
        private Object createProxyObject(Object targetObject) {
            this.targetObject = targetObject;
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(targetObject.getClass());
            enhancer.setCallback(this);
            Object proxyObject = enhancer.create();
            return proxyObject;
        }
    
        public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
            Object object = null;
            if ("addUser".equals(method.getName())) {// 过滤方法
                checkPopedom();// 检查权限
            }
            object = method.invoke(targetObject, objects);
            return object;
        }
        private void checkPopedom() {
            System.out.println(".:检查权限 checkPopedom()!");
        }
    
    }
    
## 三 Spring proxy
### 3.1 代理方式选择规则(默认JDK Proxy)
|:-|
|(1)如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP|
|(2)如果目标对象实现了接口，可以强制使用CGLIB实现AOP|
|(3)如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换|
### 3.2 CGLIB代理方式配置
#### 3.2.1 xml
在xml中配置如下标签    
    <aop:aspectj-autoproxy proxy-target-class="true">
#### 3.2.1 springboot
在application.properties或者application.yml去设置如下属性

    // application.properties
    spring.aop.proxy-target-class=true
    
    // application.yml
    spring：
        aop：
            proxy-target-class：true
            
## 四 字节码增强
### 4.1 ASM
#### 4.1.1 使用
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

#### 4.1.2 优点
以二进制的形式修改已有类或动态生成类，小巧、灵活、性能好
#### 4.1.3 缺点
要很熟悉class文件结构及熟悉字节码指令，上手难度大


### 4.2 JAVASSIST 
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

#### 4.2.2 优点
使用java编码的方式动态改变类的结构，简单，不需要了解虚拟机指令
#### 4.2.3 缺点
不够灵活，性能比ASM差一些
           

