# android-ndk
本人C语言基础很烂，所有的C语言全都是现学现卖，同样的，我也把他当做是一个总结，就目前我的使用情况来看，主要用于java和c的交互。如：在c中调用java本地方法，就需要使用jni提供的api方法。java中调用c的话可能比较常见，像友盟和环信的即时通讯的.so文件，就是c语言的动态库。

# c语言通过jni调用java静态方法 #
先来看一下java端的本地方法，

	package com.jni.test;
	
	public class JniTest {
		/**
		 * 通过jni给getStringFromc返回一个String
		 * @return
		 */
		public native static String getStringFromc();
		
		public static void main(String[] args) {
			JniTest.getStringFromc();
		}
	}

java代码写好之后，通过javah生成.h头文件。这里需要注意，我们必须要在项目bin目录下执行javah命令。深坑~~~~之前试了好多个方法都不行，不知道哪出了问题，直到切换到bin目录下才生成.h文件。
![jni001](http://oif1jvh5f.bkt.clouddn.com/jni001.png "jni001")

可以看到在项目bin目录下生成了我们的.h头文件，
![jni002](http://oif1jvh5f.bkt.clouddn.com/jni002.png "jni002")

将.h头文件粘贴到我们创建好的c程序中。
![jni003](http://oif1jvh5f.bkt.clouddn.com/jni003.png "jni003")

然后切换到ide中并没有发现我们的.h头文件

![jni004](http://oif1jvh5f.bkt.clouddn.com/jni004.png "jni004")

这是因为我们还没有添加到工程中，如下图，添加现有项目。
![jni005](http://oif1jvh5f.bkt.clouddn.com/jni005.png "jni005")

头文件添加成功之后，点开代码查看，我们会发现还有几处报红的地方。
![jni006](http://oif1jvh5f.bkt.clouddn.com/jni006.png "jni006")

这是因为找不到jni.h和jni_md.h， 这俩文件在我们的jdk目录下的include文件下，可以直接进行搜索。
![jni007](http://oif1jvh5f.bkt.clouddn.com/jni007.png "jni007")

然后将两个头文件添加到工程中，引用本地的头文件要用"";
![jni008](http://oif1jvh5f.bkt.clouddn.com/jni008.png "jni008")

到了上图呢，我们才算真正把java本地方法生成的头文件移植到了c程序中。接下来我们需要在c中写实现。

有了头文件，我们直接写实现就行了，创建好c文件，引入头文件，实现头文件中的接口。如下
## c端代码 ##

	#include "stdafx.h"
	#include "com_jni_test_JniTest.h"
	
	JNIEXPORT jstring JNICALL Java_com_jni_test_JniTest_getStringFromc
	(JNIEnv * env, jclass jclz){
		return (*env)->NewStringUTF(env, "hello jni");
	}

这样就给我们的java方法返回了一个字符串“hello jni”，那如何检测呢？我们需要先生成.dll动态库。
![jni009](http://oif1jvh5f.bkt.clouddn.com/jni009.png "jni009")
![jni010](http://oif1jvh5f.bkt.clouddn.com/jni010.png "jni010")
配置好以上两步之后呢，我们点击导航栏生成->生成解决方案。看到log打印成功之后，检查项目路径是否生成.dll文件
![jni011](http://oif1jvh5f.bkt.clouddn.com/jni011.png "jni011")

然后我们将.dll文件拷贝到java工程中，在java代码中调用。
## java端代码 ##
	package com.jni.test;
	
	public class JniTest {
		/**
		 * 通过jni给getStringFromc返回一个String
		 * @return
		 */
		public native static String getStringFromc();
		
		static {
			System.loadLibrary("Jni_Demo");
		}
		
		public static void main(String[] args) {
			System.out.println(getStringFromc());
		}
	}

## 打印结果 ##
![jni012](http://oif1jvh5f.bkt.clouddn.com/jni012.png "jni012")


# c语言通过jni调用java非静态方法 #
截图截的我有点恶心，我后悔了，不想写了怎么办？
## java端代码 ##

	public native String geyStringFromC2();

## c端代码 ##

	JNIEXPORT jstring JNICALL Java_com_jni_test_JniTest_getStringFromc2
	(JNIEnv * env, jclass jclz){
		return (*env)->NewStringUTF(env, "hello jni two");
	}
## 打印结果 ##
![jni013](http://oif1jvh5f.bkt.clouddn.com/jni013.png "jni013")

## 小结： ##

	//静态方法
	JNIEXPORT jstring JNICALL Java_com_jni_test_JniTest_getStringFromc
	(JNIEnv * env, jclass jclz){
		return (*env)->NewStringUTF(env, "hello jni");
	}
	//非静态方法
	JNIEXPORT jstring JNICALL Java_com_jni_test_JniTest_geyStringFromc2
	(JNIEnv * env, jobject jobj){
		return (*env)->NewStringUTF(env, "hello jni two");
	}

我发现生成的头文件中，两次的参数有所不同，当java方法中是静态方法时，这里的参数是(JNIEnv * env, jclass jclz)，而如果java中是非静态的方法时，这里的参数则是(JNIEnv * env, jobject jobj),给我的感觉就像是告诉C，我java中是静态方法，我给你传个类过来，你直接调用好了。或者我java中是非静态方法，我给你传个对象过来，你直接用好了。而不管哪种方式，都能够调用到所有能用的api。


# c语言通过jni调用java非静态域 #

## java端代码 ##

		// jni访问java参数
	public String key = "key";
	
	public native String accessField();

## c端代码 ##
	
	JNIEXPORT jstring JNICALL Java_com_jni_test_JniTest_accessField
	(JNIEnv * env, jobject jobj){
	
		jclass jclz = (*env)->GetObjectClass(env, jobj);
		//属性名称，属性签名
		jfieldID fid = (*env)->GetFieldID(env, jclz, "key", "Ljava/lang/String;");
		//得到fid对应的值
		jstring jstr = (*env)->GetObjectField(env, jobj, fid);
	
		char * c_str = (*env)->GetStringUTFChars(env, jstr, NULL);
	
		char text[30] = "动脑 english";
	
		strcat(text, c_str);
	
		jstring new_str = (*env)->NewStringUTF(env, text);
	
		(*env)->SetObjectField(env, jobj, fid, new_str);
	
		return new_str;
	}

## 打印结果 ##
![jni014](http://oif1jvh5f.bkt.clouddn.com/jni014.png "jni014")

## 小结： ##
细细看的话，发现这玩意很想java反射，大概总结了以下几步
1. 得到jclass
2. 得到fieldId（变量ID）
3. 获取fid对应的jni值 string类型为jstring
4. 将jstring类型转换成c语言的string类型
5. 替换数据
6. 将c语言String类型的数据转换成jstring
7. 重新设置变量值
这么看来这玩意其实一点也不难。


# c语言通过jni调用java静态域 #

## java端代码 ##

	public static int count = 9;
	
	public native void accessStaticField();

## c端代码 ##
	
	//这里访问的是java的静态变量域，所以参数是一个object
	JNIEXPORT void JNICALL Java_com_jni_test_JniTest_accessStaticField
	(JNIEnv * env, jobject jobj){
		jclass jclz = (*env)->GetObjectClass(env, jobj);
		//属性名称，属性签名
		jfieldID fid = (*env)->GetStaticFieldID(env, jclz, "count", "I");
		//得到fid对应的值
		jint count = (*env)->GetStaticIntField(env, jclz, fid);
	
		if (fid == NULL){
			printf("fid is null");
		}
	
		count++;
	
		(*env)->SetStaticIntField(env, jclz, fid, count);
	}

## 打印结果 ##
![jni015](http://oif1jvh5f.bkt.clouddn.com/jni15.png "jni015")

## 小结： ##
1. 得到jclass
2. 得到fieldId（变量ID）
3. 获取fid对应的jni值 int类型为jint
4. count++
5. 重新设置变量值


最后来一张jni数据类型对应的签名图

![jni016](http://oif1jvh5f.bkt.clouddn.com/jni016.png "jni016")