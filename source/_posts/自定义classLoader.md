---
title: 自定义classLoader
date: 2017-10-19 21:31:11
tags:
	- java
	- classloader
---



在`java`中，加载一个类到`jvm`虚拟机并为其实例化一个对象是由`ClassLoader`实现的
有时候，我们需要实现特殊的类加载方式，就需要自己实现一个`ClassLoader `
`ClassLoader`中有几个比较重要的方法：

- `loadClass`：
  该方法实现了双亲委托机制，一般不会重写该方法，他的执行步骤是先委托父加载器加载，如果加载不到，在执行自己的`findClass `加载
- `findClass`：
  一般实现自己的`ClassLoader`都是重写该方法，如果方法找不着，则抛出一个`ClassNotFound`异常
- `defineClass`：
  该方法是`jvm`提供的一个接口，验证一个`class`字节码数组，并为其创建一个`Class`对象
>下面简单实现一个`ClassLoader`，用于加载指定目录下的`jar`包内的类
```java
public class JarsClassLoader extends ClassLoader {
    private String basePath;  //jar包存放的目录

    public JarsClassLoader(String path, ClassLoader parentClasss) {
        super(parentClasss); //指定父加载器
        basePath = path;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classByte = getClassByte(name); //根据类的全路径名去获取一个字节码数组
        try {
            return defineClass(name, classByte, 0, classByte.length);//生成一个class
        } catch (Exception e) {
            throw new ClassNotFoundException("找不到类："+name);
        }
    }
    /**
     * 该方法遍历basePath下的jar包，查找是否存在指定的类文件
    **/
    private byte[] getClassByte(String className) {
        try {
            File baseDir = new File(basePath); 
            File[] childrens = baseDir.listFiles();
            for (File child : childrens) {
                if (!child.getName().endsWith(".jar"))
                    continue;
                //jar包的全路径名
                String jarName = basePath + File.separator + child.getName();
                //创建一个Zip文件对象
                ZipFile zip = new ZipFile(jarName);
                //遍历Zip内的所有实体项
                Enumeration<ZipEntry> zipEntries = (Enumeration<ZipEntry>) zip.entries();
                ZipEntry zn;
                while (zipEntries.hasMoreElements()) {
                    zn = zipEntries.nextElement();
                    if (!zn.getName().endsWith(".class"))
                        continue;
                    //处理实体名，将`/`替换成`.`，并去除`.class`后缀
                    String znName =
                            zn.getName().replace("/", ".").replace(".class", "");
                    //如果找到指定的类文件
                    if (znName.equals(className)) {
                        BufferedInputStream bis = new BufferedInputStream(zip.getInputStream(zn));
                        ByteArrayOutputStream bos = new ByteArrayOutputStream((int) zn.getSize());
                        byte[] bytes = new byte[1024];
                        int cnt;
                        while ((cnt = (bis.read(bytes, 0, bytes.length))) > 0) {
                             bos.write(bytes, 0, cnt);
                        }
                        return bos.toByteArray();
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } 
        return null;
    }
}
```