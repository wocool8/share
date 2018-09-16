# ClassLoader
---
## 一 类加载器
![class](../../picture/jvm/classLoader.jpg)
## 二 类加载过程
## 2.1 加载过程
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
## 2.2 类加载器双亲委派代码
    protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        Class c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {

            }
            if (c == null) {
                c = findClass(name);
            }
            // c是ClassFile
        }
        // 
        if (resolve) {
            resolveClass(c);
            //c是ClassReader
        }
        return c;
    }