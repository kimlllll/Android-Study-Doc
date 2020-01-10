图片，每个App都会有，而它，又是最容易引起OOM的。所以，有必要在Android开发中对图片做一些处理。


在Android中，对图片的处理，可以从两个方面着手：
#### 1.减少图片在磁盘上所占空间的大小。
减少图片在磁盘上所占空间的大小，分为两步：
1. 去掉图片的Alpha通道。
2. 使用哈夫曼算法对图片进行压缩。
#### 2.减少图片在内存中的占用大小。
1. 图片在Android程序中所占的内存大小，与图片大小无关，与图片像素点*图片格式有关。
2. 设置BitmapFactory.Options中的inSampleSize属性来进行位图的缩放。
3. 设置BitmapFactory.Options中的inBitmap属性，来实现图片的复用。
4. 加载图片时，使用4级缓存。




## 1.减少图片在磁盘上所占空间的大小。

   Android6.0之前，并没有使用哈夫曼压缩，因此，Android自身提供的图片压缩方法，在6.0之前的压缩效果很差。我们需要自己编写一个方法，解决这个问题。编写这个方法时，就需要用到第三方库libjpeg，实际上，Android本身也是使用的这个库做的图片的压缩，但Android将其放入Skia图形引擎中，我们无法直接使用，我们要使用这个库，就需要自己编译好了，放在项目中使用。

