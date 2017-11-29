# TargetSdkVersion的实现原理
   
   在Android新版本实现某些功能时,如果需要兼容以前版本,获取TargetSdkVersion,然后根据该值来觉得是否使用新的方案,还是旧的实现方案.

# 集成类,然后重写私有方法

```
public class RePluginClassLoader extends PathClassLoader{

    。。。。

    public RePluginClassLoader(ClassLoader parent, ClassLoader orig) {

        // 由于PathClassLoader在初始化时会做一些Dir的处理，所以这里必须要传一些内容进来
        // 但我们最终不用它，而是拷贝所有的Fields
        super("", "", parent);
        mOrig = orig;

        // 将原来宿主里的关键字段，拷贝到这个对象上，这样骗系统以为用的还是以前的东西（尤其是DexPathList）
        // 注意，这里用的是“浅拷贝”
        // Added by Jiongxuan Zhang
        copyFromOriginal(orig);
        //反射获取原ClassLoader中的重要方法用来重写这些方法
        initMethods(orig);
    }

    //反射获取原ClassLoader中的方法
     private void initMethods(ClassLoader cl) {
        Class<?> c = cl.getClass();
        findResourceMethod = ReflectUtils.getMethod(c, "findResource", String.class);
        findResourceMethod.setAccessible(true);
        findResourcesMethod = ReflectUtils.getMethod(c, "findResources", String.class);
        findResourcesMethod.setAccessible(true);
        findLibraryMethod = ReflectUtils.getMethod(c, "findLibrary", String.class);
        findLibraryMethod.setAccessible(true);
        getPackageMethod = ReflectUtils.getMethod(c, "getPackage", String.class);
        getPackageMethod.setAccessible(true);
    }

     //拷贝原ClassLoader中的字段到本对象中
     private void copyFromOriginal(ClassLoader orig) {
        if (LOG && IPC.isPersistentProcess()) {
            LogDebug.d(TAG, "copyFromOriginal: Fields=" + StringUtils.toStringWithLines(ReflectUtils.getAllFieldsList(orig.getClass())));
        }

        if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.GINGERBREAD_MR1) {
            // Android 2.2 - 2.3.7，有一堆字段，需要逐一复制
            // 以下方法在较慢的手机上用时：8ms左右
            copyFieldValue("libPath", orig);
            copyFieldValue("libraryPathElements", orig);
            copyFieldValue("mDexs", orig);
            copyFieldValue("mFiles", orig);
            copyFieldValue("mPaths", orig);
            copyFieldValue("mZips", orig);
        } else {
            // Android 4.0以上只需要复制pathList即可
            // 以下方法在较慢的手机上用时：1ms
            copyFieldValue("pathList", orig);
        }
    }

    //重写了ClassLoader的loadClass
    @Override
    protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {

        Class<?> c = null;

        //拦截类的加载过程，判断要加载的类是否存在对应的插件信息，如果有从插件中加载
        c = PMF.loadClass(className, resolve);

        if (c != null) {
            return c;
        }

        try {
            //如果没有在插件中找到该类，使用宿主原来的ClassLoader加载
            c = mOrig.loadClass(className);

            return c;
        } catch (Throwable e) {

        }

        return super.loadClass(className, resolve);
    }


    //重写反射的方法，执行的是原ClassLoader的方法
    @Override
    protected URL findResource(String resName) {
        try {
            return (URL) findResourceMethod.invoke(mOrig, resName);
        } catch (IllegalArgumentException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
        return super.findResource(resName);
    }

    //省略反射重写的其他方法，都是一样的
    。。。。



}



```


# hook 系统的classloader
- 建立自己的classloader
- 设置线程classloader
- 设置包的初始classloader

```
public static boolean patch(Application application) {
    try {
        // 获取Application的BaseContext
        // 该BaseContext在不同版本中具体的实例不同
        // 1. ApplicationContext - Android 2.1
        // 2. ContextImpl - Android 2.2 and higher
        // 3. AppContextImpl - Android 2.2 and higher

        Context oBase = application.getBaseContext();
        if (oBase == null) {

            return false;
        }

        // 获取mBase.mPackageInfo
        // mPackageInfo的类型主要有两种：mPackageInfo这个对象代表了apk文件在内存中的表现
        // 1. android.app.ActivityThread$PackageInfo - Android 2.1 - 2.3
        // 2. android.app.LoadedApk - Android 2.3.3 and higher
        Object oPackageInfo = ReflectUtils.readField(oBase, "mPackageInfo");
        if (oPackageInfo == null) {

            return false;
        }


       // 获取mPackageInfo.mClassLoader，也就是宿主的PathClassLoader对象
        ClassLoader oClassLoader = (ClassLoader) ReflectUtils.readField(oPackageInfo, "mClassLoader");
        if (oClassLoader == null) {
            if (LOGR) {
                LogRelease.e(PLUGIN_TAG, "pclu.p: nf mpi. mb cl=" + oBase.getClass() + "; mpi cl=" + oPackageInfo.getClass());
            }
            return false;
        }

        // 从RePluginCallbacks中获取RePluginClassLoader，通过宿主的父ClassLoader和宿主ClassLoader生成RePluginClassLoader
        ClassLoader cl = RePlugin.getConfig().getCallbacks().createClassLoader(oClassLoader.getParent(), oClassLoader);

        // 将我们创建的RePluginClassLoader赋值给mPackageInfo.mClassLoader ，来达到代理系统的PathClassLoader
        ReflectUtils.writeField(oPackageInfo, "mClassLoader", cl);

        // 设置线程上下文中的ClassLoader为RePluginClassLoader
        // 防止在个别Java库用到了Thread.currentThread().getContextClassLoader()时，“用了原来的PathClassLoader”，或为空指针
        Thread.currentThread().setContextClassLoader(cl);

    } catch (Throwable e) {
        e.printStackTrace();
        return false;
    }
    return true;
}

```
