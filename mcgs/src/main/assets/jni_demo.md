# Android Studio开发JNI示例

## 一、JNI和NDK介绍

JNI（Java Native Interface），是方便Java调用C、C++等Native代码所封装的一层接口，相当于一座桥梁。通过JNI可以操作一些Java无法完成的与系统相关的特性，尤其在图像和视频处理中大量用到。

NDK（Native Development Kit）是Google提供的一套工具，其中一个特性是提供了交叉编译，即C或者C++不是跨平台的，但通过NDK配置生成的动态库却可以兼容各个平台。比如C在Windows平台编译后生成.exe文件，那么源码通过NDK编译后可以生成在安卓手机上运行的二进制文件.so


## 二、在AS中使用ndk-build开发

Android Studio2.2之前对于JNI开发的支持不是很好，开发一般使用Eclipse+插件编写本地动态库。后面Google官方全面增强了对JNI的支持，包括内置NDK。


### 1. 在AS中新建一个项目

### 2. 声明一个native方法


```

package com.mercury.jnidemo;

public class JNITest {

    public native static String getStrFromJNI();

}

```

### 3.通过javah命令生成头文件

在AS的Terminal中，先进入要调用本地代码的类所在的目录，也就是在项目中的具体路径，比如这里是cd app\src\main\java。然后通过javah命令生成该类的头文件，注意包名+类名.这里是javah -jni com.mercury.jnidemo.JNITest，生成头文件com_mercury_jnidemo_JNITest.h

实际项目最终可以不包含此头文件，不熟悉C的语法的开发人员，借助于该头文件可以知道JNI的相关语法:

```

/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class com_mercury_jnidemo_JNITest */

#ifndef _Included_com_mercury_jnidemo_JNITest
#define _Included_com_mercury_jnidemo_JNITest
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     com_mercury_jnidemo_JNITest
 * Method:    getStrFromJNI
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_mercury_jnidemo_JNITest_getStrFromJNI
  (JNIEnv *, jclass);

#ifdef __cplusplus
}
#endif
#endif


```


首先引入jni.h,里面包含了很多宏定义及调用本地方法的结构体。重点是方法名的格式。这里的JNIEXPORT和JNICALL都是jni.h中所定义的宏。JNIEnv *表示一个指向JNI环境的指针，可通过它来访问JNI提供的接口方法。jclass也是jni.h中定义好的，类型是jobject,实际上是一个不确定类型的指针，这里用来接收Java中的this。实际编写中一般只要遵循Java_包名_类名_方法名就好了。


### 4.实现JNI方法

像上面的头文件只是定义了方法，并没有实现，就像一个接口一样。这里就用C写一个简单的无参的JNI方法。

先创建一个jni目录，我直接在src的父目录下创建的，也可以在其他目录创建，因为最终只需要编译好的动态库。在jni目录下创建Android.mk和demo.c文件。


**Android.mk是一个makefile配置文件，安卓大量采用makefile进行自动化编译。LOCAL_MODULE定义的名称就是编译好的so库名称，比如这里是jni-demo，最终生成的动态库名称就叫libjni-demo.so。 LOCAL_SRC_FILES表示参与编译的源文件名称，这里就是demo.c**


**Android.mk**

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE := jni-demo
LOCAL_SRC_FILES := demo.c

include $(BUILD_SHARED_LIBRARY)

```

**这里的demo.c实现了一个很简单的方法，返回String类型**


```
#include<jni.h>

jstring Java_com_mercury_jnidemo_JNITest_getStrFromJNI(JNIEnv *env,jobject thiz){
    return (*env)->NewStringUTF(env,"I am Str from jni libs!");
}


```

**Application.mk**

**这时候NDK编译生成的动态库会有四个CPU平台:arm64-v8a、armeabi-v7a、x86、x86_64。如果创建Application.mk就可以指定要生成的CPU平台，语法也很简单:**


```

APP_ABI := all

