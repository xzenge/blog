---
title: linux提供加载、处理动态链接库的系统调用方法
date: 2020-09-24 14:44:08
tags: 
    - Linux
    - C
categories :
    - Technology
---

我们知道目前Java提供的工具如javac、jps、jstat、jmap等都是通过Java的自举实现的。然而启动JVM的命令的java却是由C语言开发的。至于原因通过翻阅OpenJdk源码我猜测：是为了不提供JNI接口的前提下，需要动态链接调用JVM库所导致的。
<!-- more -->

### java.c中动态链接jvm库相关代码

>代码清单：src/solaris/bin/java_md_solinux.c
```C
jboolean
LoadJavaVM(const char *jvmpath, InvocationFunctions *ifn)
{
    void *libjvm;

    JLI_TraceLauncher("JVM path is %s\n", jvmpath);

    libjvm = dlopen(jvmpath, RTLD_NOW + RTLD_GLOBAL);
    if (libjvm == NULL) {
#if defined(__solaris__) && defined(__sparc) && !defined(_LP64) /* i.e. 32-bit sparc */
      FILE * fp;
      Elf32_Ehdr elf_head;
      int count;
      int location;

      fp = fopen(jvmpath, "r");
      if (fp == NULL) {
        JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
        return JNI_FALSE;
      }

      /* read in elf header */
      count = fread((void*)(&elf_head), sizeof(Elf32_Ehdr), 1, fp);
      fclose(fp);
      if (count < 1) {
        JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
        return JNI_FALSE;
      }

      /*
       * Check for running a server vm (compiled with -xarch=v8plus)
       * on a stock v8 processor.  In this case, the machine type in
       * the elf header would not be included the architecture list
       * provided by the isalist command, which is turn is gotten from
       * sysinfo.  This case cannot occur on 64-bit hardware and thus
       * does not have to be checked for in binaries with an LP64 data
       * model.
       */
      if (elf_head.e_machine == EM_SPARC32PLUS) {
        char buf[257];  /* recommended buffer size from sysinfo man
                           page */
        long length;
        char* location;

        length = sysinfo(SI_ISALIST, buf, 257);
        if (length > 0) {
            location = JLI_StrStr(buf, "sparcv8plus ");
          if (location == NULL) {
            JLI_ReportErrorMessage(JVM_ERROR3);
            return JNI_FALSE;
          }
        }
      }
#endif
        JLI_ReportErrorMessage(DLL_ERROR1, __LINE__);
        JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
        return JNI_FALSE;
    }

    ifn->CreateJavaVM = (CreateJavaVM_t)
        dlsym(libjvm, "JNI_CreateJavaVM");
    if (ifn->CreateJavaVM == NULL) {
        JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
        return JNI_FALSE;
    }

    ifn->GetDefaultJavaVMInitArgs = (GetDefaultJavaVMInitArgs_t)
        dlsym(libjvm, "JNI_GetDefaultJavaVMInitArgs");
    if (ifn->GetDefaultJavaVMInitArgs == NULL) {
        JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
        return JNI_FALSE;
    }

    ifn->GetCreatedJavaVMs = (GetCreatedJavaVMs_t)
        dlsym(libjvm, "JNI_GetCreatedJavaVMs");
    if (ifn->GetCreatedJavaVMs == NULL) {
        JLI_ReportErrorMessage(DLL_ERROR2, jvmpath, dlerror());
        return JNI_FALSE;
    }

    return JNI_TRUE;
}
```

### 总结下Linux下动态链接的使用

1. dlopen
以指定模式打开指定的动态链接库文件，并返回一个句柄给调用进程

##### 方法定义
```C
void *dlopen(const char *filename, int flag);
```

- filename 文件路径
- flag 
    1. RTLD_LAZY 暂缓决定，等有需要是再解出符号
    2. RTLD_NOW 立即决定，返回前解除所有未决定的符号
    
##### 方法使用
```c
void *handle;
handle = dlopen(动态链接库路径, RTLD_LAZY);
```

2. dlsym
通过句柄和连接符名称获取函数名或者变量名

##### 方法定义
```C
void *dlsym(void *handle, const char *symbol);
```

##### 方法使用
```C
ifn->CreateJavaVM = (CreateJavaVM_t)
        dlsym(libjvm, "JNI_CreateJavaVM");
```


3. dlclose
卸载打开的库

##### 方法定义
```C
int dlclose(void *handle);
```

##### 方法使用
```C
dlclose(handle);
```