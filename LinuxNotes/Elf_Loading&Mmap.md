1. 关于TASK_SIZE和VA_START的宏定义，参照https://blog.csdn.net/jiang_T/article/details/121258274中arm64的相关描述内容

   

2. elf程序头，每个程序头对应于可执行文件的一个段

   ```c
   typedef struct
   {
     Elf64_Word	p_type;			/* Segment type */               // 4 bytes
     Elf64_Word	p_flags;		/* Segment flags */
     Elf64_Off	p_offset;		/* Segment file offset */          // 8 bytes
     Elf64_Addr	p_vaddr;		/* Segment virtual address */    // 8 bytes
     Elf64_Addr	p_paddr;		/* Segment physical address */
     Elf64_Xword	p_filesz;		/* Segment size in file */       // 8 bytes
     Elf64_Xword	p_memsz;		/* Segment size in memory */
     Elf64_Xword	p_align;		/* Segment alignment */
   } Elf64_Phdr;
   ```

   通常来说p_vaddr和p_paddr的值相同，前者指的是该segment数据应该加载到进程的虚拟地址; 后者指的是该segment数据应该加载到进程的物理地址(如果对应系统使用的是物理地址)

   编译时不指定-static， 就会产生dynamic segment, 这时通过readelf -l查看时会显示这是一个shared object file。

   > Elf file type is DYN (Shared object file)

   否则，将会输出：

   > Elf file type is EXEC (Executable file)

   