```

这样就会生成各个CPU平台下的动态库。


### 5.使用ndk-build编程生成.so库

切回到jni目录的父目录下，在Terminal中运行ndk-build指令，就可以在和jni目录同级生成一个libs文件夹，里面存放相对应的平台的.so库。同时生成的还有一个中间临时的obj文件夹，和jni文件夹可以一起删除。

**需要注意，使用NDK一定要先在build.gradle下要配置ndk-build的相关路径，这样在编写本地代码时才会有相关的提示功能，并且可以关联到相关的头文件：**


```

externalNativeBuild {
        ndkBuild {
            path 'jni/Android.mk'
        }
    }

```


**build.gradle中加入以下代码:**

```

sourceSets{
        main{
            jniLibs.srcDirs=['libs']
        }
    }

```

这样就指定了目标.so库的存放位置。但在实际使用中，就算不指定，运行时仍然可以加载正确的.so库文件，并且如果添加该代码后有时会报出以下错误:

```

 Error:Execution failed for task ':usejava:transformNativeLibsWithMergeJniLibsForDebug'.
	> More than one file was found with OS independent path 'lib/x86/libjni-calljava.so'
	> 

```

### 6.加载.so库并调用方法


在类初始化的时候要加载该.so库，一般会写在静态代码块里。名称就是前面的LOCAL_MODULE。

```

  static {
        System.loadLibrary("jni-demo");
    }


```

需要注意的是如果是有参的JNI方法，那么直接在参数列表里补充在jni.h预先typedef好的数据类型就可以了。


## 三、JNI调用Java

不同于JNI调用C，JNI调用Java的过程不是单独存在的。而是编写native方法，Java先通过JNI调用该方法，在方法内部再去回调类中对应的Java方法。步骤有些类似于Java中的反射。这里写定义三个点击事件，三个Native方法，三种Java的方法类型，根据相关的Log判断是否成功。



```
public class MainActivity extends AppCompatActivity {

    public static final String TAG = "MainActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    static {
        System.loadLibrary("jni-calljava");
    }

    public void noParamMethod() {
        Log.i(TAG, "无参的Java方法被调用了");
    }

    public void paramMethod(int number) {
        Log.i(TAG, "有参的Java方法被调用了" + number + "次");
    }

    public static void staticMethod() {
        Log.i(TAG, "静态的Java方法被调用了");
    }

    public void click1(View view) {
        test1();
    }

    public void click2(View view) {
        test2();
    }

    public void click3(View view) {
        test3();
    }

    public native void test1();

    public native void test2();

    public native void test3();

}


```


### 1.调用Java无参方法

JNI调用本地方法，根据类名找到类，注意类名用"/"分隔。<BR>
找到类后，根据方法名找到方法。该函数GetMethodID最后一个形参是该形参列表的签名。不同于Java，C中是通过签名标识去找方法。<BR>
获取方法的签名:首先定位到该类的字节码文件所在的父目录，一般在module\build\intermediates\classes\debug>,通过javap -s com.mercury.usejava.MainActivity获取整个类所有的内部类型签名。无参方法test1()的签名是()V。<BR>
通过JNIEnv对象的CallVoidMethod来完成方法的回调,最后一个形参是可变参数。

```

JNIEXPORT void JNICALL Java_com_mercury_usejava_MainActivity_test1
  (JNIEnv * env, jobject obj){
       //回调MainActivity中的noParamMethod
    jclass clazz = (*env)->FindClass(env, "com/mercury/usejava/MainActivity");
    if (clazz == NULL) {
        printf("find class Error");
        return;
    }
    jmethodID id = (*env)->GetMethodID(env, clazz, "noParamMethod", "()V");
    if (id == NULL) {
        printf("find method Error");
    }
    (*env)->CallVoidMethod(env, obj, id);
  }
  
