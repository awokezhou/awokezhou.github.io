---
title: C Program Skill
date: 2019-11-27 10:31:08
tags: [Program Skill, C]
categories:
- Program Language
- C
comments: true
---

本文记录一些实际工作中使用到的C语言编程技巧，或者学到的一些好用的用法

## kernel与application通信

### proc文件系统&mmap

kernel和application通信方式有很多，但是当数据量较大时，常用的`ioctl`、`netlink`方式并不适合，`mmap`较为适用。

#### 原理

对于应用层程序而言，系统调用`mmap()`可以将一个文件映射到内存空间，对该文件的读写就是对该块内存的读写。对于内核空间而言，proc文件系统的文件操作集`file_operations`支持`mmap`方法，在文件proc方法的具体实现中，可以将内存映射到应用层调用`mmap()`的虚拟地址上，从而实现应用层和内核空间通过proc文件关联同一块内存

#### 代码

内核空间

```c
#define MMAP_PROCFILE    "mmap_test"
#define LINUX_PAGE_SIZE 4096
#define MMAP_MEM_SIZE  (LINUX_PAGE_SIZE * 8)

static char *mmap_mem = NULL;

static int proc_mmap(struct file *filp, struct vm_area_struct *vma)  
{
    int ret;
    struct page *page = NULL;
    unsigned long size = (unsigned long)(vma->vm_end - vma->vm_start); 

    if (size > MMAP_MEM_SIZE) {  
        ret = -EINVAL;  
        goto err;  
    } 

    /* map mem block to process's address space */
    page = virt_to_page((unsigned long)mmap_mem + (vma->vm_pgoff << PAGE_SHIFT));
    ret = remap_pfn_range(vma, 
                          vma->vm_start, 
                          page_to_pfn(page), 
                          size, 
                          vma->vm_page_prot);
    if (ret)
        goto err;

    /* your operation */

    return 0;

err:
    return ret;
}

static struct file_operations proc_fops =  
{  
    .owner = THIS_MODULE,  
    .mmap = proc_mmap,  
};  

static int proc_mmap_create()
{
    /* create mem block */
    mmap_mem = kmalloc(MMAP_MEM_SIZE, GFP_KERNEL);
    if (!mmap_mem) {
        printk("kmalloc error\n");
        return -1;
    }

    /* your operation */

    /* create procfile */
    struct proc_dir_entry *proc_file = 
        proc_create(MMAP_PROCFILE, 0x0644, NULL, &proc_fops);
}
```

用户空间

```c
static int mmap_read_once()
{
    int fd;
    char *mmap_mem = NULL;
    char mmap_file[64] = {'\0'};

    sprintf(mmap_file, "/proc/%s", MMAP_PROCFILE)
    fd = open(mmap_file, O_RDWR|O_NDELAY);
    if (fd < 0) {
        log_err("open %s error", mmap_file);
        return -1;
    }

    /* do mapping */
    mmap_mem = (char *)mmap(0, 
                            MMAP_MEM_SIZE, 
                            PROT_READ | PROT_WRITE, 
                            MAP_SHARED, 
                            fd, 
                            0);
    if (!mmap_mem) {
        log_err("mmap error");
        goto out;
    }

    /* your operation */

out:
    if (mmap_mem)
        munmap(mmap_mem, MMAP_MEM_SIZE);
    if (fd > 0)
        close(fd);
    return -1;
}
```

需注意数据同步问题