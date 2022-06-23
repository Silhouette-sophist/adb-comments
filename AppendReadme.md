针对android 12中adb模块进行注释分析原理，理解各种adb的操作

## 参考分析文档
```text
https://www.jianshu.com/p/a47e1c90b9bf
```

## adb client、adb server 和adb daemon 三模块

### 源码位置
三个模块源码都在Android系统源码中，可参考清华镜像，由同一份源码编译得到三个模块。
```text
https://aosp.tuna.tsinghua.edu.cn/platform/packages/modules/adb
```

### 运行位置

adb client和adb server运行于PC主机上
adb daemon则是运行于Android系统上，在Android系统启动时就被init进程拉起


### 运行表现
PC主机上，adb client和adb server构成CS模型。
在终端中输入adb命令，相当于adb client向adb server发送命令，如果发送期间没有adb server运行，则会直接拉起。

#### adb daemon初始化逻辑
adb daemon即adbd的入口在system/core/adb/daemon/main.cpp中，main函数获取selinux标签、banner名称、版本信息参数以及设置一些调试信息后，调用adbd_main函数：
路径：daemon/main.cpp

```c++
/**
 * adbd 启动入口，由Android系统的init进程拉起
 * 
 * @param argc 
 * @param argv 
 * @return 
 */
int main(int argc, char** argv) {
#if defined(__BIONIC__)
    // Set M_DECAY_TIME so that our allocations aren't immediately purged on free.
    mallopt(M_DECAY_TIME, 1);
#endif

    while (true) {
        static struct option opts[] = {
                {"root_seclabel", required_argument, nullptr, 's'},
                {"device_banner", required_argument, nullptr, 'b'},
                {"version", no_argument, nullptr, 'v'},
                {"logpostfsdata", no_argument, nullptr, 'l'},
        };

        int option_index = 0;
        int c = getopt_long(argc, argv, "", opts, &option_index);
        if (c == -1) {
            break;
        }

        switch (c) {
#if defined(__ANDROID__)
            case 's':
                root_seclabel = optarg;
                break;
#endif
            case 'b':
                adb_device_banner = optarg;
                break;
            case 'v':
                printf("Android Debug Bridge Daemon version %d.%d.%d\n", ADB_VERSION_MAJOR,
                       ADB_VERSION_MINOR, ADB_SERVER_VERSION);
                return 0;
            case 'l':
                LOG(ERROR) << "post-fs-data triggered";
                return 0;
            default:
                // getopt already prints "adbd: invalid option -- %c" for us.
                return 1;
        }
    }

    close_stdin();

    adb_trace_init(argv);

    D("Handling main()");
    return adbd_main(DEFAULT_ADB_PORT);
}
```

#### adb client初始化逻辑
路径：client/main.cpp
````c++
/**
 *  adb client启动入口，即在PC主机上开启终端输入adb命令后的操作
 * @param argc 
 * @param argv 
 * @param envp 
 * @return 
 */
int main(int argc, char* argv[], char* envp[]) {
    __adb_argv = const_cast<const char**>(argv);
    __adb_envp = const_cast<const char**>(envp);
    adb_trace_init(argv);
    return adb_commandline(argc - 1, const_cast<const char**>(argv + 1));
}
````


#### adb server初始化逻辑

