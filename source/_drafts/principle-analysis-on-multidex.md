---
title: Android MultiDex实现原理解析
date: 2016-11-17 19:20:10
tags: 
- android
- multidex
- 源码分析 
---

![](http://ww2.sinaimg.cn/large/65e4f1e6gw1fa0zxgti6jj20hs0bv405.jpg)

本文主要从源码角度出发，分析MultiDex的实现原理。

<!-- more -->

## 分析

调用MultiDex的方式有多种，不论是直接使用官方提供的`MultiDexApplication`，还是继承`MultiDexApplication`，或者是重写自定义Application的`attachBaseContext`方法，最后都会调用到`MultiDex.install(this);`：

```java
  @Override
  protected void attachBaseContext(Context base) {
    super.attachBaseContext(base);
    MultiDex.install(this);
  }
```

MultiDex.install是整个MultiDex的入口点，我们以此为切入点开始分析：

```java
   public static void install(Context context) {
        Log.i(TAG, "install");

        // 检查当前系统是否支持multidex
        if (IS_VM_MULTIDEX_CAPABLE) {
            Log.i(TAG, "VM has multidex support, MultiDex support library is disabled.");
            try {
                clearOldDexDir(context);
            } catch (Throwable t) {
                Log.w(TAG, "Something went wrong when trying to clear old MultiDex extraction, "
                        + "continuing without cleaning.", t);
            }
            return;
        }

        // MultiDex最低只支持到1.6
        if (Build.VERSION.SDK_INT < MIN_SDK_VERSION) {
            throw new RuntimeException("Multi dex installation failed. SDK " + Build.VERSION.SDK_INT
                    + " is unsupported. Min SDK version is " + MIN_SDK_VERSION + ".");
        }

        try {
            ApplicationInfo applicationInfo = getApplicationInfo(context);
            if (applicationInfo == null) {
                // Looks like running on a test Context, so just return without patching.
                return;
            }

            synchronized (installedApk) {

                // sourceDir对应于/data/app/<package-name>.apk
                String apkPath = applicationInfo.sourceDir;

                // 若给定apk已经install过，直接退出
                if (installedApk.contains(apkPath)) {
                    return;
                }
                installedApk.add(apkPath);


                // MultiDex 最高只支持到20(Android 4.4W)，更高的版本不能保证正常工作

                if (Build.VERSION.SDK_INT > MAX_SUPPORTED_SDK_VERSION) {
                    Log.w(TAG, "MultiDex is not guaranteed to work in SDK version "
                            + Build.VERSION.SDK_INT + ": SDK version higher than "
                            + MAX_SUPPORTED_SDK_VERSION + " should be backed by "
                            + "runtime with built-in multidex capabilty but it's not the "
                            + "case here: java.vm.version=\""
                            + System.getProperty("java.vm.version") + "\"");
                }

                /* 
                 * 待Patch的class loader应该是BaseDexClassLoaderd的子类，
                 * MultiDex主要通过修改pathList字段来添加更多的dex
                 */
                ClassLoader loader;
                try {
                    loader = context.getClassLoader();
                } catch (RuntimeException e) {
                    /* Ignore those exceptions so that we don't break tests relying on Context like
                     * a android.test.mock.MockContext or a android.content.ContextWrapper with a
                     * null base Context.
                     */
                    Log.w(TAG, "Failure while trying to obtain Context class loader. " +
                            "Must be running in test mode. Skip patching.", e);
                    return;
                }
                if (loader == null) {
                    // Note, the context class loader is null when running Robolectric tests.
                    Log.e(TAG,
                            "Context class loader is null. Must be running in test mode. "
                            + "Skip patching.");
                    return;
                }


                // MultiDex的二级dex文件将存放在 /data/data/<package-name>/secondary-dexes 下

                File dexDir = new File(context.getFilesDir(), SECONDARY_FOLDER_NAME);

                // 从apk中查找并解压二级dex文件到/data/data/<package-name>/secondary-dexes
                List<File> files = MultiDexExtractor.load(context, applicationInfo, dexDir, false);

                // 检查dex压缩文件的完整性
                if (checkValidZipFiles(files)) {

                    // 开始安装dex
                    installSecondaryDexes(loader, dexDir, files);
                } else {
                    Log.w(TAG, "Files were not valid zip files.  Forcing a reload.");

                    // 第一次检查失败，MultiDex会尽责的再检查一次
                    files = MultiDexExtractor.load(context, applicationInfo, dexDir, true);

                    if (checkValidZipFiles(files)) {

                        // 开始安装dex
                        installSecondaryDexes(loader, dexDir, files);
                    } else {
                        // Second time didn't work, give up
                        throw new RuntimeException("Zip files were not valid.");
                    }
                }
            }

        } catch (Exception e) {
            Log.e(TAG, "Multidex installation failure", e);
            throw new RuntimeException("Multi dex installation failed (" + e.getMessage() + ").");
        }
        Log.i(TAG, "install done");
    }
```

这个方法涵盖了MultiDex安装的整个流程：

**1. 检查虚拟机版本判断是否需要MultiDex;**

在ART虚拟机中（部分4.4机器及5.0以上的机器），采用了Ahead-of-time（AOT）compilation技术，系统在apk的安装过程中，会使用自带的dex2oat工具对apk中可用的dex文件进行编译，并生成一个可在本地机器上运行的odex(optimized dex)文件，这样做会提高应用的启动速度。（但是安装速度降低了）

若不需要使用MultiDex，将使用clearOldDexDir清除/data/data/pkgName/code-cache/secondary-dexes目录下下所有文件

**2. 根据applicationInfo.sourceDir的值获取安装的apk路径**

安装完成的apk路径为`/data/app/<package-name>.apk`

**3. 检查apk是否执行过MultiDex.install，若已经安装直接退出**

**4. 使用MultiDexExtractor.load获取apk中可用的二级dex列表**

```java
static List<File> load(Context context, ApplicationInfo applicationInfo, File dexDir, boolean forceReload) throws IOException {

    Log.i("MultiDex", "MultiDexExtractor.load(" + applicationInfo.sourceDir + ", " + forceReload + ")");
    File sourceApk = new File(applicationInfo.sourceDir);
    long currentCrc = getZipCrc(sourceApk);
    List files;
    if(!forceReload && !isModified(context, sourceApk, currentCrc)) {
        try {
            files = loadExistingExtractions(context, sourceApk, dexDir);
        } catch (IOException var9) {
            Log.w("MultiDex", "Failed to reload existing extracted secondary dex files, falling back to fresh extraction", var9);
            files = performExtractions(sourceApk, dexDir);
            putStoredApkInfo(context, getTimeStamp(sourceApk), currentCrc, files.size() + 1);
        }
    } else {
        Log.i("MultiDex", "Detected that extraction must be performed.");
        files = performExtractions(sourceApk, dexDir);
        putStoredApkInfo(context, getTimeStamp(sourceApk), currentCrc, files.size() + 1);
    }

    Log.i("MultiDex", "load found " + files.size() + " secondary dex files");
    return files;
}
```

MultiDexExtractor.load会先判断是否需要从apk中解压dex文件，主要判断依据是：上次保存的apk（zip文件）的CRC校验码和last modify日期与dex的总数量是否与当前apk相同。此外，forceReload也会决定是否需要重新解压，这个参数后文会提到。

如果需要解压dex文件，将会使用performExtractions将.dex从apk中解压出来，解压路径为 
```bash
/data/data/<package-name>/code_cache/secondary-dexes/<package-name>.apk.classes2.zip
/data/data/<package-name>/code_cache/secondary-dexes/<package-name>.apk.classes3.zip 
...
```

```java
private static List<File> performExtractions(File sourceApk, File dexDir) throws IOException {

        // extractedFilePrefix值为<package-name>.apk.classes

        String extractedFilePrefix = sourceApk.getName() + ".classes";
        prepareDexDir(dexDir, extractedFilePrefix);
        ArrayList files = new ArrayList();
        ZipFile apk = new ZipFile(sourceApk);

        try {
            int e = 2;

            // 扫描apk内所有classes2.dex、classes3.dex...文件
            for(ZipEntry dexFile = apk.getEntry("classes" + e + ".dex"); dexFile != null; dexFile = apk.getEntry("classes" + e + ".dex")) {

                // 解压路径为 /data/data/<package-name>/secondary-dexes/<package-name>.classes2.dex.zip 、/data/data/<package-name>/secondary-dexes/<package-name>.classes3.dex.zip ...
                String fileName = extractedFilePrefix + e + ".zip";

                File extractedFile = new File(dexDir, fileName);
                files.add(extractedFile);

                Log.i("MultiDex", "Extraction is needed for file " + extractedFile);
                
                int numAttempts = 0;
                boolean isExtractionSuccessful = false;

                // 每个dex文件都会尝试3次解压
                while(numAttempts < 3 && !isExtractionSuccessful) {
                    ++numAttempts;
                    extract(apk, dexFile, extractedFile, extractedFilePrefix);
                    isExtractionSuccessful = verifyZipFile(extractedFile);
                    Log.i("MultiDex", "Extraction " + (isExtractionSuccessful?"success":"failed") + " - length " + extractedFile.getAbsolutePath() + ": " + extractedFile.length());
                    if(!isExtractionSuccessful) {
                        extractedFile.delete();
                        if(extractedFile.exists()) {
                            Log.w("MultiDex", "Failed to delete corrupted secondary dex \'" + extractedFile.getPath() + "\'");
                        }
                    }
                }

                if(!isExtractionSuccessful) {
                    throw new IOException("Could not create zip file " + extractedFile.getAbsolutePath() + " for secondary dex (" + e + ")");
                }

                ++e;
            }
        } finally {
            try {
                apk.close();
            } catch (IOException var16) {
                Log.w("MultiDex", "Failed to close resource", var16);
            }

        }

        return files;
    }
```

解压成功后，会保存本次解压所使用的apk信息，用于下次调用MultiDexExtractor.load时判断是否需要重新解压：

```java
private static void putStoredApkInfo(Context context, long timeStamp, long crc, int totalDexNumber) {
    SharedPreferences prefs = getMultiDexPreferences(context);
    Editor edit = prefs.edit();

    // apk最后修改时间戳
    edit.putLong("timestamp", timeStamp);

    // apk的CRC校验码
    edit.putLong("crc", crc);

    // dex的总数量
    edit.putInt("dex.number", totalDexNumber);
    apply(edit);
}
```

如果apk未被修改，将会调用loadExistingExtractions方法，直接加载上一次解压出来的文件：
```java
private static List<File> loadExistingExtractions(Context context, File sourceApk, File dexDir) throws IOException {
        Log.i("MultiDex", "loading existing secondary dex files");
        String extractedFilePrefix = sourceApk.getName() + ".classes";
        int totalDexNumber = getMultiDexPreferences(context).getInt("dex.number", 1);
        ArrayList files = new ArrayList(totalDexNumber);

        for(int secondaryNumber = 2; secondaryNumber <= totalDexNumber; ++secondaryNumber) {
            String fileName = extractedFilePrefix + secondaryNumber + ".zip";
            File extractedFile = new File(dexDir, fileName);
            if(!extractedFile.isFile()) {
                throw new IOException("Missing extracted secondary dex file \'" + extractedFile.getPath() + "\'");
            }

            files.add(extractedFile);
            if(!verifyZipFile(extractedFile)) {
                Log.i("MultiDex", "Invalid zip file: " + extractedFile);
                throw new IOException("Invalid ZIP file.");
            }
        }

        return files;
}
```

不管是调用了loadExistingExtractions还是performExtractions，都会返回一个解压后的<package-name>.apk.classes2.zip、<package-name>.apk.classes3.zip...File列表，供下一步使用。

### 5. 两次校验dex压缩包的完整性
通过上一步得到解压后的dex File列表后，在MultiDex中会两次检查zip文件的完整性：

```java
   public static void install(Context context) {

        ...

        try {
            ...

            synchronized (installedApk) {

                ...

                // 从apk中查找并解压二级dex文件到/data/data/<package-name>/secondary-dexes
                List<File> files = MultiDexExtractor.load(context, applicationInfo, dexDir, false);

                // 检查dex压缩文件的完整性
                if (checkValidZipFiles(files)) {

                    // 开始安装dex
                    installSecondaryDexes(loader, dexDir, files);
                } else {
                    Log.w(TAG, "Files were not valid zip files.  Forcing a reload.");

                    // 第一次检查失败，MultiDex会尽责的再检查一次
                    files = MultiDexExtractor.load(context, applicationInfo, dexDir, true);

                    if (checkValidZipFiles(files)) {

                        // 开始安装dex
                        installSecondaryDexes(loader, dexDir, files);
                    } else {
                        // Second time didn't work, give up
                        throw new RuntimeException("Zip files were not valid.");
                    }
                }
            }

        } catch (Exception e) {
            Log.e(TAG, "Multidex installation failure", e);
            throw new RuntimeException("Multi dex installation failed (" + e.getMessage() + ").");
        }
        Log.i(TAG, "install done");
    }
```

若第一次校验失败（dex文件损坏等），MultiDex会重新调用MultiDexExtractor.load方法重查找加载二级dex文件列表，*值得注意的是此时forceReload的值为true，会强制重新从apk中解压dex文件。*

### 6. 开始dex的安装
经过上面的重重检验和解压，终于到了最关键的一步：将二级dex添加到我们classLoader中
```java
    private static void installSecondaryDexes(ClassLoader loader, File dexDir, List<File> files) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException, IOException {
        if(!files.isEmpty()) {
            if(VERSION.SDK_INT >= 19) {
                MultiDex.V19.install(loader, files, dexDir);
            } else if(VERSION.SDK_INT >= 14) {
                MultiDex.V14.install(loader, files, dexDir);
            } else {
                MultiDex.V4.install(loader, files);
            }
        }

    }
```

由于SDK版本不同，ClassLoader中的实现存在差异，所以使用了三个分支去执行dex的安装。这里我们选择MultiDex.V14.install进行分析，其他两个大同小异：

先明确入参：

| 入参 | 含义 | 
| ------| ------ |
| ClassLoader loader | 通过context.getClassLoader获取到的默认类加载器 | 
| List<File> additionalClassPathEntries | 二级dex文件解压后的路径(通过步骤4获得) |
| optimizedDirectory    |   对应/data/data/<package-name>/code_cache/secondary-dexes/目录   |


```java
private static Field findField(Object instance, String name) throws NoSuchFieldException {
    Class clazz = instance.getClass();

    while(clazz != null) {
        try {
            Field e = clazz.getDeclaredField(name);
            if(!e.isAccessible()) {
                e.setAccessible(true);
            }

            return e;
        } catch (NoSuchFieldException var4) {
            clazz = clazz.getSuperclass();
        }
    }

    throw new NoSuchFieldException("Field " + name + " not found in " + instance.getClass());
}

private static Method findMethod(Object instance, String name, Class... parameterTypes) throws NoSuchMethodException {
    Class clazz = instance.getClass();

    while(clazz != null) {
        try {
            Method e = clazz.getDeclaredMethod(name, parameterTypes);
            if(!e.isAccessible()) {
                e.setAccessible(true);
            }

            return e;
        } catch (NoSuchMethodException var5) {
            clazz = clazz.getSuperclass();
        }
    }

    throw new NoSuchMethodException("Method " + name + " with parameters " + Arrays.asList(parameterTypes) + " not found in " + instance.getClass());
}


private static final class V14 {
        private V14() {
        }

        private static void install(ClassLoader loader, List<File> additionalClassPathEntries, File optimizedDirectory) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException {

            // 通过反射获取 ClassLoader中的pathList
            Field pathListField = MultiDex.findField(loader, "pathList");
            Object dexPathList = pathListField.get(loader);

            // 先调用pathList的makeDexElements，然后将生成的Element[]传入expandFieldArray中
            MultiDex.expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList, new ArrayList(additionalClassPathEntries), optimizedDirectory));
        }

        private static Object[] makeDexElements(Object dexPathList, ArrayList<File> files, File optimizedDirectory) throws IllegalAccessException, InvocationTargetException, NoSuchMethodException {

            Method makeDexElements = MultiDex.findMethod(dexPathList, "makeDexElements", new Class[]{ArrayList.class, File.class});
            return (Object[])((Object[])makeDexElements.invoke(dexPathList, new Object[]{files, optimizedDirectory}));
        }
}

```

/libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java
```java
public class BaseDexClassLoader extends ClassLoader {
    ...

    /** structured lists of path elements */
    private final DexPathList pathList;

    ...

    public BaseDexClassLoader(String dexPath, File optimizedDirectory, String libraryPath, ClassLoader parent) {
        super(parent);

        this.originalPath = dexPath;
        this.originalLibraryPath = libraryPath;
        this.pathList =
            new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
}
```

/libcore/dalvik/src/main/java/dalvik/system/DexPathList.java
```java
/*package*/ final class DexPathList {
    ....

    /**
     * Makes an array of dex/resource path elements, one per element of
     * the given array.
     */
    private static Element[] makeDexElements(ArrayList<File> files,
            File optimizedDirectory) {
        ArrayList<Element> elements = new ArrayList<Element>();

        /*
         * Open all files and load the (direct or contained) dex files
         * up front.
         */
        for (File file : files) {
            File zip = null;
            DexFile dex = null;
            String name = file.getName();

            if (name.endsWith(DEX_SUFFIX)) {
                // Raw dex file (not inside a zip/jar).
                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException ex) {
                    System.logE("Unable to load dex file: " + file, ex);
                }
            } else if (name.endsWith(APK_SUFFIX) || name.endsWith(JAR_SUFFIX)
                    || name.endsWith(ZIP_SUFFIX)) {
                zip = file;

                try {
                    dex = loadDexFile(file, optimizedDirectory);
                } catch (IOException ignored) {
                    /*
                     * IOException might get thrown "legitimately" by
                     * the DexFile constructor if the zip file turns
                     * out to be resource-only (that is, no
                     * classes.dex file in it). Safe to just ignore
                     * the exception here, and let dex == null.
                     */
                }
            } else {
                System.logW("Unknown file type for: " + file);
            }

            if ((zip != null) || (dex != null)) {
                elements.add(new Element(file, zip, dex));
            }
        }

        return elements.toArray(new Element[elements.size()]);
    }

    ...
}
```

所以MultiDex在安装开始时，会先通过反射调用[BaseDexClassLoader](http://androidxref.com/4.2.2_r1/xref/libcore/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java)里
DexPathList类型的pathList字段，接着通过pathList调用[DexPathList](http://androidxref.com/4.2.2_r1/xref/libcore/dalvik/src/main/java/dalvik/system/DexPathList.java)的makeDexElements方法，将上面解压得到的additionalClassPathEntries（二级dex文件列表）封装成Element数组。

需要注意的是，makeDexElements最终会去进行dex2opt操作，这是一个比较耗时的过程，如果全部放在main线程去处理的话，比较影响用户体验，甚至可能引起ANR。

dex2opt后，/data/data/<package-name>/code_cache/secondary-dexes/下的会出现优化后的文件：<package-name>.apk.classes2.dex等

最后调用`MultiDex.expandFieldArray`：

```java
private static void expandFieldArray(Object instance, String fieldName, Object[] extraElements) throws NoSuchFieldException, IllegalArgumentException, IllegalAccessException {
    Field jlrField = findField(instance, fieldName);
    Object[] original = (Object[])((Object[])jlrField.get(instance));
    Object[] combined = (Object[])((Object[])Array.newInstance(original.getClass().getComponentType(), original.length + extraElements.length));
    System.arraycopy(original, 0, combined, 0, original.length);
    System.arraycopy(extraElements, 0, combined, original.length, extraElements.length);
    jlrField.set(instance, combined);
}
```

expandFieldArray同样是通过反射调用，找到pathList中的dexElements字段，并将上一步生成的封装了二级dex的Element数组添加到dexElements之后，完成整个安装流程


## 总结

通过上面的分析，我们可以总结出来MultiDex的原理如下：

1. apk在Applicaion实例化之后，会检查系统版本是否支持MultiDex，判断二级dex是否需要安装；
2. 如果需要安装则会从apk中解压出classes2.dex并将其拷贝到应用的/data/data/<package-name>/code_cache/secondary-dexes/目录下；
3. 通过反射将classes2.dex等注入到当前的ClassLoader的pathList中，完成整体安装流程。













