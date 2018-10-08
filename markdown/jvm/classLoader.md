# ClassLoader
---
## 一 加载顺序
![class](../../picture/jvm/classLoader.jpg)
## 二 加载过程(go语言描述)
    /**
     * 类加载器 加载流程
     * 如果用户没有提供 -classpath/-cp选项，则使用当前的目录作为用户路径
     */
    func Parse(jreOption, cpOption string) *Classpath {
        cp := &Classpath{}
        cp.parseBootAndExtClasspath(jreOption)
        cp.parseUserClasspath(cpOption)
        return cp
    }
    
    /**
     * 加载bootClasspath和extClasspath
     */
    func (self *Classpath) parseBootAndExtClasspath(jreOption string) {
    	jreDir := getJreDir(jreOption)
    	// jre/lib/*
    	jreLibPath := filepath.Join(jreDir, "lib", "*")
    	self.bootClasspath = newWildcardEntry(jreLibPath)
    	// jre/lib/ext/*
    	jreExtPath := filepath.Join(jreDir, "lib", "ext", "*")
    	self.extClasspath = newWildcardEntry(jreExtPath)
    }
    
    /**
     * 加载 userClasspath
     */
    func (self *Classpath) parseUserClasspath(cpOption string) {
    	if cpOption == "" {
    		cpOption = "."
    	}
    	self.userClasspath = newEntry(cpOption)
    }
## 三 ClassLoader
### 3.1 loadClass方法
JVM在加载类时默认采用的是双亲委派机制。通俗的讲，就是某个特定的类加载器在接到加载类的请求时，首先将加载任务委托给父类加载器，依次递归，如果父类加载器可以完成类加载任务，就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载，
双亲委派机制的根本原因是loadClass方法的 c = parent.loadClass(name, false);

    protected Class<?> loadClass(String name, boolean resolve)
            throws ClassNotFoundException
        {
            synchronized (getClassLoadingLock(name)) {
                // First, check if the class has already been loaded
                Class<?> c = findLoadedClass(name);
                if (c == null) {
                    long t0 = System.nanoTime();
                    try {
                        if (parent != null) {
                            c = parent.loadClass(name, false);
                        } else {
                            c = findBootstrapClassOrNull(name);
                        }
                    } catch (ClassNotFoundException e) {
                        // ClassNotFoundException thrown if class not found
                        // from the non-null parent class loader
                    }
    
                    if (c == null) {
                        // If still not found, then invoke findClass in order
                        // to find the class.
                        long t1 = System.nanoTime();
                        c = findClass(name);
    
                        // this is the defining class loader; record the stats
                        sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                        sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                        sun.misc.PerfCounter.getFindClasses().increment();
                    }
                }
                if (resolve) {
                    resolveClass(c);
                }
                return c;
            }
        }
    }
### 3.2 defineClass方法
将字节数组转化为类实例

    /**
     * Converts an array of bytes into an instance of class <tt>Class</tt>.
     */
    protected final Class<?> defineClass(String name, byte[] b, int off, int len)
        throws ClassFormatError
    {
        return defineClass(name, b, off, len, null);
    }

### 3.3 findClass方法
自定义类加载机制可以通过重写findClass方法加载自定义路径类

    /**
     * Finds the class with the specified <a href="#name">binary name</a>.
     * This method should be overridden by class loader implementations that
     * follow the delegation model for loading classes, and will be invoked by
     * the {@link #loadClass <tt>loadClass</tt>} method after checking the
     * parent class loader for the requested class.  The default implementation
     * throws a <tt>ClassNotFoundException</tt>.
     */
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }


## 四 Custom ClassLoader
### 4.1 实现ClassLoader(自定义不打破双亲委派机制的类加载器，只重写findClass方法)

    public class MumuClassLoader extends ClassLoader {
    
        @Override
        protected Class<?> findClass(String name) throws ClassNotFoundException {
            byte [] byteCode = getBytes(name);
            if(null != byteCode){
                //字节码转化为类
                return defineClass(name, byteCode,0, byteCode.length);
            }
            return super.findClass(name);
        }
    }