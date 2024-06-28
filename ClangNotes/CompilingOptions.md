1. 编写指针相关llvm pass时，若实现中需要诸如:

   > type->isPointerTy() && type->getPointerElementType()->isFunctionTy()

   需要注意两点：

   * `getPointerElementType`在llvm16版本下已被标记为`Deprecated`（实际上更早的某个版本就已更新该内容），后续某版本可能彻底移除。如果考虑使用该api，需要注意llvm版本，原文如下：

     > This method is deprecated without replacement. Pointer element types are not available with opaque pointers.

   * 尽管被标记为`Deprecated`，但该api依然可以正常使用，不过需要结合相应的编译参数：

     > -Xclang -no-opaque-pointers

     [当然，更早的版本应该能够直接使用该特性]

2. 自行构建的clang，若需要执行交叉编译，通常默认使用系统apt-get获取到的交叉编译器。

   使用指定的toolchain，如optee自带的 / 自行下载解压的prebuilt版本等，以optee自带的`aarch64-none-linux-gnu`为例：

   * 编译参数：

     > --target=aarch64-none-linux-gnu 
     >
     > --gcc-toolchain=/path/to/toolchains/aarch64 
     >
     > --sysroot=/path/to/toolchains/aarch64/aarch64-none-linux-gnu/libc

   * 编译参数对应的目录结构：

     ``` shell
     toolchains
      | -- aarch32
      | -- aarch64
               | -- aarch64-none-linux-gnu
                         | -- bin
                         | -- include
                         | -- lib
                         | -- lib64
                         | -- libc
               | -- bin
               | -- include
               | -- lib
               | -- lib64
               | -- libexec
               | -- share
               | -- xxx-manifest.txt
     ```

     

3. 