3. elf加载

   https://proninyaroslav.gitbooks.io/linux-insides-ru/content/SysCall/linux-syscall-4.html

   https://www.cnblogs.com/edver/p/13752835.html

   do_execve

   -> __do_execve_file.isra.14

   -> exec_binprm

   -> search_binary_handler

   -> load_elf_binary

   -> setup_arg_pages

   

   > __do_execve_file

   其中会调用bprm_mm_init(bprm)，用于初始化bprm的mm结构体，即内存相关分配,主要是初始化了mm_struct（用一个vm_area_struct来初始化这个mm_struct），第一次为进程的栈空间分配了一个页

   ``` c
       ...
       file = do_open_exec(filename);      // 在内核中打开可执行文件
       ...
       retval = bprm_mm_init(bprm);        // 初始化进程内存空间描述符
       ...
       /* 拷贝文件名、环境变量和执行参数到bprm */
       retval = copy_strings_kernel(1, &bprm->filename, bprm);
       ...
       retval = copy_strings(bprm->envc, envp, bprm);
       ...
       retval = copy_strings(bprm->argc, argv, bprm);
       ...
       retval = exec_binprm(bprm);         // 处理bprm
       ...
   ```

   此时，进程的栈空间如下图所示，用来存放二进制文件名、参数和环境变量等：

   ![进程虚拟地址空间-bprm_mm_init.png](https://cdn.nlark.com/yuque/0/2021/png/1679957/1619073695987-abd6cd30-9cd7-4f21-a9cb-25e66030609f.png?t=0.9972158338492184)

   到目前为止，内核还没有识别到可执行文件的格式，也没有解析可执行文件中各个段的数据；在`exec_binprm()`中，会遍历在内核中注册支持的可执行文件格式，并调用该格式的`load_binary`方法来处理对应格式的二进制文件：

   > search_binary_handler：

   递归查找对应文件类型的加载函数（elf则为load_elf_binary)

   > load_elf_binary：

   检查文件的魔术数，确认其是否是elf文件

   解析可执行文件的文件头，找到程序头表，然后解析程序头表的每一个表项，对每一个可加载的段进行映射：在elf文件各段和虚拟地址空间之间建立映射，虚拟地址空间的属性配置由elf段决定，如是否可读写等（可通过readelf -l查看）——将段映射为一个vma

   先调用`setup_arg_pages`

   `setup_arg_pages`:

   ``` c
   int setup_arg_pages(struct linux_binprm *bprm,
               unsigned long stack_top,
               int executable_stack)
   {
       ...
       stack_top = arch_align_stack(stack_top);
       stack_top = PAGE_ALIGN(stack_top);
       ...
       stack_shift = vma->vm_end - stack_top;
       ...
       /* Move stack pages down in memory. */
       if (stack_shift) {
           ret = shift_arg_pages(vma, stack_shift);        // 移动arg pages
           ...
       }
       ...
       stack_expand = 131072UL; /* randomly 32*4k (or 2*64k) pages */
       ...
       if (stack_size + stack_expand > rlim_stack)
           stack_base = vma->vm_end - rlim_stack;
       else
           stack_base = vma->vm_start - stack_expand;
       ...
       ret = expand_stack(vma, stack_base);
       ...
   }
   ```

   前面已经初始化了一个页的栈空间，用来存放二进制文件名、参数和环境变量等；在`setup_arg_pages()`中，把前面这一个页的栈空间移动到`stack_top`的位置；在调用函数时，`stack_top`的值是`randomize_stack_top(STACK_TOP)`，即一个随机地址，这里是为了安全性而实现的栈地址随机化；函数通过`shift_arg_pages()`将页移动到新的地址，移动后的栈如下图所示：

   ![进程虚拟地址空间-shift_arg_pages.png](https://cdn.nlark.com/yuque/0/2021/png/1679957/1619073715280-ba9fe2c7-e41a-44ac-a0de-2a875d118e36.png?t=0.6096794337308024)

   之后在`setup_arg_pages`中：

   ``` c
   stack_expand = 131072UL; /* randomly 32*4k (or 2*64k) pages */
   ...
   if (stack_size + stack_expand > rlim_stack)
       stack_base = vma->vm_end - rlim_stack;
   else
       stack_base = vma->vm_start - stack_expand;
   ...
   ret = expand_stack(vma, stack_base);
   ```

   `expand_stack()`函数用来扩展栈虚拟地址空间的大小，`stack_base`是新的栈基地址，这里的`stack_expand`是一个固定值，大小为`128k`，即此处将栈空间扩展`128k`的大小，扩展后栈空间如下：

   ![进程虚拟地址空间-expand_stack.png](https://cdn.nlark.com/yuque/0/2021/png/1679957/1619073769672-7b6e92c2-f096-446d-8b61-b11c2d141d8b.png?t=0.33818725999851773)

   完成对setup_arg_pages的分析后，来看一下`load_elf_binary`整体代码

   如下所示，包括最后调用的`elf_map`：

   ``` c
   load_elf_binary()
   {
   	...      
       retval = setup_arg_pages(bprm, randomize_stack_top(STACK_TOP),
   				 executable_stack);
       ... 
   	for(i = 0, elf_ppnt = elf_phdata; i < elf_ex->e_phnum; i++, elf_ppnt++) { 
           ...   
           //映射load类型的段    
           if (elf_ppnt->p_type != PT_LOAD)   
               continue;                          
           ...
           //根据elf文件的程序头表项获得vma的读写执行权限
           elf_prot = make_prot(elf_ppnt->p_flags, &arch_state,   
                           	 !!interpreter, false); 
           ...
           //通过mmap映射vma     
   		error = elf_map(bprm->file, load_bias + vaddr, elf_ppnt,   
                           elf_prot, elf_flags, total_size); 
           ...
       }
   }
   ```

   其中涉及的`make_prot`:

   ``` c
   static inline int make_prot(u32 p_flags)
   {
   	int prot = 0;
   
   	if (p_flags & PF_R)
   		prot |= PROT_READ;
   	if (p_flags & PF_W)
   		prot |= PROT_WRITE;
   	if (p_flags & PF_X)
   		prot |= PROT_EXEC;
   	return prot;
   }
   ```

   

   

4. unshare_files()：将当前进程的文件描述符表从共享状态中分离，使其成为独立的副本。在 Linux 中，进程间通常共享相同的文件描述符表，这意味着它们共享相同的打开文件、管道、套接字等。然而，通过调用 `unshare_files` 函数，可以将当前进程的文件描述符表与其他进程分离，使其成为独立的副本。

   这个函数的具体实现会将当前进程的文件描述符表复制一份，使其成为独立的副本。这样，新命名空间中的进程就可以独立地管理自己的文件描述符，而不受原始命名空间中其他进程的影响。

   

5. AT_FDCWD，定义在include/uapi/linux/fcntl.h，特殊值(-100)，该值表明当filename为相对路径的情况下将当前进程的工作目录设置为起始路径，在一些系统调用中，会为这个起始路径指定一个目录，此时AT_FDCWD就会被该目录的描述符所替代（如openat, doexecat）

   [因此shelter中根据`shelter_exec`中后续调用`do_execve`等函数实现，其使用必须为相对地址]

   

6. [https://baike.baidu.com/item/vm_area_struct%E7%BB%93%E6%9E%84%E4%BD%93/23620940?fr=ge_ala]

   `vm_area_struct`

   [https://www.leviathan.vip/2019/01/13/mmap%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/]

   [https://www.cnblogs.com/binlovetech/p/17754173.html]

   mmap源码分析

   https://zhuanlan.zhihu.com/p/588649206?utm_id=0

   task_struct -> mm_struct -> vma_area_struct（一个区域一个，所有区域组成mm_struct）

   `mm_struct`则包括装入的可履行映像信息和进程的页目录指针pgd。该结构还包括有指向`vm_area_struct`结构的几个指针，每一个`vm_area_struct`代表进程的1个虚拟地址区间。`vm_area_struct`结构含有指向`vm_operations_struct`结构的1个指针，`vm_operations_struct`描写了在这个区间的操作。`vm_operations_struct`结构中包括的是函数指针

   ![img](https://pic2.zhimg.com/v2-1be29d6871fdf9f75e8c2a81ad910451_r.jpg)

   

7. Linux 虚拟内存管理

   https://zhuanlan.zhihu.com/p/577035415

   

8. 虚拟内存区域可以映射到物理内存上，也可以映射到文件中，映射到物理内存上我们称之为匿名映射，映射到文件中我们称之为文件映射。

   当我们调用 malloc 申请内存时，如果申请的是小块内存（低于 128K）则会使用 do_brk() 系统调用通过调整堆中的 brk 指针大小来增加或者回收堆内存。

   如果申请的是比较大块的内存（超过 128K）时，则会调用 mmap 在上图虚拟内存空间中的文件映射与匿名映射区创建出一块 VMA 内存区域（这里是匿名映射）。这块匿名映射区域就用 `struct anon_vma` 结构表示。

   当调用 mmap 进行文件映射时，vm_file 属性就用来关联被映射的文件。这样一来虚拟内存区域就与映射文件关联了起来。vm_pgoff 则表示映射进虚拟内存中的文件内容，在文件中的偏移。

   >  当然在匿名映射中，`vm_area_struct` 结构中 `vm_file` 为 null，`vm_pgoff` 也就没有了意义。



9. `vm_area_struct`如何寻找对应的物理内存页?

   > `vm_area_struct`结构中并没有直接的存放`Page`指针的结构体，但包含虚拟地址的起始地址和结束地址`vm_start`和`vm_end`, 通过虚拟地址转换物理地址的方法可以直接寻找到指定的`Page`.

   

10. https://www.cnblogs.com/binlovetech/p/17754173.html

    https://www.leviathan.vip/2019/01/13/mmap%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/

    内核中的mmap实现和调用流程：

    ksys_mmap_pgoff                            【/linux/mm/mmap.c】

    -> vm_mmap_pgoff                          【/linux/mm/util.c】

    -> do_mmap                                      【/linux/mm/mmap.c】

    -> mmap_region                               【/linux/mm/mmap.c】

    ![image](https://img2023.cnblogs.com/blog/2907560/202310/2907560-20231010105924697-1167793116.png)

    > mmap完整流程

    - 检查参数，并根据传入的映射类型设置`vma`的flags. 

      [do_mmap]

    - 进程查找其虚拟地址空间，找到一块空闲的满足要求的虚拟地址空间. 

      [do_mmap->get_unmapped_area]

      [在进程虚拟空间中寻找还未被映射的 VMA ，核心逻辑被内核实现在特定于体系结构的函数中]

    - 根据找到的虚拟地址空间初始化`vma`.

      [do_mmap]

    - 设置`vma->vm_file`.

      [do_mmap]

    - 根据文件系统类型，将`vma->vm_ops`设为对应的`file_operations`.

      [mmap_region]

    - 将`vma`插入`mm`的链表中.

      [mmap_region]

    > do_mmap

    根据用户传入的参数做了一系列的检查，然后根据参数初始化`vm_area_struct`的标志`vm_flags`，`vma->vm_file = get_file(file)`建立文件与`vma`的映射

    > mmap_region

    负责创建虚拟内存区域，同时调用了`call_mmap(file, vma)`: `call_mmap`根据文件系统的类型选择适配的`mmap()`函数，我们选择目前常用的`ext4`:

    `ext4_file_mmap()`是`ext4`对应的`mmap`, 功能非常简单，更新了 file 的修改时间(`file_accessed(flie))`，将对应的 operation 赋给`vma->vm_flags`:



11. 进程执行到最后一定会死亡，死亡的原因可以分为两大类，正常死亡和非正常死亡。

    正常死亡的情况： 

    > `main`函数返回
    >
    > 进程调用了`exit`、`__exit`、`__Exit` 等函数
    >
    > 进程的所有线程都调用了`pthread_exit`
    >
    > 主线程调用了`pthread_exit`，其他线程都从线程函数返回。

    非正常死亡的情况： 

    > 进程访问非法内存而收到信号`SIGSEGV`
    >
    > 库程序发现异常情况给进程发送信号`SIGABRT`
    >
    > 在终端上输入Ctrl+C给进程发送信号`SIGINT`
    >
    > 通过kill命令给进程发送信号`SIGTERM`
    >
    > 通过kill -9命令给进程发送信号`SIGKILL`
    >
    > 进程收到其它一些会导致死亡的信号

    main函数返回本质上是调用的`exit`，因为main函数外还有一层函数`__libc_start_main`，它会在main函数返回后调用`exit`，形如：

    > __libc_start_main () {
    >
    > ​	...
    >
    > ​	main();
    >
    > ​	...
    >
    > ​	exit();
    >
    > ​	...
    >
    > }

    `exit`中实质上是系统调用`exit_group`，`pthread_exit`的实现调用的是系统调用`exit`。

    这里体现出了API和系统调用的不同。进程由于信号而死，其死亡方法也是内核在信号处理中调用了系统调用`exit_group`，但是是直接调用的函数，并没有走系统调用的流程。

    系统调用`exit`是杀死线程；系统调用`exit_group`杀死当前线程，并给同进程的所有其它线程发`SIGKILL`信号，导致所有的线程死亡，从而整个进程死亡。每个线程死亡时都会释放对进程资源的引用，最后一个线程死亡时，资源的引用计数会变成0，从而会去释放这个资源。

    > 总结一下就是进程的第一个线程创建的时候会去创建进程的资源，进程的最后一个线程死亡的时候会去释放进程的资源。

    

12. 进程死亡的过程进一步细分为两个状态，僵尸进程和"彻底销毁"，分别对应进程死亡的两个子状态EXIT_ZOMBIE和EXIT_DEAD。进程“彻底销毁"之后才算是彻底死亡。

    僵尸进程并没有“彻底销毁"和"注销户口"，此时进程的各种状态信息还能查到，进程被“彻底销毁"后其相关信息”自动注销“，此时内核中相关函数、proc文件系统以及ps命令不再能查到它的信息。

    对进程来说只有父进程有“彻底销毁"子进程的权限，如果父进程一直不去“彻底销毁"子进程，那么子进程就会一直作为僵尸进程。

    父进程“彻底销毁"子进程的方法？

    > 系统调用wait、waitid、waitpid、wait3、wait4

    如果父进程提前死亡？

    > 子进程会被托孤给init进程，由init进程负责对其“彻底销毁"

    任何进程死亡都会经历僵尸进程状态，大部分情况下这个状态持续时间非常短，用户空间无法感知。当父进程没有对子进程wait，子进程就会一直处于僵尸状态，不会被“彻底销毁"。此时用户空间通过ps命令可以看到进程处于僵尸进程状态。

    僵尸进程不是没有死，而是没有完整地完成进程死亡过程，“彻底销毁"。因此杀死僵尸进程的说法并不正确。清理僵尸进程的方法是kill其父进程，父进程死亡，僵尸进程会被托孤给init进程，init进程会立马对其进行“彻底销毁"。

    进程的exit_group执行完成后，该进程即成为僵尸进程。僵尸进程没有用户空间，也不能再执行。僵尸进程的文件等所有资源都已经被释放，但仍剩余一个task_struct结构体。如果父进程此时正wait子进程或者之前就已经在wait子进程，wait会返回，task_struct会被销毁，这个进程就"彻底销毁"。