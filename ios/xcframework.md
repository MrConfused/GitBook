# XCFramework

Framework:是一个有着特定组织格式的文件夹（bundle）。由Info.plist描述内容组织格式。

静态库（.a/.framework）：在链接时完整的拷贝到可执行程序中，每个用到的程序都需要拷贝。

动态库（.dylib/.tbd/.framework）：运行时动态加载到内存中，可供多个程序共享。



XCFramework：是一种特殊的Framework，但是他里面的代码是支持多个平台和架构的。是一种打包和发布库的方式。



优点：

1.  Packing dependencies under all target platforms and architectures in one single bundle from the box

    不同平台和架构下的代码都被打包到一个budle中。
2.  Connection of the bundle in the format of XCFramework, as a single dependency for all target platforms and architectures

    在不同的平台和架构下使用的方式是一样的，都是以XCFramework的格式来使用。
3.  Missing the need in fat/universal build of the framework

    不再需要生成胖二进制了。
4.  No need to get rid of x86\_64 slices before uploading end applications to AppStore

    应用上传到App Store之前，不再需要处理x86\_64部分（给模拟器用的）了。



组织格式：

![](<../.gitbook/assets/image (6) (1) (1).png>)