使用NDK编译第三方库libjpeg。
libjpeg这个库非常有名，大多数的第三方图片压缩框架所使用的底层库都是它，它在Github的下载地址：[https://github.com/libjpeg-turbo/libjpeg-turbo](https://github.com/libjpeg-turbo/libjpeg-turbo)  

### 1.1.在Linux中下载这个库。
在Github上复制release版本中的最新版本的Linux版本的链接：[https://github.com/libjpeg-turbo/libjpeg-turbo/archive/2.0.3.tar.gz](https://github.com/libjpeg-turbo/libjpeg-turbo/archive/2.0.3.tar.gz) ，在Linux上使用命令
```

wget  https://github.com/libjpeg-turbo/libjpeg-turbo/archive/2.0.3.tar.gz

```
### 1.2.Linux中解压libjpeg
```
tag -xvf  2.0.3.tar.gz
```
-xvf命令之后跟着的是文件名和后缀
### 1.3.编译下载的libjpeg库 

#### 1.3.1 使用CMake
查看编译文档https://github.com/libjpeg-turbo/libjpeg-turbo/blob/master/BUILDING.md，发现必须要在CMake环境下编译，因此需要下载CMake。
##### 1.3.1.1 下载CMake 
进入CMake官网，点击下载Latest Release 的Linux版本，右键复制其下载链接：[https://github.com/Kitware/CMake/releases/download/v3.16.0/cmake-3.16.0.tar.gz](https://github.com/Kitware/CMake/releases/download/v3.16.0/cmake-3.16.0.tar.gz) 在Linux中下载并解压，下载与解压的命令与1、2相同。

#### 1.3.1.2 安装CMake
进入到Cmake文件夹中，Linux中使用命令 cd 文件夹名称 使用ls命令查看其中的文件： 其中绿色的是可执行文件，比如bootstrap文件就是可执行文件。使用如下命令来运行bootstrap
```
./bootstrap
```

#### 1.3.1.3 安装完成后，依次执行gmake、make install命令，将CMake安装完成。
```
gmake
make install
```

#### 1.3.1.4 使用cmake - -version 查看安装的cmake版本，如果可以看到cmake的版本 说明安装成功
```
cmake - -version
```

### 1.3.2 如果要编译x86的库，还需要下载NASM，下载解压运行的过程和CMake一致。运行configure文件。

### 1.3.3 编译libjpeg库。
编译后的文件是.a文件或者是我们熟悉的.so文件，这些文件我们可以直接放在项目中使用的。
##### 1.3.3.1  编写编译命令
使用cd命令，进入解压后的libjpeg目录，在该目录中新建记事本文件build.sh，使用vim命令创建文本文件。
```
 vim build.sh
```
我们查看编译文档，发现其中给出的编译命令模板如下： 
```
NDK_PATH={full path to the NDK directory-- for example, # NDK路径 在网上下载 如：\..\android-ndk-r19c
  /opt/android/android-ndk-r16b}
TOOLCHAIN={"gcc" or "clang"-- "gcc" must be used with NDK r14b and earlier,
  and "clang" must be used with NDK r17c and later} #c语言的编译器 ： gcc或者clang
ANDROID_VERSION={the minimum version of Android to support.  "21" or later
  is required for a 64-bit build.}

cd {build_directory}
cmake -G"Unix Makefiles" \
  -DANDROID_ABI=arm64-v8a \  # 根据库的类型，这里可以写四种：armeabi-v7a、arm64-v8a、x86、x86_64
  -DANDROID_ARM_MODE=arm \ # 如果是x86类型，这句不用写
  -DANDROID_PLATFORM=android-${ANDROID_VERSION} \
  -DANDROID_TOOLCHAIN=${TOOLCHAIN} \
  -DCMAKE_ASM_FLAGS="--target=aarch64-linux-android${ANDROID_VERSION}" \
  -DCMAKE_TOOLCHAIN_FILE=${NDK_PATH}/build/cmake/android.toolchain.cmake \
  [additional CMake flags] {source_directory}
make
```
其中NDK_PATH是NDK路径，NDK需要自己在网上下载，下载地址是：https://developer.android.google.cn/ndk/downloads/older_releases.html
TOOLCHAIN 是C语言编译器的类型，一般有两种gcc和clang。ANDROID_VERSION是指，Android的版本号。DANDROID_ABI 是所要编译的库的类型，有四种，分别是：armeabi-v7a、arm64-v8a、x86、x86_64。这几个选项都是根据自己的情况填写的。填写之后的编译命令如下所示：
```
#!/bin/bash
# Set these variables to suit your needs
NDK_PATH=../android-ndk-r17c
TOOLCHAIN=gcc
ANDROID_VERSION=21

cmake -G"Unix Makefiles" \
  -DANDROID_ABI=x86_64 \
  -DANDROID_PLATFORM=android-${ANDROID_VERSION} \
  -DANDROID_TOOLCHAIN=${TOOLCHAIN} \
  -DCMAKE_TOOLCHAIN_FILE=${NDK_PATH}/build/cmake/android.toolchain.cmake \
make
```
这里需要注意的是，在编写时，需要添加一个抬头，在文件最前面写上：
```
#!/bin/bash
```
编写完成之后保存： esc —> 写入命令：wq（写入退出） 编译文件就写好了

#### 1.3.3.2 执行build.sh文件
首先需要给build.sh添加执行权限，使用以下命令：
```
 chmod +x build.sh
```
添加完执行权限之后，build.sh文件变成了绿色，也就是Linux中的可执行文件。然后使用./build.sh命令来执行build.sh文件
```
./build.sh
```
编译好之后，回到之前的文件夹，使用ls查看其中的文件，会发现其中多了一个libturbojpeg.a的静态库

### 1.4.使用静态库 
#### 1.4.1 创建项目
创建我们要编写native方法的项目，在创建项目时，需要勾选include c++ support ，勾选完成后，会发现项目中多了个cpp文件夹，这个文件夹就是用来装一些NDK相关的文件等。然后将libjpeg-turbo文件夹中的 jconfig.h、jerror.h  、jmorecfg.h 、jpeglib.h 、turbojpeg.h 、静态库 libturbojpeg.a 拷贝进项目里。如下图所示：
![Cpp文件夹](https://user-gold-cdn.xitu.io/2019/12/26/16f416642a7e96c2?w=482&h=568&f=png&s=33643)

#### 1.4.2 准备工作 ：
在build.gradle文件中写入：
```
cmake {
    cppFlags ""
    abiFilers "x86_64"
    //指定 android编译器
    arguments '-DANDROID_TOOLCHAIN=gcc'
}
```
编辑 cMakeLists.txt 文件
```
cmake_minimum_required(VERSION 3.4.1)

add_library(
        native-lib
        SHARED
        src/main/cpp/native-lib.cpp)

add_library(libjpeg STATIC IMPORTED)
set_target_properties(libjpeg PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/src/main/cpp/libs/libturbojpeg.a)

#引入头文件    import
include_directories(src/main/cpp/include)

target_link_libraries(
        native-lib
        libjpeg
        #jnigraphics是安卓NDK目录中直接有的
        jnigraphics
        log)
```

写好之后，编译，编译通过之后，我们在项目中写的native方法，都会在navite-lib.cpp文件中生成对应的c语言方法。

#### 1.4.3 编写Native方法，进行图片的压缩 
前面可以说都是准备工作，到这一步，终于到正题了，开始撸代码了。
在Java文件中编写native方法，会在项目中的native-lib.cpp文件中生成相应的C语言方法。
```
//Java方法 第一个参数是要压缩的Bitmap，第二个参数是压缩质量，第三个参数是压缩后的图片放置的位置
public native void nativeCompress(Bitmap bitmap,int q,String path);
```
对应生成的c语言方法,首先我们需要引入几个头文件，C语言里的头文件就相当于Java中的库，引入之后就可以使用其中的方法。
```
#include <malloc.h> //操作内存区需要的头文件
#include <android/bitmap.h> // 操作Bitmap对象需要的头文件
#include <jpeglib.h> 
```

##### 1.4.3.1 去掉图片的Alpha通道。
什么叫做去掉图片的Alpha通道呢？ 我们知道，图片都是由一个个的像素点构成的，每个像素点都是由argb组成，a就是透明度，rgb分别是红、绿、蓝三种颜色。去掉Alpha通道，就是把每个像素点的透明度去掉。
```
extern "C"
JNIEXPORT void JNICALL
Java_com_example_administrator_lsn_15_1demo_MainActivity_nativeCompress(JNIEnv *env,
                                                                        jobject instance,
                                                                        jobject bitmap, jint q,
                                                                        jstring path_) {
    const char *path = env->GetStringUTFChars(path_, 0);
    //从bitmap获取argb数据
    AndroidBitmapInfo info;//info=new 对象();
    //获取里面的信息
    AndroidBitmap_getInfo(env, bitmap, &info);//  void method(list)
    //得到图片中的像素信息
    uint8_t *pixels;//uint8_t char    java   byte     *pixels可以当byte[]
    AndroidBitmap_lockPixels(env, bitmap, (void **) &pixels);
    //jpeg argb中去掉他的a ===>rgb
    int w = info.width;
    int h = info.height;
    int color;
    //开一块内存用来存入rgb信息
    uint8_t *data = (uint8_t *) malloc(w * h * 3);//data中可以存放图片的所有内容
    uint8_t *temp = data;
    uint8_t r, g, b;//byte
    //循环取图片的每一个像素
    for (int i = 0; i < h; i++) {
        for (int j = 0; j < w; j++) {
            color = *(int *) pixels;//0-3字节  color4 个字节  一个点
            //取出rgb
            r = (color >> 16) & 0xFF;//    #00rrggbb  16  0000rr   8  00rrgg
            g = (color >> 8) & 0xFF;
            b = color & 0xFF;
            //存放，以前的主流格式jpeg    bgr
            *data = b;
            *(data + 1) = g;
            *(data + 2) = r;
            data += 3;
            //指针跳过4个字节
            pixels += 4;
        }
    }
    //把得到的新的图片的信息存入一个新文件中，这个方法中，使用哈夫曼压缩，将图片进一步的压缩
    write_JPEG_file(temp, w, h, q, path);


    //释放内存
    free(temp);
    AndroidBitmap_unlockPixels(env, bitmap);
    env->ReleaseStringUTFChars(path_, path);

}

```

![内存示意图](https://user-gold-cdn.xitu.io/2019/12/26/16f41665e231a577?w=1230&h=1070&f=png&s=163712)


在代码注释中，每一步已经写的很清楚。辅以图片的解释，大致的思路就是，取得图片的每一个像素点，使用位运算取出rgb，开一块新的内存，将取出的每一个rgb存入新内存中。这个新内存显示的图片，就是去掉了Alpha通道的图片了。


##### 1.4.3.2 使用哈夫曼算法进行图片的压缩
首先来了解一下哈夫曼算法。哈夫曼算法，也叫做最优二叉树，原本是哈夫曼当年为了解决远距离通信的数据传输最优化的问题。这里被我们用来做图片的压缩。
我们知道，图片是由一个个的像素点组成的，我们取到这个图片每种像素点的个数，假如，一张图片只有6种颜色，我们将其排序。
![哈夫曼算法一](https://user-gold-cdn.xitu.io/2019/12/26/16f416642ff3b239?w=1240&h=1135&f=png&s=231232)
![哈夫曼算法二](https://user-gold-cdn.xitu.io/2019/12/26/16f416642eaf5f84?w=1240&h=1063&f=png&s=198627)
![哈夫曼算法三](https://user-gold-cdn.xitu.io/2019/12/26/16f416642f0344f1?w=1240&h=950&f=png&s=240440)

一开始，图片的每个像素是由RGB组成，是3 * 8byte=24b，
这张图片所占的字节就是 5 * 24+8 * 24+15 * 24+···+30 * 24，使用哈夫曼算法得到的结果就是：
4b+4b+3b+2b+1b+2b，可以看到，使用哈夫曼算法压缩后的图片，压缩率可达到90%

那么怎么去写哈夫曼算法呢？ 在去掉Alpha通道那一步里，我们留了一个方法write_JPEG_file(temp, w, h, q, path)还没写，就在这个方法中，来写哈夫曼压缩。

使用哈夫曼压缩一共要经历7步，这7步是固定的，只需调用第三方库的一些方法即可。
1、创建jpeg压缩对象
2、指定存储文件
3、设置压缩参数
4、开始压缩
5、循环写入每一行数据
6、压缩完成
7、释放jpeg对象

```
void write_JPEG_file(uint8_t *data, int w, int h, jint q, const char *path) {
//    3.1、创建jpeg压缩对象
    jpeg_compress_struct jcs;
    //错误回调
    jpeg_error_mgr error;
    jcs.err = jpeg_std_error(&error);
    //创建压缩对象
    jpeg_create_compress(&jcs);
//    3.2、指定存储文件  write binary
    FILE *f = fopen(path, "wb");
    jpeg_stdio_dest(&jcs, f);
//    3.3、设置压缩参数
    jcs.image_width = w;
    jcs.image_height = h;
    //bgr
    jcs.input_components = 3;
    jcs.in_color_space = JCS_RGB;
    jpeg_set_defaults(&jcs);
    //开启哈夫曼功能
    jcs.optimize_coding = true;
    jpeg_set_quality(&jcs, q, 1);
//    3.4、开始压缩
    jpeg_start_compress(&jcs, 1);
//    3.5、循环写入每一行数据
    int row_stride = w * 3;//一行的字节数
    JSAMPROW row[1];
    //jcs.next_scanline：行数指针 jcs.image_height：整个图片的高度
    while (jcs.next_scanline < jcs.image_height) {
        //取一行数据 row_stride：一行的大小
        uint8_t *pixels = data + jcs.next_scanline * row_stride;
        row[0]=pixels;
        //写一行数据
        jpeg_write_scanlines(&jcs,row,1);
    }
//    3.6、压缩完成
    jpeg_finish_compress(&jcs);
//    3.7、释放jpeg对象
    fclose(f);
    jpeg_destroy_compress(&jcs);
}
```
到这里磁盘压缩就写完了，要使用这两个方法，调用在Java代码中写的native方法即可。
```
public native void nativeCompress(Bitmap bitmap,int q,String path);
```

## 2.减少图片在内存中的占用大小
### 2.1. 图片在Android程序中所占的内存大小，与图片大小无关，与图片像素点*图片格式有关。

图片在Android程序中所占内存的大小和图片在磁盘上的大小其实没有多大的关系，真正有关系的是Android程序中的图片所在的Drawable文件夹。
为什么这样说呢，Android系统，会根据不同的Drawable文件夹来缩放图片，而图片在内存中的大小，是由图片长* 图片宽* 4来决定的（图片长* 图片宽 = 整张图片的像素点个数，在RGB_8888格式下每个像素点占4字节（argb各占一字节））。

因此，我们要减少图片在Android程序中所占用的内存，首先需要注意将图片放在哪个文件夹下；然后要注意使用哪一种图片格式。
 Android中的图片格式有以下几种：
| 位图规格 | 每像素所占字节数   |
|:--------:| :----------:|
| ARGB_8888 | 32 |
| RGB_565 |16 |
| ARGB_4444 |16 |
| ALPHA_8 | 8 |

ARGB_8888是系统默认的位图格式。其他几种都减小了位图通道，可以减少内存开销并提升内存显示的性能。
如果不需要Alpha通道，除了大图模式，一般都可以使用RGB_565，几乎看不出差别。
如果需要更小的格式，但需要透明通道，就可以使用ARGB_4444，减少了一半的数据，但保留了透明通道，视觉上差异较大，一般可以用于用户头像。
Aplha_8主要是用于染色，图片不常用。
如何设置图片的位图规格呢？ 在BitmapFactory.Options中设置，代码如下：
```
BitmapFactory.Options options = new BitmapFactory.Options();
options.inPreferredConfig = Bitmap.Config.RGB_565;
BitmapFactory.decodeStream(is, null,options);
```

### 2.2.使用inSampleSize属性。
BitmapFactory.Options 中的inSampleSize 属性实现了位图的缩放功能。将这个属性设置为1时，可以在不加载完整大小图片的前提下，生成一张只有原始图片部分大小的新图片。设置为2时，获得只有1/2大小的图片。设置为4，获得1/4大小的图片，以此类推。
```
BitmapFactory.Options options = new BitmapFactory.Options();
options.inSampleSize = 4;
BitmapFactory.decodeStream(is, null,options);
```

### 2.3.使用inBitmap属性，内存复用。
  如果设置了这个属性，那么，当需要使用decode方法显示一张图片时，decode方法会尝试重用一个已经存在的内存块，从而改善性能，不会再重新分配和释放内存。这里需要注意的是，在Android4.4之前，只能重用相同大小的Bitmap内存区域，而4.4之后，可以重用任何Bitmap的内存区域。
```
BitmapFactory.Options options=new BitmapFactory.Options();
//如果要复用，需要设计成易变
options.inMutable=true;
Bitmap bitmap=BitmapFactory.decodeResource(getResources(),R.mipmap.wyz_p,options);
// 将inBitmap设置成异变内存块bitmap，那么下次再显示图片时，就会使用内存块bitmap 
options.inBitmap=bitmap;//需要指定复用的内存块
bitmap=BitmapFactory.decodeResource(getResources(),R.mipmap.wyz_p,options);
```

前面几步了解之后，我们可以将其整理封装成一个方法，放入工具类中，每次加载图片时，都使用这个工具类加载。
```
public class ImageResize {

    /**
     *  缩放bitmap
     * @param context
     * @param id
     * @param maxW
     * @param maxH
     * @return
     */
    public static Bitmap resizeBitmap(Context context,int id,int maxW,int maxH,boolean hasAlpha,Bitmap reusable){
        Resources resources = context.getResources();
        BitmapFactory.Options options = new BitmapFactory.Options();
        // 只解码出 outxxx参数 比如 宽、高
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeResource(resources,id,options);
        //根据宽、高进行缩放
        int w = options.outWidth;
        int h = options.outHeight;
        //设置缩放系数
        options.inSampleSize = calcuteInSampleSize(w,h,maxW,maxH);
        if (!hasAlpha){
            options.inPreferredConfig = Bitmap.Config.RGB_565;
        }
        options.inJustDecodeBounds = false;
        //设置成能复用
        options.inMutable=true;
        options.inBitmap=reusable;
        return BitmapFactory.decodeResource(resources,id,options);
    }

    /**
     * 计算缩放系数
     * @param w
     * @param h
     * @param maxW
     * @param maxH
     * @return 缩放的系数
     */
    private static int calcuteInSampleSize(int w,int h,int maxW,int maxH) {
        int inSampleSize = 1;
        if (w > maxW && h > maxH){
            inSampleSize = 2;
            //循环 使宽、高小于 最大的宽、高
            while (w /inSampleSize > maxW && h / inSampleSize > maxH){
                inSampleSize *= 2;
            }
        }
        return inSampleSize;
    }
}
```

### 2.4.加载图片时，使用4级缓存。
4级缓存，就是在三级缓存的基础上，再多添加一层缓存。实际上，是在内存缓存中，再做一级缓存。
首先，了解一下三级缓存。
目前比较流行的图片框架，如Fresco，Glide等，都使用了“内存-磁盘-网络”三级缓存策略。首先，App访问网络拉取图片，分别将加载的图片保存在本地SD卡和内存中。当App再一次需要加载图片时，先判断内存中是否有缓存，有则直接从内存中拉取，没有则查看本地缓存目录，看其中是否有缓存，本地磁盘中如果存在缓存，则从本地缓存卡中拉取，否则从网络加载图片。
#### 2.4.1.内存缓存。
内存缓存通常使用的是LruCache。LruCache在android.util包下，可以翻译为最近最少使用缓存，它用强引用保存需要缓存的对象，内部维护一个队列，LinkedHashMap内部的双向链表，添加了线程安全操作。原理就是：当其中的一个值被访问时，它被放到队列的尾部，当缓存将满时，队列头部的值会被丢弃，之后被垃圾回收。
LruCache的用法，也很简单，只需要调用Android提供的Api即可。
```
        //获取程序最大可用内存 单位是M
        ActivityManager am = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        int memoryClass=am.getMemoryClass();
        //   内存缓存的大小，一般是程序最大可用内存的1/8  单位是byte 
        memoryCache=new LruCache<String,Bitmap>(memoryClass/8*1024*1024){
            /**
             * @return 每张图片value所占内存的大小
               这个方法的作用，就是通过LruCache的缓存大小，
               和返回的bitmap占用的内存大小，来计算LruCache中能放多少张图片。
             */
            @Override
            protected int sizeOf(String key, Bitmap value) {
               /* LruCache中的所有图片都是设置成可用复用的，
                   value.getByteCount()：取到的是图片的大小，而不是内存的大小
                   如果一个内存块被复用后，新的图片大小<内存块大小 ，
                   那么得到的是图片大小，而不是内存块的大小  */
                if(Build.VERSION.SDK_INT > Build.VERSION_CODES.KITKAT){
                    return value.getAllocationByteCount();//19之后，获取内存块大小
                }
              /*取到的是图片的大小，而不是内存的大小，
                但在19之前，必须是相同大小的Bitmap才可以复用，
                因此可以直接使用这个方法获取内存大小。*/
                return value.getByteCount();
            }
            /**
             * 当lru满了，bitmap从lru中移除对象时，会回调 
             * oldValue就是要从LruCache中移除的对象
             */
            @Override
            protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
                oldValue.recycle();
            }
        };
```

#### 在内存缓存的基础上，再添加一级缓存，也就是，被丢弃的Bitmap，不急着回收，而是放入一个复用池中。 再在复用池中开启一个引用队列，如果复用池中的Bitmap被GC扫到过，那么就将该Bitmap放入到引用队列中，将其释放。这里要注意的是，复用池中不是图片，而是一个个可以复用的内存块。

第四级缓存的代码如下：
1.  定义一个复用池
```
   public static Set<WeakReference<Bitmap>> reuseablePool;
```
2.定义一个引用队列
```
ReferenceQueue referenceQueue;
```
3.单开一个线程，死循环，一直轮询引用队列中是否已有Bitmap。如果没有，一直阻塞，如果有就移除。
```
 Thread clearReferenceQueue;
    boolean shutDown;

    private ReferenceQueue<Bitmap> getReferenceQueue(){
        if(null==referenceQueue){
            //当弱用引需要被回收的时候，会进到这个队列中
            referenceQueue=new ReferenceQueue<Bitmap>();
            //单开一个线程，去扫描引用队列中GC扫到的内容，交到native层去释放
            clearReferenceQueue=new Thread(new Runnable() {
                @Override
                public void run() {
                    while(!shutDown){
                        try {
                            //remove 获取并移除头结点，是阻塞式的 ，如果队列中没有数据，就会一直阻塞在这里。
                            Reference<Bitmap> reference=referenceQueue.remove();
                            Bitmap bitmap=reference.get();
                            if(null!=bitmap && !bitmap.isRecycled()){
                                bitmap.recycle();
                            }
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            });
            clearReferenceQueue.start();
        }
        return referenceQueue;
    }

```
4. 在内存缓存中，把队伍头的Bitmap，也就是要回收的Bitmap放入引用队列中。也就是在LruCache的entryRemoved中，把oldValue放入复用池中。
```
            @Override
            protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
                if(oldValue.isMutable()){//如果是设置成能复用的内存块，拉到java层来管理
                    //3.0以下 Bitmap放在 native层，3.0--8.0 Bitmap放在java层 8.0之后 Bitmap又放回Navite层
                    //把这些图片放到一个复用沲中
                    reuseablePool.add(new WeakReference<Bitmap>(oldValue,referenceQueue));
                }else{
                    //oldValue就是移出来的对象
                    oldValue.recycle();
                }
            }
```

#### 2.4.2.磁盘缓存。
磁盘缓存使用的是第三方的DiskLruCache。DiskLruCache是Jake Wharton写的一个第三方库，它的地址是https://github.com/JakeWharton/DiskLruCache，下载下来之后，发现其中只用三个类，拷贝进自己的项目中进行使用即可。
1. 开启磁盘缓存，使用open方法
```
diskLruCache = DiskLruCache.open(new File(dir), BuildConfig.VERSION_CODE, 1, 10 * 1024 * 1024);
```
2.写入磁盘缓存，使用DiskLruCache.Editor
```
/**
     * 加入磁盘缓存
     */
    public void putBitMapToDisk(String key,Bitmap bitmap){
        DiskLruCache.Snapshot snapshot=null;
        OutputStream os=null;
        try {
            snapshot=diskLruCache.get(key);
            //如果缓存中已经有这个文件  不理他
            if(null==snapshot){
                //如果没有这个文件，就生成这个文件
                DiskLruCache.Editor editor=diskLruCache.edit(key);
                if(null!=editor){
                    os=editor.newOutputStream(0);
                    bitmap.compress(Bitmap.CompressFormat.JPEG,50,os);
                    editor.commit();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            if(null!=snapshot){
                snapshot.close();
            }
            if(null!=os){
                try {
                    os.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
```
3.从磁盘缓存中取
```
 /**
     * 从磁盘缓存中取
     */
    public Bitmap getBitmapFromDisk(String key,Bitmap reuseable){
        DiskLruCache.Snapshot snapshot=null;
        Bitmap bitmap=null;
        try {
            snapshot=diskLruCache.get(key);
            if(null==snapshot){
                return null;
            }
            //获取文件输入流，读取bitmap
            InputStream is=snapshot.getInputStream(0);
            //解码个图片，写入
            options.inMutable=true;
            options.inBitmap=reuseable;
            bitmap=BitmapFactory.decodeStream(is,null,options);
            if(null!=bitmap){
                memoryCache.put(key,bitmap);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally{
            if(null!=snapshot){
                snapshot.close();
            }
        }
        return bitmap;
    }
```

整理成一个工具类，就是这样：
```
public class ImageCache {

    private static ImageCache instance;
    private Context context;
    private LruCache<String,Bitmap> memoryCache;
    private DiskLruCache diskLruCache;

    BitmapFactory.Options options=new BitmapFactory.Options();

    /**
     * 定义一个复用池
     */
    public static Set<WeakReference<Bitmap>> reuseablePool;


    public static ImageCache getInstance(){
        if(null==instance){
            synchronized (ImageCache.class){
                if(null==instance){
                    instance=new ImageCache();
                }
            }
        }
        return instance;
    }

    //引用队列
    ReferenceQueue referenceQueue;
    Thread clearReferenceQueue;
    boolean shutDown;

    private ReferenceQueue<Bitmap> getReferenceQueue(){
        if(null==referenceQueue){
            //当弱用引需要被回收的时候，会进到这个队列中
            referenceQueue=new ReferenceQueue<Bitmap>();
            //单开一个线程，去扫描引用队列中GC扫到的内容，交到native层去释放
            clearReferenceQueue=new Thread(new Runnable() {
                @Override
                public void run() {
                    while(!shutDown){
                        try {
                            //remove 获取并移除头结点，是阻塞式的
                            Reference<Bitmap> reference=referenceQueue.remove();
                            Bitmap bitmap=reference.get();
                            if(null!=bitmap && !bitmap.isRecycled()){
                                bitmap.recycle();
                            }
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            });
            clearReferenceQueue.start();
        }
        return referenceQueue;
    }

    //dir是用来存放图片文件的路径
    public void init(Context context,String dir){
        this.context=context.getApplicationContext();

        //复用池 使用一个带锁的set集合
        reuseablePool=Collections.synchronizedSet(new HashSet<WeakReference<Bitmap>>());

        ActivityManager am = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        //获取程序最大可用内存 单位是M
        int memoryClass=am.getMemoryClass();
        // 内存缓存的大小，一般是程序最大可用内存的1/8 单位是byte
        memoryCache=new LruCache<String,Bitmap>(memoryClass/8*1024*1024){
            /**
             *
             * @return bitmap占用的内存大小 
             */
            @Override
            protected int sizeOf(String key, Bitmap bitmap) {
                //19之前   必需同等大小，才能复用  inSampleSize=1
                if(Build.VERSION.SDK_INT > Build.VERSION_CODES.KITKAT){
                    return bitmap.getAllocationByteCount();
                }
                return bitmap.getByteCount();
            }
            /**
             * 当lru满了，bitmap从lru中移除对象时，会回调
             */
            @Override
            protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
                if(oldValue.isMutable()){//如果是设置成能复用的内存块，拉到java层来管理
                    //3.0以下   Bitmap   native
                    //3.0以后---8.0之前  java
                    //8。0开始      native
                    //把这些图片放到一个复用沲中
                    reuseablePool.add(new WeakReference<Bitmap>(oldValue,referenceQueue));
                }else{
                    //oldValue就是移出来的对象
                    oldValue.recycle();
                }


            }
        };
        // 开启磁盘缓存，第三个参数 valueCount:表示一个key对应valueCount个文件  第四个参数 磁盘缓存的最大缓存
       try {
           diskLruCache = DiskLruCache.open(new File(dir), BuildConfig.VERSION_CODE, 1, 10 * 1024 * 1024);
       }catch(Exception e){
           e.printStackTrace();
       }

       getReferenceQueue();
    }





    public void putBitmapToMemeory(String key,Bitmap bitmap){
        memoryCache.put(key,bitmap);
    }
    public Bitmap getBitmapFromMemory(String key){
        return memoryCache.get(key);
    }
    public void clearMemoryCache(){
        memoryCache.evictAll();
    }



    /*
    从复用池中取得Bitmap 
    这个方法的使用时机：是在内存缓存中没有某张图片时，调用该方法看缓存池中有没有这张图片
    */
    public Bitmap getReuseable(int w,int h,int inSampleSize){
        if(Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB){
            //3.0之前 
            return null;
        }
        Bitmap reuseable=null;
        Iterator<WeakReference<Bitmap>> iterator = reuseablePool.iterator();
        while(iterator.hasNext()){
            Bitmap bitmap=iterator.next().get();
            if(null!=bitmap){
                //看是否可以复用，bitmap必须<=复用的内存块大小才可以复用
                if(checkInBitmap(bitmap,w,h,inSampleSize)){
                    //从复用池中拿一个，就删掉一个
                    reuseable=bitmap;
                    iterator.remove();
                    break;
                }else{
                    iterator.remove();
                }
            }
        }
        return reuseable;
    }



    private boolean checkInBitmap(Bitmap bitmap, int w, int h, int inSampleSize) {
        if(Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT){
            //19以前，必须宽高相同，才可以复用
            return bitmap.getWidth()==w && bitmap.getHeight()==h && inSampleSize==1;
        }
        if(inSampleSize>=1){
            //做过缩放
            w/=inSampleSize;
            h/=inSampleSize;
        }
        int byteCount=w*h*getPixelsCount(bitmap.getConfig());//图片所占的字节数
        return byteCount<=bitmap.getAllocationByteCount();//图片的大小<复用内存块的大小时，可以复用
    }

    private int getPixelsCount(Bitmap.Config config) {
        if(config==Bitmap.Config.ARGB_8888){
            return 4;
        }
        return 2;
    }


    //磁盘缓存的处理
    /**
     * 加入磁盘缓存
     */
    public void putBitMapToDisk(String key,Bitmap bitmap){
        DiskLruCache.Snapshot snapshot=null;
        OutputStream os=null;
        try {
            snapshot=diskLruCache.get(key);
            //如果缓存中已经有这个文件  不理他
            if(null==snapshot){
                //如果没有这个文件，就生成这个文件
                DiskLruCache.Editor editor=diskLruCache.edit(key);
                if(null!=editor){
                    os=editor.newOutputStream(0);
                    bitmap.compress(Bitmap.CompressFormat.JPEG,50,os);
                    editor.commit();
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }finally {
            if(null!=snapshot){
                snapshot.close();
            }
            if(null!=os){
                try {
                    os.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
    /**
     * 从磁盘缓存中取
     */
    public Bitmap getBitmapFromDisk(String key,Bitmap reuseable){
        DiskLruCache.Snapshot snapshot=null;
        Bitmap bitmap=null;
        try {
            snapshot=diskLruCache.get(key);
            if(null==snapshot){
                return null;
            }
            //获取文件输入流，读取bitmap
            InputStream is=snapshot.getInputStream(0);
            //解码个图片，写入
            options.inMutable=true;
            options.inBitmap=reuseable;
            bitmap=BitmapFactory.decodeStream(is,null,options);
            if(null!=bitmap){
                memoryCache.put(key,bitmap);
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally{
            if(null!=snapshot){
                snapshot.close();
            }
        }
        return bitmap;
    }

}
```

使用：
```
Bitmap bitmap=ImageCache.getInstance().getBitmapFromMemory(String.valueOf(position));
        if(null==bitmap){
            //如果内存没数据，就去复用池找，找的是可以复用的内存块，这个内存块在磁盘缓存中可以使用
            Bitmap reuseable=ImageCache.getInstance().getReuseable(60,60,1);
            //从磁盘找
            bitmap = ImageCache.getInstance().getBitmapFromDisk(String.valueOf(position),reuseable);
            //如果磁盘中也没缓存,就从网络下载
            if(null==bitmap){
                bitmap=ImageResize.resizeBitmap(context,R.mipmap.wyz_p,80,80,false,reuseable);
                ImageCache.getInstance().putBitmapToMemeory(String.valueOf(position),bitmap);
                ImageCache.getInstance().putBitMapToDisk(String.valueOf(position),bitmap);
                Log.i("","从网络加载了数据");
            }else{
                Log.i("","从磁盘中加载了数据");
            }

        }else{
            Log.i("","从内存中加载了数据");
        }
```

到这里，基本上就总结了全部Android中的图片优化的方式。使用这一套全家桶下来，图片是可以减少50%以上的占用内存的。