```


### 2.调用Java有参方法
类似于无参方法，只是参数签名和可变参数的不同

### 3.调用Java静态方法

**注意获取方法名的方法是GetStaticMethodID，调用方法的函数名是CallStaticVoidMethod，并且由于是静态方法，不应该传入jobject参数，而直接是jclass.**


```
JNIEXPORT void JNICALL Java_com_mercury_usejava_MainActivity_test3
  (JNIEnv * env, jobject obj){
    jclass clazz = (*env)->FindClass(env, "com/mercury/usejava/MainActivity");
    if (clazz == NULL) {
        printf("find class Error");
        return;
    }
    jmethodID id = (*env)->GetStaticMethodID(env, clazz, "staticMethod", "()V");
    if (id == NULL) {
        printf("find method Error");
    }

    (*env)->CallStaticVoidMethod(env, clazz, id);
  }

  
```


## 四、使用CMake开发JNI


CMake是一个跨平台的安装(编译)工具，通过编写CMakeLists.txt，可以生成对应的makefile或project文件，再调用底层的编译。AS 2.2之后工具中增加了对CMake的支持，官方也推荐用CMake+CMakeLists.txt的方式，代替ndk-build+Android.mk+Application.mk的方式去构建JNI项目.


### 1.创建使用CMake构建的项目


**开始前AS要先在SDK Manager中安装SDK Tools->CMake，创建要勾选Include C++ Support。其中会提示配置C++支持的功能.**

创建好的工程主Module下直接就有.externalNativeBuild，多出一个CMakeLists.txt，相当于以前的配置文件。并且在src/main目录下多了一个cpp文件夹，里面存放的是C++文件，相当于以前的jni文件夹。这个是工程创建后AS生成的示例JNI方法，返回了一个字符串。后面开发JNI就可以按照这个目录结构。

**build.gradle下也增加了一些配置：**


```
android {
    ...
    defaultConfig {
        ...
        externalNativeBuild {
            cmake {
                cppFlags "-std=c++14 -frtti -fexceptions"
            }
        }
    }
    buildTypes {
        ...
    }
    externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
        }
    }
}


```


defaultConfig中的externalNativeBuild各项属性和前面创建项目时的选项配置有关，外部的externalNativeBuild则定义了CMakeLists.txt的存放路径。

如果只是在自己的项目中使用，CMake的方式在打包APK的时候会自动将cpp文件编译成so文件拷贝进去。如果要提供给外部使用时，Make Project，之后在libs目录下就可以看到生成的对应配置的相关CPU平台的.so文件。

**CMakeLists.txt**

CMakeLists.txt可以自定义命令、查找文件、头文件包含、设置变量，具体可见 官方文档。项目默认生成的CMakeLists.txt核心内容如下:


编译本地库时我们需要的最小的cmake版本
cmake_minimum_required(VERSION 3.4.1)

**相当于Android.mk**

```
add_library( # Sets the name of the library.设置编译生成本地库的名字
             native-lib

             # Sets the library as a shared library.库的类型
             SHARED

             # Provides a relative path to your source file(s).编译文件的路径
             src/main/cpp/native-lib.cpp )

# 添加一些我们在编译我们的本地库的时候需要依赖的一些库，这里是用来打log的库
find_library( # Sets the name of the path variable.
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log )

# 关联自己生成的库和一些第三方库或者系统库
target_link_libraries( # Specifies the target library.
                       native-lib

                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib} )


使用CMakeLists.txt同样可以指定so库的输出路径,但一定要在add_library之前设置，否则不会生效:


set(CMAKE_LIBRARY_OUTPUT_DIRECTORY 
	${PROJECT_SOURCE_DIR}/libs/${ANDROID_ABI}) #指定路径
#生成的so库在和CMakeLists.txt同级目录下的libs文件夹下


```


**如果想要配置so库的目标CPU平台，可以在build.gradle中设置**

```
android {
    ...
    defaultConfig {
        ...
        ndk{
            abiFilters "x86","armeabi","armeabi-v7a"
        }
    }
	...
  
}

```

**需要注意的是，如果是多次使用add_library，则会生成多个so库。如果想将多个本地文件编译到一个so库中，只要最后一个参数添加多个C/C++文件的相对路径就可以**



## 五、总结

**写该文档的目的是在mcgs_demo 中有jni使用调用硬件库，后面有需要的可以参考该文档及mcgs_demo 的source code。**