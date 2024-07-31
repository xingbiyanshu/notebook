# Android Framework 源码分析（基于aosp-13.0.0_r83）


## 日志系统
日志系统基于CS模型，读写都是通过socket通信。


### 日志服务进程

rc文件路径：system/logging/logd/logd.rc
service logd /system/bin/logd
    socket logd stream 0666 logd logd
    socket logdr seqpacket 0666 logd logd
    socket logdw dgram+passcred 0222 logd logd
    file /proc/kmsg r
    file /dev/kmsg w
    user logd
    group logd system package_info readproc
    capabilities SYSLOG AUDIT_CONTROL
    priority 10
    task_profiles ServiceCapacityLow
    onrestart setprop logd.ready false

源码入口： system/logging/logd/main.cpp

    int main(int argc, char* argv[]) {
        // 初始化日志（自己通过Log()添加打印一定要在该语句后面）
        android::base::InitLogging(argv, [](android::base::LogId log_id, android::base::LogSeverity severity,
            const char* tag, const char* file, unsigned int line, const char* message) {
                android::base::KernelLogger(log_id, severity, "logd", file, line, message);
            }
        });
        
        // 打开内核日志文件
        static const char dev_kmsg[] = "/dev/kmsg";
        int fdDmesg = android_get_control_file(dev_kmsg); // 父进程已经打开了，尝试直接获取
        if (fdDmesg < 0) { // 从父进程获取失败则自己打开
            fdDmesg = TEMP_FAILURE_RETRY(open(dev_kmsg, O_WRONLY | O_CLOEXEC)); 
        }

        int fdPmesg = -1;
        bool klogd = GetBoolPropertyEngSvelteDefault("ro.logd.kernel");
        if (klogd) { 
            // 该属性指示是否允许内核日志通过logcat 打印
            SetProperty("ro.logd.kernel", "true");
            static const char proc_kmsg[] = "/proc/kmsg";
            fdPmesg = android_get_control_file(proc_kmsg);
            if (fdPmesg < 0) {
                fdPmesg = TEMP_FAILURE_RETRY(
                    open(proc_kmsg, O_RDONLY | O_NDELAY | O_CLOEXEC));
            }
        }
        // 是否允许selinux相关日志通过 logcat 打印。
        bool auditd = GetBoolProperty("ro.logd.auditd", true);
        DropPrivs(klogd, auditd);
        ...

        // LogReader listens on /dev/socket/logdr. When a client
        // connects, log entries in the LogBuffer are written to the client.
        // 用于处理读日志请求，如 adb shell logcat 
        LogReader* reader = new LogReader(log_buffer, &reader_list);
        reader->startListener(){
            listen(mSock, backlog);
    
            pipe2(mCtrlPipe, O_CLOEXEC);
        
            pthread_create(&mThread, nullptr, SocketListener::threadStart{
                SocketListener *me = reinterpret_cast<SocketListener *>(obj);
                me->runListener(){

                    while (true) {
                
                        poll(fds.data(), fds.size(), -1);
                
                        if (mListen && (fds[1].revents & (POLLIN | POLLERR))) {
                            accept4(mSock, nullptr, nullptr, SOCK_CLOEXEC);
                            mClients[c] = new SocketClient(c, true, mUseCmdNum);
                        }
                        
                        for (int i = mListen ? 2 : 1; i < fds.size(); ++i) {
                            const struct pollfd& p = fds[i];
                            if (p.revents & (POLLIN | POLLERR)) {
                                auto it = mClients.find(p.fd);
                                SocketClient* c = it->second;
                                pending.push_back(c);
                            }
                        }
                
                        for (SocketClient* c : pending) {
                            onDataAvailable(c){
                                // TODO
                            }
                        }
                    }

                }
            }, this);

        }

        // LogListener listens on /dev/socket/logdw for client
        // initiated log messages. New log entries are added to LogBuffer
        // and LogReader is notified to send updates to connected clients.
        // 用于处理写日志请求，如android.util.Log.i("Hello")，
        // 是udp方式，rc文件有定义“socket logdw dgram+passcred 0222 logd logd”，dgram对应udp，所以没有listen的动作。
        LogListener* swl = new LogListener(log_buffer){
            LogListener::LogListener(LogBuffer* buf) : socket_(GetLogSocket()), logbuf_(buf) {}

            int LogListener::GetLogSocket() {
                static const char socketName[] = "logdw";
                int sock = android_get_control_socket(socketName); 
                /* init进程会为service创建service在rc中定义的socket、file等资源。所以此处尝试获取init进程替它创建的（会继承父进程的）
                logcat的输出：tag可以看到这些socket都是init进程创建的
                03-14 11:22:05.658 I/init    (    0): starting service 'logd'...  
                03-14 11:22:05.658 I/init    (    0): Created socket '/dev/socket/logd', mode 666, user 1036, group 1036
                03-14 11:22:05.659 I/init    (    0): Created socket '/dev/socket/logdr', mode 666, user 1036, group 1036
                03-14 11:22:05.659 I/init    (    0): Created socket '/dev/socket/logdw', mode 222, user 1036, group 1036
                ...
                // logd入口打印：
                03-14 11:22:05.718 I/logd    (    0): =#=logd start...
                03-14 11:22:05.719 I/logd    (    0): arg[0]=/system/bin/logd
                */  
                if (sock < 0) {  // logd started up in init.sh
                    sock = socket_local_server(socketName, ANDROID_SOCKET_NAMESPACE_RESERVED, SOCK_DGRAM);
                    int on = 1;
                    setsockopt(sock, SOL_SOCKET, SO_PASSCRED, &on, sizeof(on));/*通过设置 SO_PASSCRED，能够接收一个 SCM_CREDENTIALS 消息，
                                                                                消息中包含发送者的 pid, uid, 和 gid。该消息通过 struct ucred 结构返回。
                                                                                通过这个选项，我们就能够知道是谁写入了 log。*/
                }
                return sock;
            }

        }
        swl->StartListener(){ 
            std::thread(&LogListener::ThreadFunction{ // 开辟新线程处理
                while (true) { // 死循环处理请求
                    HandleData(){
                        // 阻塞等待用户发送（写）日志。因为是udp,所以直接recv,没有listen的动作。
                        ssize_t n = recvmsg(socket_, &hdr, 0);
                        ...
                        // 将用户发过来的日志写入logbuf
                        logbuf_->Log(logId, header->realtime, cred->uid, cred->pid, header->tid, msg,
                            ((size_t)n <= UINT16_MAX) ? (uint16_t)n : UINT16_MAX){
                            auto entry = LogToLogBuffer(logs_[log_id], max_size_[log_id], sequence, realtime, uid, pid, tid,
                            msg, len);
                            // 通知reader有更新，需要重新读取（Log.i写入，当前开着的logcat能实时刷新）                            
                            reader_list_->NotifyNewLog(1 << log_id);
                        }
                    }
                }
            },this);
        }

        // Command listener listens on /dev/socket/logd for incoming logd
        // administrative commands.
        // 负责处理发送过来的控制命令，比如logcat控制命令。
        CommandListener* cl = new CommandListener(log_buffer, &log_tags, &prune_list, &log_statistics);
        cl->startListener()


    }


### 客户写日志（如App调用android.os.Log接口写日志）

// file: android.util.Log.java

    i(@Nullable String tag, @NonNull String msg) {
        println_native(LOG_ID_MAIN, INFO, tag, msg){
            // file: frameworks/base/core/jni/android_util_Log.cpp
            static jint android_util_Log_println_native {
                // file: system/logging/liblog/logger_write.cpp
                __android_log_buf_write(bufID, (android_LogPriority)priority, tag, msg){
                    __android_log_write_log_message(&log_message){
                        logger_function(log_message); // logger_function是一个函数指针，默认是__android_log_logd_logger
                                                    // static __android_logger_function logger_function = __android_log_logd_logger;
                        __android_log_logd_logger{
                            write_to_log(static_cast<log_id_t>(buffer_id), vec, 3){
                                ...
                                check_log_uid_permissions(); // 权限校验
                                LogdWrite(log_id, &ts, vec, nr){
                                    // file : system/logging/liblog/logd_writer.cpp
                                    LogdSocket& logd_socket = logId == LOG_ID_SECURITY ? LogdSocket::BlockingSocket() : LogdSocket::NonBlockingSocket();
                                    writev(logd_socket.sock(), newVec, i); // 向属性服务的监听socket发送写日志请求（udp方式，rc文件中有定义“socket logdw dgram+passcred 0222 logd logd”）
                                }
                            }
                        }


### Init过程logcat输出分析

#### 内核启动init进程（Run /init as init process）
03-14 11:22:02.590 I/        (    0): Run /init as init process
03-14 11:22:02.590 D/with arguments(    0):  
03-14 11:22:02.590 D/        (    0): /init
03-14 11:22:02.590 D/with environment(    0):  
03-14 11:22:02.590 D/        (    0): HOME=/
03-14 11:22:02.590 D/        (    0): TERM=linux
03-14 11:22:02.591 I/init    (    0): init first stage started!

#### 加载modules（Loading module）
03-14 11:22:02.591 I/init    (    0): init first stage started!
03-14 11:22:02.591 I/init    (    0): Loading module /lib/modules/virtio_blk.ko with args ''
03-14 11:22:02.592 I/virtio_blk(    0): module verification failed: signature and/or required key missing - tainting kernel
03-14 11:22:02.606 I/init    (    0): Loaded kernel module /lib/modules/virtio_blk.ko
03-14 11:22:02.606 I/init    (    0): Loading module /lib/modules/virtio_console.ko with args ''
...

#### 加载系统属性
03-14 11:22:05.155 I/init    (    0): init second stage started!
03-14 11:22:05.169 I/init    (    0): Using Android DT directory /proc/device-tree/firmware/android/
03-14 11:22:05.171 W/init    (    0): Couldn't load property file '/vendor/default.prop': open() failed: No such file or directory: No such file or directory
03-14 11:22:05.172 W/init    (    0): Overriding previous property 'ro.control_privapp_permissions':'disable' with new value 'enforce'
03-14 11:22:05.175 W/init    (    0): Couldn't load property file '/odm_dlkm/etc/build.prop': open() failed: No such file or directory: No such file or directory
03-14 11:22:05.177 W/init    (    0): Overriding previous property 'ro.config.notification_sound':'OnTheHunt.ogg' with new value 'pixiedust.ogg'
03-14 11:22:05.178 I/init    (    0): Setting product property ro.product.brand to 'Android' (from ro.product.product.brand)
03-14 11:22:05.179 I/init    (    0): Setting product property ro.product.device to 'emulator_x86_64' (from ro.product.product.device)
03-14 11:22:05.179 I/init    (    0): Setting product property ro.product.manufacturer to 'unknown' (from ro.product.product.manufacturer)
03-14 11:22:05.180 I/init    (    0): Setting product property ro.product.model to 'Android SDK built for x86_64' (from ro.product.product.model)
03-14 11:22:05.180 I/init    (    0): Setting product property ro.product.name to 'sdk_phone_x86_64' (from ro.product.product.name)
03-14 11:22:05.181 I/init    (    0): Setting property 'ro.build.fingerprint' to 'Android/sdk_phone_x86_64/emulator_x86_64:13/TQ3A.230805.001.S1/eng.sissi.20240227.110612:eng/test-keys'
03-14 11:22:05.182 I/init    (    0): Setting property 'ro.product.cpu.abilist' to 'x86_64,x86'
03-14 11:22:05.182 I/init    (    0): Setting property 'ro.product.cpu.abilist32' to 'x86'
03-14 11:22:05.182 I/init    (    0): Setting property 'ro.product.cpu.abilist64' to 'x86_64'

#### 解析rc文件（Parsing file）
03-14 11:22:05.189 I/init    (    0): Parsing file /system/etc/init/hw/init.rc...
03-14 11:22:05.191 I/init    (    0): Added '/init.environ.rc' to import list
03-14 11:22:05.191 I/init    (    0): Added '/system/etc/init/hw/init.usb.rc' to import list
03-14 11:22:05.191 I/init    (    0): Added '/init.ranchu.rc' to import list

#### 执行action（processing action）
action开始执行表示已经进入init进程主循环；
service由action触发；
03-14 11:22:05.363 I/init    (    0): processing action (SetKptrRestrict) from (<Builtin Action>:0)
03-14 11:22:05.364 I/init    (    0): processing action (TestPerfEventSelinux) from (<Builtin Action>:0)
03-14 11:22:05.365 I/init    (    0): processing action (ConnectEarlyStageSnapuserd) from (<Builtin Action>:0)
03-14 11:22:05.366 I/init    (    0): processing action (early-init) from (/system/etc/init/hw/init.rc:15)

#### 启动服务进程（starting service）
03-14 11:22:05.658 I/init    (    0): starting service 'logd'...
03-14 11:22:05.658 I/init    (    0): Created socket '/dev/socket/logd', mode 666, user 1036, group 1036
03-14 11:22:05.659 I/init    (    0): Created socket '/dev/socket/logdr', mode 666, user 1036, group 1036
03-14 11:22:05.659 I/init    (    0): Created socket '/dev/socket/logdw', mode 222, user 1036, group 1036

#### 服务进程退出（Service.*exited）
03-14 11:22:05.536 I/init    (    0): Service 'apexd-bootstrap' (pid 160) exited with status 0 waiting took 0.082000 seconds
03-14 11:22:05.537 I/init    (    0): Sending signal 9 to service 'apexd-bootstrap' (pid 160) process group...

#### Java进程启动
03-14 11:22:08.752 D/AndroidRuntime(  356): >>>>>> START com.android.internal.os.ZygoteInit uid 0 <<<<<<



## Init进程
内核启动init进程
logcat相关内容：
03-14 11:22:02.590 I/        (    0): Run /init as init process // 内核加载完毕启动init进程
03-14 11:22:02.590 D/with arguments(    0):  
03-14 11:22:02.590 D/        (    0): /init
03-14 11:22:02.590 D/with environment(    0):  
03-14 11:22:02.590 D/        (    0): HOME=/
03-14 11:22:02.590 D/        (    0): TERM=linux
03-14 11:22:02.591 I/init    (    0): init first stage started!

开机->bootloader->内核->init进程(用户空间1号进程，进入system/core/init/main.cpp执行main方法){<br>

### 源码入口 system/core/init/main.cpp
    int main(int argc, char** argv) {
        // Boost prio which will be restored later
        setpriority(PRIO_PROCESS, 0, -20);
        if (!strcmp(basename(argv[0]), "ueventd")) {
            // ueventd服务负责设备节点的管理。
            return ueventd_main(argc, argv); 

            // ueventd进程和init进程共用一个init可执行文件。所以代码入口同init
            emulator_x86_64:/ # ls -l /system/bin/|grep init                 
            -rwxr-xr-x 1 root shell 2372016 2024-03-08 14:11 init
            lrwxr-xr-x 1 root shell       4 2024-03-08 14:11 ueventd -> init
            // 为最早起来的服务进程：/system/etc/init/hw/init.rc
            on early-init
                ...
                start ueventd
            ...
            service ueventd /system/bin/ueventd
                class core
                critical
                seclabel u:r:ueventd:s0
                shutdown critical
        }
    
        if (argc > 1) {
            if (!strcmp(argv[1], "subcontext")) {
                android::base::InitLogging(argv, &android::base::KernelLogger);
                const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();
    
                return SubcontextMain(argc, argv, &function_map);
            }
    
            if (!strcmp(argv[1], "selinux_setup")) {
                return SetupSelinux(argv);
            }
    
            if (!strcmp(argv[1], "second_stage")) {
                return SecondStageMain(argc, argv);
            }
        }
    
        return FirstStageMain(argc, argv);
    }


###      FirstStageMain{  
         // 创建及挂载文件系统
         CHECKCALL(mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755")); // 挂载tmpfs文件系统
         CHECKCALL(mkdir("/dev/socket", 0755));// 创建dev/socket设备节点
         CHECKCALL(mount("devpts", "/dev/pts", "devpts", 0, NULL));// 挂载devpts文件系统
         CHECKCALL(mount("sysfs", "/sys", "sysfs", 0, NULL));// 挂载sysfs文件系统
         CHECKCALL(mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11)));// 提前创建了kmsg设备节点文件，用于输出log信息
         ...
      
         // 重定向STDIN/OUT/ERROR到/dev/null，所以我们看不到printf的console输出。
         // 这么做的原因猜测是为了节省开销，因为android有自己的日志系统，所有日志都保存到相应的缓存，
         // 在此前提下再输出到标准输出就显得有些浪费了。
         SetStdioToDevNull(argv);

         // ？干嘛用的？
         // 上面已经 mount("tmpfs", "/dev"), mknod("/dev/kmsg")
         // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually
         // talk to the outside world...
         InitKernelLogging(argv);

         ...
         // ???
         LoadKernelModules()

         ...

         // 通过execv替换当前进程执行镜像，进入init下一阶段
         const char* path = "/system/bin/init";
         const char* args[] = {path, "selinux_setup", nullptr}; 
         // selinux_setup本身内容省略不予分析，最终同样通过execv进入到SecondStageMain
         execv(path, const_cast<char**>(args));
      }
   

###      SecondStageMain{  
        if (REBOOT_BOOTLOADER_ON_PANIC) {
            InstallRebootSignalHandlers();
        }

        // 重启/关机事件的处理。如adb shell输入reboot/shutdown
        trigger_shutdown = [](const std::string &command) {
            shutdown_state.TriggerShutdown(command);
        };

         ...
         SetStdioToDevNull(argv);
         InitKernelLogging(argv);
         ...
         
         
####         PropertyInit(){ //属性初始化
            ...
            // 把各个分区的属性 SeLinux 配置文件解析后加载进内存，序列化处理后，存入 /dev/__properties__/property_info 文件中。
             CreateSerializedPropertyInfo(); 

            /*
            从 /dev/__properties__/property_info 文件中加载信息
            根据加载到的信息，构建出一个 ContextNode 数组
            在 /dev/__properties__ 目录下，创建属性文件
            */
             __system_property_area_init(){
               system_properties.AreaInit(PROP_FILENAME, &fsetxattr_failed){
                     ContextsSerialized::Initialize(bool writable, const char* filename, bool* fsetxattr_failed) {
                        ContextsSerialized::InitializeProperties() {
                           property_info_area_file_.LoadDefaultPath(){
                              LoadPath("/dev/__properties__/property_info"){
                                 ...
                                 // mmap 映射/dev/__properties__/property_info文件到内存。
                                 // 这样其他进程可以直接读取属性值（不可以直接写，写需要和properties server交互）
                                 void* map_result = mmap(nullptr, mmap_size, PROT_READ, MAP_SHARED, fd, 0);

                                 // 将映射区内存地址强转为管理类 PropertyInfoArea 的指针，通过这个指针就可以读写 /dev/__properties__/property_info 文件中的数据了
                                 auto property_info_area = reinterpret_cast<PropertyInfoArea*>(map_result);
                                 ...
                                 mmap_base_ = map_result; // 把映射区地址保存到成员变量 mmap_base_ 中
                                 ...
                              }
                           }

                        }

                       InitializeContextNodes(){
                        }

                        MapSerialPropertyArea(true, fsetxattr_failed){

                        }
                     }
                  }
               }
             }

            ...
         }
         ...


####         InstallSignalFdHandler(&epoll){ // 监听子进程退出信号。
            sigaddset(&mask, SIGCHLD); // 监听子进程退出信号
            sigaddset(&mask, SIGTERM);
            signal_fd = signalfd(-1, &mask, SFD_CLOEXEC | SFD_NONBLOCK);
            constexpr int flags = EPOLLIN | EPOLLPRI;
            auto handler = std::bind(HandleSignalFd, false);
            epoll->RegisterHandler(signal_fd, handler, flags);
            // HandleSignalFd{
                read(signal_fd, &siginfo, sizeof(siginfo));
                switch (siginfo.ssi_signo) {
                    case SIGCHLD:
                        ReapAnyOutstandingChildren(){
                            ReapOneProcess{
                                waitid(P_ALL, 0, &siginfo, WEXITED | WNOHANG | WNOWAIT);
                                auto pid = siginfo.si_pid;
                                waitpid(pid, nullptr, WNOHANG);

                                if (SubcontextChildReap(pid)) {
                                    name = "Subcontext";
                                } else {
                                    service = ServiceList::GetInstance().FindService(pid, &Service::pid);
                                    service->Reap(siginfo){
                                        if (!(flags_ & SVC_ONESHOT) || (flags_ & SVC_RESTART)) {
                                            KillProcessGroup(SIGKILL, false); // kill掉退出的子进程的所有子进程。
                                        }
                                        
                                        if (flags_ & SVC_EXEC) UnSetExec(){
                                            is_exec_service_running_ = false;   // init主循环中会判断这个标志
                                            flags_ &= ~SVC_EXEC;
                                        }
                                        
                                        if (name_ == "zygote" || name_ == "zygote64") {
                                            removeAllEmptyProcessGroups();
                                        }
                                        
                                        flags_ &= (~SVC_RUNNING);
                                    
                                        // Oneshot processes go into the disabled state on exit,
                                        // except when manually restarted.
                                        if ((flags_ & SVC_ONESHOT) && !(flags_ & SVC_RESTART) && !(flags_ & SVC_RESET)) {
                                            flags_ |= SVC_DISABLED;
                                        }
                                    
                                        // Disabled and reset processes do not get restarted automatically.
                                        if (flags_ & (SVC_DISABLED | SVC_RESET))  {
                                            NotifyStateChange("stopped");
                                            return;
                                        }
                                        ...
                                        
                                        flags_ |= SVC_RESTARTING;
                                    
                                        // Execute all onrestart commands for this service.
                                        onrestart_.ExecuteAllCommands();
                                    
                                        NotifyStateChange("restarting"){
                                            std::string prop_name = "init.svc." + name_; 
                                            SetProperty(prop_name, new_state){
                                                __system_property_set(key.c_str(), value.c_str()){
                                                    PropertyServiceConnection connection;
                                                    SocketWriter writer(&connection);
                                                    writer.WriteUint32(PROP_MSG_SETPROP2).WriteString(key).WriteString(value).Send(); // 向property server发请求
                                                    connection.RecvInt32(&result); // 等待请求结果
                                                }
                                            }
                                            // getprop命令可以查看service状态：
                                            emulator_x86_64:/ # getprop
                                            [init.svc.adbd]: [running]
                                            ...
                                            [init.svc.audioserver]: [running]
                                            ...
                                        }
                                    }
                                }
                            }
                        }
                        break;
                    case SIGTERM:
                        HandleSigtermSignal(siginfo);
                        break;
                }
            }
         }

         // init进程在做完所有工作后会陷入死循环。每次循环依次抽取action执行后陷入epoll.wait等待
         // 该函数提供一个init主循环在epoll.wait的时候，可被唤醒的接口
####         InstallInitNotifier(&epoll){ // Init通知器。各种事件可通过该通知器通知Init进程处理
            wake_main_thread_fd = eventfd(0, EFD_CLOEXEC);   // WakeMainInitThread会向该fd写入，fd有可读事件会唤醒epoll，进而驱动init循环。
            auto clear_eventfd = [] {
                uint64_t counter;
                TEMP_FAILURE_RETRY(read(wake_main_thread_fd, &counter, sizeof(counter))); // 读取的内容无所谓，主要是唤醒
            };

            epoll->RegisterHandler(wake_main_thread_fd, clear_eventfd); 
         }

         
####         StartPropertyService(&property_fd){ // 启动属性服务，处理设置属性请求
             InitPropertySet("ro.property_service.version", "2");
         
             int sockets[2];
             if (socketpair(AF_UNIX, SOCK_SEQPACKET | SOCK_CLOEXEC, 0, sockets) != 0) {
                 PLOG(FATAL) << "Failed to socketpair() between property_service and init";
             }
             *epoll_socket = from_init_socket = sockets[0];
             init_socket = sockets[1];
         
             StartSendingMessages(){accept_messages=true}
         
             property_set_fd = CreateSocket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK,
                                            /*passcred=*/false, /*should_listen=*/false, 0666, /*uid=*/0,
                                            /*gid=*/0, /*socketcon=*/{});
         
             listen(property_set_fd, 8); // 通过socket监听属性设置事件。
         
            // 开辟新线程（仍在init进程中）处理属性设置请求
             auto new_thread = std::thread{PropertyServiceThread}{
                   Epoll epoll;
                  ...
                  epoll.RegisterHandler(property_set_fd, handle_property_set_fd{
                     ...
                    socket.RecvString(&name, &timeout_ms);
                    socket.RecvString(&value, &timeout_ms);
            
                    socket.GetSourceContext(&source_context);

                    uint32_t result = HandlePropertySet(name, value, source_context, cr, &socket, &error){
                     // 校验属性名及值合法性，校验设置者权限
                      CheckPermissions(name, value, source_context, cr, error); ret != PROP_SUCCESS);
                  
                      if (StartsWith(name, "ctl.")) {
                          return SendControlMessage(name.c_str() + 4, value, cr.pid, socket, error);
                      }
                  
                      return PropertySet(name, value, error){
                         prop_info* pi = (prop_info*) __system_property_find(name.c_str());
                         if (pi != nullptr) {  // 有就更新值
                             // ro.* properties are actually "write-once".
                             if (StartsWith(name, "ro.")) { // 只读的不能更新
                                 *error = "Read-only property was already set";
                                 return PROP_ERROR_READ_ONLY_PROPERTY;
                             }
                             __system_property_update(pi, value.c_str(), valuelen){
                                 SystemProperties::Update(prop_info* pi, const char* value, unsigned int len) {
                                    // TODO
                                 }
                              }
                         } else { // 没有就先添加
                             __system_property_add(name.c_str(), name.size(), value.c_str(), valuelen);
                         }
                     
                         // 持久化的属性设置写入文件
                         if (persistent_properties_loaded && StartsWith(name, "persist.")) {
                             WritePersistentProperty(name, value); 
                         }
                     
                         auto lock = std::lock_guard{accept_messages_lock};
                         if (accept_messages) {
                              // 通知init主循环属性已经更改（所有期待该属性值的action可以执行了）
                             PropertyChanged(name, value){
                                 CheckAndResetWait{
                                     if (waiting_for_prop_) {
                                          ResetWaitForPropLocked(){  waiting_for_prop_.reset(); } // waiting_for_prop_置为false
                                          WakeMainInitThread(){
                                             write(wake_main_thread_fd, &counter, sizeof(counter)); // 参见InstallInitNotifier()
                                          }
                                     }
                                 }
                              }
                         }
                        }
                     }

                    socket.SendUint32(result);

                  })
               
                   epoll.RegisterHandler(init_socket, HandleInitSocket); // init进程自身的属性加载等事件监听，如：{
                                                                              do_load_persist_props{
                                                                                void SendLoadPersistentPropertiesMessage() {
                                                                                    SendMessage(property_fd, init_message); // property_fd是前面创建的socketpair的[0]，init_socket是[1]，写其中一个，另一个就能读了
                                                                                }
                                                                                 // 等待该属性被设置（waiting_for_prop_会被reset，init的主循环中也会判断该waiting_for_prop_）
                                                                                 start_waiting_for_property("ro.persistent_properties.ready", "true");
                                                                              }
                                                                           }

                                                                           HandleInitSocket{
                                                                               auto message = ReadMessage(init_socket);
                                                                               switch (init_message.msg_case()) {
                                                                                   case InitMessage::kLoadPersistentProperties: {
                                                                                       load_override_properties();
                                                                                       // Read persistent properties after all default values have been loaded.
                                                                                       auto persistent_properties = LoadPersistentProperties();
                                                                                       for (const auto& persistent_property_record : persistent_properties.properties()) {
                                                                                           InitPropertySet(persistent_property_record.name(),
                                                                                                           persistent_property_record.value());
                                                                                       }
                                                                                       InitPropertySet("ro.persistent_properties.ready", "true"); // 上面在等待该属性被设置
                                                                                       persistent_properties_loaded = true;
                                                                                       break;
                                                                                   }
                                                                               }                                                                           
                                                                           }
               
                   while (true) { // 循环等待属性事件
                       auto pending_functions = epoll.Wait(std::nullopt);
                        for (const auto& function : *pending_functions) {
                            (*function)(); // handle_property_set_fd || HandleInitSocket
                        }
                   }
                    
                    /*客户端设置属性流程：
                    file: android.os.SystemProperties.java
                    SystemProperties::set(@NonNull String key, @Nullable String val) {
                        native_set{
                            file: frameworks/base/core/jni/android_os_SystemProperties.cpp（jni函数用的是动态注册的方式，可以以native方法名"native_set"为关键字搜索）
                            void SystemProperties_set(JNIEnv *env, jobject clazz, jstring keyJ, jstring valJ){
                                __system_property_set(key.c_str(), value ? value->c_str() : ""){
                                    PropertyServiceConnection connection;
                                    SocketWriter writer(&connection);
                                    writer.WriteUint32(PROP_MSG_SETPROP2).WriteString(key).WriteString(value).Send(); // 向property server发请求
                                    connection.RecvInt32(&result); // 等待请求结果
                                }
                            }
                        }
                    }
                    */
            }
         }

         ...

         // 设置初始化脚本命令到函数的映射
         const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap(){
             static const BuiltinFunctionMap builtin_functions = {
                                                          /*这列指示是否在subcontext进程执行*/
                 {"bootchart",               {1,     1,    {false,  do_bootchart}}},
                 {"chmod",                   {2,     2,    {true,   do_chmod}}},
                 {"chown",                   {2,     3,    {true,   do_chown}}},
                 {"class_reset",             {1,     1,    {false,  do_class_reset}}},
                 {"class_restart",           {1,     2,    {false,  do_class_restart}}},
                 {"class_start",             {1,     1,    {false,  do_class_start}}},
                 {"class_stop",              {1,     1,    {false,  do_class_stop}}},
                 {"copy",                    {2,     2,    {true,   do_copy}}},
                 {"copy_per_line",           {2,     2,    {true,   do_copy_per_line}}},
                 {"domainname",              {1,     1,    {true,   do_domainname}}},
                 ...
                 {"symlink",                 {2,     2,    {true,   do_symlink}}},
                 {"sysclktz",                {1,     1,    {false,  do_sysclktz}}},
                 {"trigger",                 {1,     1,    {false,  do_trigger}}},
                 ...
                 {"mount_all",               {0,     kMax, {false,  do_mount_all}}}, 
#####                  /*zygote触发流程
                     file: /system/etc/init/hw/init.rc
                     import /vendor/etc/init/hw/init.${ro.hardware}.rc
                     ...
                     on late-init
                        trigger early-fs
                     
                         # Mount fstab in init.{$device}.rc by mount_all command. Optional parameter
                         # '--early' can be specified to skip entries with 'latemount'.
                         # /system and /vendor must be mounted by the end of the fs stage,
                         # while /data is optional.
                         trigger fs
                         trigger post-fs

                     file: /vendor/etc/init/hw/init.ranchu.rc  // init.rc中引入了该文件：import /vendor/etc/init/hw/init.${ro.hardware}.rc
                     on fs
                     mount_all /vendor/etc/fstab.ranchu --early
                     
                     on late-fs
                     # Mount RW partitions which need run fsck
                     mount_all /vendor/etc/fstab.ranchu --late

                     // do_mount_all最终会触发启动zygote。do_mount_all{
                        queue_fs_event{
                           ...
                           ActionManager::GetInstance().QueueEventTrigger("nonencrypted"); // init.rc中有： 
                                                                                             on nonencrypted
                                                                                                 class_start main // 会导致class为main的service启动，其中包括zygote
                                                                                                 class_start late_start
         
                        }
                     }
                  */


             // 上面每个关键字都对应rc文件中action节下面的一个command,比如：
               action语法：
               on <trigger>  [&& <trigger>]*  # 触发器，分event triggers 和property triggers. event可以由trigger command触发或者QueueEventTrigger函数。
                  <command1>  ##执行命令
                  <command2>  
                  ...

              file: /system/etc/init/hw/init.rc
              on init
                sysclktz 0
                copy /proc/cmdline /dev/urandom
                copy /system/etc/prop.default /dev/urandom
                symlink /proc/self/fd/0 /dev/stdin
                symlink /proc/self/fd/1 /dev/stdout
                symlink /proc/self/fd/2 /dev/stderr
                ...
                start logd // 启动日志服务进程

         }
         Action::set_function_map(&function_map);
         ...

         // 初始化init的子进程Subcontext，init会将某些命令丢给它执行
         InitializeSubcontext();

         // 解析android初始化脚本
         ActionManager& am = ActionManager::GetInstance();
         ServiceList& sm = ServiceList::GetInstance();
####         LoadBootScripts(am, sm){ // 解析android初始化脚本，创建Service、Action队列
            Parser parser = CreateParser(am, sm){
               Parser parser;
               // 创建init.rc文件里面的两种section service,on(action)，以及import语句，对应的解析器
               parser.AddSectionParser("service", ServiceParser()); 
               parser.AddSectionParser("on", ActionParser());
               parser.AddSectionParser("import", ImportParser());
            }
            ...
            parser.ParseConfig("/system/etc/init/hw/init.rc"){
               ...
               // 初始化脚本被解析为state结构体了，循环处理该结构体
               for (;;) {
                 switch (next_token(&state)) {
                     ...
                     case T_NEWLINE: {
                        ...
                        // 开始下一个section解析前先结束当前section解析
                        section_parser->endSection(){
                           // 解析好的action添加到action manager
                           void ActionManager::AddAction(std::unique_ptr<Action> action) {
                               actions_.emplace_back(std::move(action));
                           }
                           // 解析好的service添加到service_list
                           service_list_->AddService(std::move(service_));
                        }

                        section_parser("service"|"on"|"import")->ParseSection(std::move(args), filename, state.line){
                           ActionParser|ServiceParser::ParseSection{
                              // 创建Action（保存在Parser中，最终保存到action manager）
                              std::make_unique<Action>(false, action_subcontext, filename, line, event_trigger,
                                                                property_triggers);
                              // 创建Service（保存在Parser中，最终保存到service list）
                              std::make_unique<Service>(name, restart_action_subcontext, str_args, from_apex_);
                           }
                        }
                        ...
                        "action_parser|service_parser"->ParseLineSection(std::move(args), state.line){
                           action_->AddCommand(std::move(args), line) {
                               auto map_result = function_map_->Find(args); // function_map_ 即为前面创建的BuiltinFunctionMap
                              // 根据参数找到对应的function构建一个command
                               commands_.emplace_back(map_result->function, map_result->run_in_subcontext, std::move(args),line);
                           }
                           ServiceParser::ParseLineSection(std::vector<std::string>&& args, int line) {
                               parserMap = GetParserMap(){
                                     static const KeywordMap<ServiceParser::OptionParser> parser_map = {
                                          // 每一个函数都是一个解析器
                                          ...
                                         {"class",                   {1,     kMax, &ServiceParser::ParseClass}},
                                         {"critical",                {0,     2,    &ServiceParser::ParseCritical}},
                                         {"disabled",                {0,     0,    &ServiceParser::ParseDisabled}},
                                         {"group",                   {1,     NR_SVC_SUPP_GIDS + 1, &ServiceParser::ParseGroup}},
                                         {"oneshot",                 {0,     0,    &ServiceParser::ParseOneshot}},
                                         {"onrestart",               {1,     kMax, &ServiceParser::ParseOnrestart}},
                                         {"oom_score_adjust",        {1,     1,    &ServiceParser::ParseOomScoreAdjust}},
                                         {"priority",                {1,     1,    &ServiceParser::ParsePriority}},
                                         {"socket",                  {3,     6,    &ServiceParser::ParseSocket}},
                                          ...

                                     // 上面每个关键字都对应rc文件中service节下面的一个option,比如：
                                       service语法：
                                       service <name> <pathname> [ <argument> ]*
                                          <option>
                                          <option>
                                          ...
                        
                                      file: /system/etc/init/hw/init.zygote64_32.rc
                                       service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
                                       class main
                                       priority -20
                                       user root
                                       group root readproc reserved_disk
                                       socket zygote stream 660 root system
                                       socket usap_pool_primary stream 660 root system
                                       onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
                                       onrestart write /sys/power/state on
                                       onrestart restart audioserver
                                       onrestart restart cameraserver
                                       onrestart restart media
                                       onrestart restart media.tuner
                                       onrestart restart netd
                                       onrestart restart wificond
                                       task_profiles ProcessCapacityHigh MaxPerformance
                                       critical window=${zygote.critical_window.minute:-off} target=zygote-fatal

                               }

                               auto parseFunc = parserMap.Find(args);
                               return std::invoke(*parseFunc, this, std::move(args)); // 解析service的option并把相应结果保存到service
                           }
                        }
                     }
                     ...
                 }
               }
            }
            parser.ParseConfig("/xxx/init.rc");
            ...
         }

         // 构建一些内置action并添加到action队列
         am.QueueBuiltinAction(SetupCgroupsAction,/*command func*/ "SetupCgroups"/*command name*/){
             auto action = std::make_unique<Action>(true, nullptr, "<Builtin Action>", 0, name,
                                                    std::map<std::string, std::string>{});
             action->AddCommand(std::move(func), {name}, 0);
             event_queue_.emplace(action.get());
             actions_.emplace_back(std::move(action)); // 添加到action manager
         }
         ...

         // ===== 添加事件触发器 "early-init"
         am.QueueEventTrigger("early-init"){
            event_queue_.emplace("early-init");
         }
        am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done"){
            wait_for_coldboot_done_action{
                prop_waiter_state.StartWaiting() // 后面的prop_waiter_state.MightBeWaiting()会为true
            }
        }
         ...
         
         am.QueueEventTrigger("init"); // ===== 添加事件触发器 "init"{
                                             ...
                                              # Start essential services.
                                              start servicemanager // servicemanager在zygote之前？？？
                                             ...
                                          }
         ...
         am.QueueEventTrigger("late-init");  // ===== 添加事件触发器 "late-init"。
         /* // file: /system/etc/init/hw/init.rc
            on late-init
                trigger early-fs
                trigger fs
                trigger post-fs
                trigger late-fs
                trigger post-fs-data
                trigger load_bpf_programs
            
                # Now we can start zygote for devices with file based encryption
                trigger zygote-start  //======== 触发“zygote-start”事件{
                                       on zygote-start && property:ro.crypto.state=encrypted && property:ro.crypto.type=file
                                           wait_for_prop odsign.verification.done 1
                                           # A/B update verifier that marks a successful boot.
                                           exec_start update_verifier_nonencrypted
                                           start statsd
                                           start netd
                                           start zygote //======= 启动zygote服务进程{
                                                         // file: /system/etc/init/hw/init.zygote64_32.rc
                                                         service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
                                                            class main
                                                            priority -20
                                                            user root
                                                            group root readproc reserved_disk
                                                            socket zygote stream 660 root system
                                                            socket usap_pool_primary stream 660 root system
                                                            onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
                                                            onrestart write /sys/power/state on
                                                            onrestart restart audioserver
                                                            onrestart restart cameraserver
                                                            onrestart restart media
                                                            onrestart restart media.tuner
                                                            onrestart restart netd
                                                            onrestart restart wificond
                                                            task_profiles ProcessCapacityHigh MaxPerformance
                                                            critical window=${zygote.critical_window.minute:-off} target=zygote-fatal
  
                                                         }
                                                        /*查找service对应的源码：
                                                        1、取出二进制文件名“app_process”（64或32后缀剔除）
                                                        2、ctrl+shift+f 全局搜索该二进制文件名，过滤文件类型*.bp
                                                        3、搜出来的结果若有多个，注意挑选cmds目录下的，cc_binary上下文的：
                                                            cc_binary {
                                                            name: "app_process",
                                                            srcs: ["app_main.cpp"], 
                                                        4、其中的srcs字段即指明了源文件路径
                                                        */
                                           start zygote_secondary
                                       }
            
                trigger firmware_mounts_complete
            
                trigger early-boot
                trigger boot

            
                on nonencrypted
                    class_start main   // 这里也能触发zygote的启动，但是会落在后面，前面已经启动
                    // logcat输出的日志：
                    03-14 11:22:09.004 I/init    (    0): processing action (nonencrypted) from (/system/etc/init/hw/init.rc:1184)
                    ...
                    03-14 11:22:09.015 I/init    (    0): service 'zygote' requested start, but it is already running (flags: 4)
         */

         am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");


         
####         while(true){ // init主循环
            auto shutdown_command = shutdown_state.CheckShutdown(){return do_shutdown_}
            /* do_shutdown_字段的赋值。
              如adb shell敲如命令reboot或shutdown。会通过property server更改sys.powerctl这个property
            PropertySet{
                PropertyChanged {
                    if (name == "sys.powerctl") {
                        trigger_shutdown(value){
                            shutdown_state.TriggerShutdown(command){
                                do_shutdown_ = true;
                                WakeMainInitThread();
                            }
                        }
                    }
            */
#####       if (shutdown_command) {  // 处理重启命令
                HandlePowerctlMessage(*shutdown_command);
                shutdown_state.set_do_shutdown(false);
            }

             bool prop_waiting=bool MightBeWaiting() {
                 auto lock = std::lock_guard{lock_};
                 return static_cast<bool>(waiting_for_prop_); // StartWaiting中会赋值
             } /* waiting_for_prop_字段的赋值
               GetBuiltinFunctionMap{
                  "wait_for_prop"-> do_wait_for_prop{
                     start_waiting_for_property{
                        StartWaiting()
                     }
                  }
               }。
                waiting_for_prop_起来后会赋值为true,只有EnterShutdown以及PropertySet->PropertyChanged->CheckAndResetWait后才会变为false
                  其他进程设置属性值->PropertySet(){
                     PropertyChanged{
                        CheckAndResetWait{
                            if (waiting_for_prop_) {
                                 ResetWaitForPropLocked(){  waiting_for_prop_.reset(); } // waiting_for_prop_置为false
                                 WakeMainInitThread();
                            }
                        }
                     }
                  }
               */

           bool is_exec_service_running = Service::is_exec_service_running(){
               return is_exec_service_running_;
            }/* is_exec_service_running_字段的赋值
                 "exec|exec_start|start"-> do_exec|do_exec_start|do_start{ // action->excutecommand{command->InvokeFunc}的时候就会执行到builtinfunction
                     Service::ExecStart() {
                        Start(){
                            ...
                            LOG(INFO) << "starting service '" << name_ << "'..."; // logcat中可以通过该条日志查看service启动情况
                            ...

                            std::vector<Descriptor> descriptors;
                            // 创建socket
                            for (const auto& socket : sockets_) {
                                auto result = socket.Create(scon){ 
                                创建rc文件中定义的socket, 例如logd服务进程：system/logging/logd/logd.rc
                                service logd /system/bin/logd
                                    #option sockname type mode gid uid
                                    socket logd stream 0666 logd logd
                                    socket logdr seqpacket 0666 logd logd
                                    socket logdw dgram+passcred 0222 logd logd
                                    ...
                                    CreateSocket(name, type | SOCK_CLOEXEC, passcred, listen, perm, uid, gid,
                                                       socket_context){
                                        android::base::unique_fd fd(socket(PF_UNIX, type, 0)); // 创建socket
                                    
                                        struct sockaddr_un addr; // 创建socket地址
                                        addr.sun_family = AF_UNIX;
                                    
                                        if (passcred) {
                                            setsockopt(fd, SOL_SOCKET, SO_PASSCRED, &on, sizeof(on))； // 
                                        }
                                    
                                        int ret = bind(fd, (struct sockaddr *) &addr, sizeof (addr)); // 绑定socket到addr
                                    
                                        lchown(addr.sun_path, uid, gid)； // 对应rc中的chown
                                    
                                        fchmodat(AT_FDCWD, addr.sun_path, perm, AT_SYMLINK_NOFOLLOW); // 对应rc中的chmod
                                    
                                        should_listen && listen(fd, /* use OS maximum */ 1 << 30);  // 开始监听该socket
                                    
                                        LOG(INFO) << "Created socket '" << addr.sun_path << "'"
                                                  << ", mode " << std::oct << perm << std::dec
                                                  << ", user " << uid
                                                  << ", group " << gid; // logcat中可以通过该条日志查看socket创建情况
                                ... 
                                descriptors.emplace_back(std::move(*result));
                            }

                            // 打开文件
                            for (const auto& file : files_) {
                                auto result = file.Create();  // 对应rc中的file option
                                descriptors.emplace_back(std::move(*result));
                            }
                            ...

                            pid = fork(); // 启动子进程
                            if (pid == 0) {
                                RunService(override_mount_namespace, descriptors, std::move(pipefd)){
                                    ExpandArgsAndExecv(args_, sigstop_){
                                       // c_strings[0]即为rc文件中service定义的可执行文件的路径
                                       execv(c_strings[0], c_strings.data()); // 服务执行体占用子进程
                                    }
                                }
                            }
                        }
                        is_exec_service_running_ = true
                     }
                 }

                HandleSignalFd{
                    ReapAnyOutstandingChildren(){
                        ReapOneProcess{
                            service->Reap(siginfo){
                                if (flags_ & SVC_EXEC) UnSetExec(){
                                    is_exec_service_running_ = false; 
                                }
                                        
            */
            
#####           if (!(prop_waiting || is_exec_service_running)) { // 执行command
               // command是前面解析rc文件解析出来的保存在action里面。执行一条command就是执行一个builtinfunction               
               am.ExecuteOneCommand(){

                    // 如果没有执行中的action（action中有很多commands待执行，逐条执行），且有新的事件，则找寻响应该事件的action。
                     // 待执行的action都是由事件触发的，如early-init,init等
                    while (current_executing_actions_.empty() && !event_queue_.empty()) {
                        for (const auto& action : actions_) {
                            if (std::visit([&action](const auto& event) { return action->CheckEvent(event); },
                                           event_queue_.front())) {
                                current_executing_actions_.emplace(action.get());
                            }
                        }
                        event_queue_.pop();
                    }

                   auto action = current_executing_actions_.front(); 
                  // 执行action里面的一条command
                   action->ExecuteOneCommand(command){
                        command.InvokeFunc(subcontext_); // 执行command需要的参数都保存在command内部比如builtinfunc、para
                        /*rc文件主要分两种section：action和service。每个以on开头的section最终会被解析为一个action，
                           下面每一行就是一个command,尤其有些command是用来启动service的。每个command对应一个上文提到的builtinfunction。
                           每个以service开头的会被解析为一个service。其下每一行对应执行ServiceParser::GetParserMap中注册的function
                        */
                   }
               }
           }

            if (!IsShuttingDown()) {
#####                // 重启子进程
                auto next_process_action_time = HandleProcessActions(){
                    for (const auto &s: ServiceList::GetInstance()) {
                        ...
                        if (!(s->flags() & SVC_RESTARTING)) continue; // service进程退出后，被init的sighandler捕获最终处理service（非oneshot）状态会进入SVC_RESTARTING
        
                        auto restart_time = s->time_started() + s->restart_period();
                        if (boot_clock::now() > restart_time) {
                            result = s->Start();  // 重启该服务
                        } 
                        ...
                    }
                }
    
                // If there's a process that needs restarting, wake up in time for that.
                if (next_process_action_time) {
                    epoll_timeout = std::chrono::ceil<std::chrono::milliseconds>(
                            *next_process_action_time - boot_clock::now());
                    if (*epoll_timeout < 0ms) epoll_timeout = 0ms;
                }
            }


          if (!(prop_waiting || is_exec_service_running)) {
              // If there's more work to do, wake up again immediately.
              if (am.HasMoreCommands()) epoll_timeout = 0ms;
          }

#####            // 等待事件的发生。如子进程退出（通过信号处理）、系统属性变更(通过WakeMainInitThread)、MountHandler(/proc/mounts)挂载事件，例如,如果SD卡被插入，init会收到一个设备插入事件。
            auto pending_functions = epoll.Wait(epoll_timeout);
            if (!pending_functions->empty()) {
                // We always reap children before responding to the other pending functions. This is to
                // prevent a race where other daemons see that a service has exited and ask init to
                // start it again via ctl.start before init has reaped it.
                ReapAnyOutstandingChildren();
                for (const auto &function: *pending_functions) {
                    (*function)();
                }
            } else if (is_exec_service_running) {
                
            }
         }

      }

}

## Zygote进程
rc文件路径：system/core/rootdir/init.zygote64_32.rc
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart media.tuner
    onrestart restart netd
    onrestart restart wificond
    task_profiles ProcessCapacityHigh MaxPerformance
    critical window=${zygote.critical_window.minute:-off} target=zygote-fatal

入口：frameworks/base/cmds/app_process/app_main.cpp

    // service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    int main(int argc, char* const argv[]){

        // 创建AppRuntime 的作用就是创建一个可以执行字节码的环境
        AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv)){ // class AppRuntime : public AndroidRuntime
            AndroidRuntime(argBlockStart, argBlockLength) {
                init_android_graphics(){SkGraphics::Init();} // 初始化 skia 图形系统

                // Pre-allocate enough space to hold a fair number of options.
                mOptions.setCapacity(20);
            
                assert(gCurRuntime == NULL);        // one per process
                gCurRuntime = this;
            }
        }

        // 跳过argv[0]即“/system/bin/app_process64”，argv指向下一个参数
        argc--; argv++;

        // 解析vm参数并添加到AppRuntime。（“/system/bin/app_process64”和"/system/bin"之间的参数）
        // Everything up to '--' or first non '-' arg goes to the vm.
        const char* spaced_commands[] = { "-cp", "-classpath" };
        // Allow "spaced commands" to be succeeded by exactly 1 argument (regardless of -s).
        bool known_command = false;
        int i;
        for (i = 0; i < argc; i++) {
            if (known_command == true) {
              // "-cp"或"-classpath"之后的token，虽然不以"-"开头但是期望的参数，这里直接添加，不能走后面的流程（后面要判断“-”开头）
              runtime.addOption(strdup(argv[i])); 
              known_command = false;
              continue;
            }
    
            for (int j = 0; j < static_cast<int>(sizeof(spaced_commands) / sizeof(spaced_commands[0])); ++j) {
              if (strcmp(argv[i], spaced_commands[j]) == 0) { // "-cp"或"-classpath"
                known_command = true;
              }
            }
    
            if (argv[i][0] != '-') {
                // 遇上"/system/bin"了，跳出解析
                break;
            }
            if (argv[i][1] == '-' && argv[i][2] == 0) {
                ++i; // 遇上“-- ”，注意是悬挂的--后面没紧跟选项.
                break;
            }
    
            runtime.addOption(strdup(argv[i])); // "-"开头的token
        }


        // Parse runtime arguments.  Stop at first unrecognized option.
        bool zygote = false;
        bool startSystemServer = false;
        bool application = false;
        String8 niceName;
        String8 className;
    
        ++i;  // Skip unused "parent dir" argument.
        while (i < argc) {
            const char* arg = argv[i++];
            if (strcmp(arg, "--zygote") == 0) {
                zygote = true;
                niceName = ZYGOTE_NICE_NAME;
            } else if (strcmp(arg, "--start-system-server") == 0) {
                startSystemServer = true;
            } else if (strcmp(arg, "--application") == 0) {
                application = true;
            } else if (strncmp(arg, "--nice-name=", 12) == 0) {
                niceName.setTo(arg + 12);
            } else if (strncmp(arg, "--", 2) != 0) {
                className.setTo(arg);
                break;
            } else {
                --i;
                break;
            }
        }

        Vector<String8> args;
        if (!className.isEmpty()) { // APP模式
            // We're not in zygote mode, the only argument we need to pass
            // to RuntimeInit is the application argument.
            //
            // The Remainder of args get passed to startup class main(). Make
            // copies of them before we overwrite them with the process name.
            args.add(application ? String8("application") : String8("tool"));
            runtime.setClassNameAndArgs(className, argc - i, argv + i);
    
        } else { // zygote本尊
            maybeCreateDalvikCache();
    
            if (startSystemServer) {
                args.add(String8("start-system-server"));
            }
    
            char prop[PROP_VALUE_MAX];
            if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
                LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                    ABI_LIST_PROPERTY);
                return 11;
            }
    
            String8 abiFlag("--abi-list=");
            abiFlag.append(prop);
            args.add(abiFlag);
    
            // In zygote mode, pass all remaining arguments to the zygote
            // main() method.
            for (; i < argc; ++i) {
                args.add(String8(argv[i]));
            }
        }
    
        if (!niceName.isEmpty()) { // 64位zygote更名为zygote64
            runtime.setArgv0(niceName.string(), true /* setProcName */);
        }
    
        // 启动
        if (zygote) {
            runtime.start("com.android.internal.os.ZygoteInit", args, zygote){
                // logcat中过滤这个打印观察zygote的启动
                ALOGD(">>>>>> START %s uid %d <<<<<<\n", className != NULL ? className : "(unknown)", getuid());

                // 获取一些环境变量并校验是否存在
                const char* rootDir = getenv("ANDROID_ROOT");
                if (rootDir == NULL) {
                    rootDir = "/system";
                    if (!hasDir("/system")) {
                        return;
                    }
                    setenv("ANDROID_ROOT", rootDir, 1);
                }
                const char* artRootDir = getenv("ANDROID_ART_ROOT");
                if (artRootDir == NULL) {
                    return;
                }
                const char* i18nRootDir = getenv("ANDROID_I18N_ROOT");
                if (i18nRootDir == NULL) {
                    return;
                }
                const char* tzdataRootDir = getenv("ANDROID_TZDATA_ROOT");
                if (tzdataRootDir == NULL) {
                    return;
                }

                // 启动虚拟机
                JniInvocation jni_invocation;
                jni_invocation.Init(NULL);
                JNIEnv* env;
                startVm(&mJavaVM, &env, zygote, primary_zygote){
                    // TODO
                }
                onVmCreated(env);

                // 注册jni方法
                startReg(env){
                    register_jni_procs(gRegJNI, NELEM(gRegJNI), env){
                        for (size_t i = 0; i < count; i++) {
                            array[i].mProc(env);
                        }
                    }
                    /*
                    static const RegJNIRec gRegJNI[] = {
                        REG_JNI(register_com_android_internal_os_RuntimeInit),
                        REG_JNI(register_com_android_internal_os_ZygoteInit_nativeZygoteInit){
                            const JNINativeMethod methods[] = {
                            { "nativeZygoteInit", "()V",
                                (void*) com_android_internal_os_ZygoteInit_nativeZygoteInit },
                            };
                            return jniRegisterNativeMethods(env, "com/android/internal/os/ZygoteInit",
                                methods, NELEM(methods));
                        }
                        REG_JNI(register_android_os_SystemClock),
                        REG_JNI(register_android_util_CharsetUtils),
                        REG_JNI(register_android_util_EventLog),
                        REG_JNI(register_android_util_Log){
                            RegisterMethodsOrDie(env, "android/util/Log", gMethods{ 
                                // 注册android.util.Log.java中用到的native方法。注意这里是通过动态注册的方式，不是静态的通过特殊的函数名称规则映射。
                                { "isLoggable",      "(Ljava/lang/String;I)Z", (void*) android_util_Log_isLoggable },
                                { "println_native",  "(IILjava/lang/String;Ljava/lang/String;)I", (void*) android_util_Log_println_native },
                                { "logger_entry_max_payload_native",  "()I", (void*) android_util_Log_logger_entry_max_payload_native },
                            }, NELEM(gMethods));
                        }
                        ...
                    }
                    */
                }

                /*
                 * Start VM.  This thread becomes the main thread of the VM, and will
                 * not return until the VM exits.
                 */
                char* slashClassName = toSlashClassName(className != NULL ? className : ""); // className="com.android.internal.os.ZygoteInit"
                jclass startClass = env->FindClass(slashClassName);
        
                jmethodID startMeth = env->GetStaticMethodID(startClass, "main", "([Ljava/lang/String;)V");
        
                env->CallStaticVoidMethod(startClass, startMeth, strArray){ // 转到"com.android.internal.os.ZygoteInit.java"的main方法执行
                    // file: com.android.internal.os.ZygoteInit.java
                    public static void main(String[] argv) {
                        // Zygote goes into its own process group.
                        Os.setpgid(0, 0);
                
                        Runnable caller;
                
                        boolean startSystemServer = false;
                        String zygoteSocketName = "zygote";
                        String abiList = null;
                        boolean enableLazyPreload = false;
                        for (int i = 1; i < argv.length; i++) {
                            if ("start-system-server".equals(argv[i])) {
                                startSystemServer = true;
                            } else if ("--enable-lazy-preload".equals(argv[i])) {
                                enableLazyPreload = true;
                            } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                                abiList = argv[i].substring(ABI_LIST_ARG.length());
                            } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                                zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
                            } else {
                                throw new RuntimeException("Unknown command line argument: " + argv[i]);
                            }
                        }
                
                        final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);
                
                        if (!enableLazyPreload) {
                            // 预加载一些类及资源文件
                            preload(bootTimingsTraceLog){
                                preloadClasses(){
                                    /* String PRELOADED_CLASSES = "/system/etc/preloaded-classes"; 
                                        preloaded-classes文件是一个完整类名的列表，非常长。加载可能很耗时，优化可考虑这块。
                                    */
                                    InputStream is = new FileInputStream(PRELOADED_CLASSES);
                                    BufferedReader br = new BufferedReader(new InputStreamReader(is), Zygote.SOCKET_BUFFER_SIZE);
                                    while ((line = br.readLine()) != null) {
                                        ...
                                        Class.forName(line, true, null);
                                        ...
                                    }
                                }
                                ...
                                preloadResources();
                                ...
                            }
                        }
                
                        Zygote.initNativeState(isPrimaryZygote);
                
                        ZygoteServer zygoteServer = new ZygoteServer(isPrimaryZygote){
                            // 创建socket来接收请求
                            mZygoteSocket = Zygote.createManagedSocketFromInitSocket(Zygote.PRIMARY_SOCKET_NAME){
                                final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
                                String env = System.getenv(fullSocketName);
                                int fileDesc = Integer.parseInt(env);
                                FileDescriptor fd = new FileDescriptor();
                                fd.setInt$(fileDesc);
                                return new LocalServerSocket(fd); // 创建LocalServerSocket
                            }
                            mUsapPoolSocket = Zygote.createManagedSocketFromInitSocket(Zygote.USAP_POOL_PRIMARY_SOCKET_NAME);
                        }
                
                        if (startSystemServer) {
                            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer){
                                // 设置system server的启动参数 parsedArgs
                                    ...
                                
                                // 创建systemserver子进程
                                pid = Zygote.forkSystemServer(
                                        parsedArgs.mUid, parsedArgs.mGid,
                                        parsedArgs.mGids,
                                        parsedArgs.mRuntimeFlags,
                                        null,
                                        parsedArgs.mPermittedCapabilities,
                                        parsedArgs.mEffectiveCapabilities);

                                if (pid == 0) {
                                    // 这里已经是systemserver进程
                                    return handleSystemServerProcess(parsedArgs){
                                        // 反射找到systemserver的main方法，返回caller,参考后面App的启动流程
                                        return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                                                parsedArgs.mDisabledCompatChanges,
                                                parsedArgs.mRemainingArgs, cl);
                                    }
                                }else{
                                    return null;
                                }

                            }
                            // {@code r == null} in the parent (zygote) process, and {@code r != null} in the child (system_server) process.
                            if (r != null) {
                                r.run(); // zygote的子进程system_server走这里，这里执行systemserver的main函数
                                return;  
                            }
                        }
                
                        // The select loop returns early in the child process after a fork and
                        // loops forever in the zygote.
                        caller = zygoteServer.runSelectLoop(abiList){

                            while(true){

                                StructPollfd[] pollFDs = new StructPollfd[socketFDs.size()];
                                int pollIndex = 0;
                                for (FileDescriptor socketFD : socketFDs) {
                                    pollFDs[pollIndex] = new StructPollfd();
                                    pollFDs[pollIndex].fd = socketFD;
                                    pollFDs[pollIndex].events = (short) POLLIN;
                                    ++pollIndex;
                                }
                    
                                // 监听事件
                                int pollReturnValue = Os.poll(pollFDs, pollTimeoutMs);
                    
                                while (--pollIndex >= 0) {
                
                                    if (pollIndex == 0) {
                                        // Zygote server socket。rc文件中定义的那个socket。“--socket-name=zygote”
                                        // service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
                                        ZygoteConnection newPeer = acceptCommandPeer(abiList);
                                        peers.add(newPeer);
                                        socketFDs.add(newPeer.getFileDescriptor());
                                    } else if (pollIndex < usapPoolEventFDIndex) {
                                        // Session socket accepted from the Zygote server socket
                
                                        ZygoteConnection connection = peers.get(pollIndex);
                                        final Runnable command = connection.processCommand(this, multipleForksOK){
                                            ...
                                            pid = Zygote.forkAndSpecialize();
                                            if (pid == 0) {
                                                // in child
                                                zygoteServer.setForkChild();
                    
                                                zygoteServer.closeServerSocket();
                                                IoUtils.closeQuietly(serverPipeFd);
                                                serverPipeFd = null;
                    
                                                return handleChildProc(parsedArgs, childPipeFd, parsedArgs.mStartChildZygote){
                                                    return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                                                        parsedArgs.mDisabledCompatChanges,
                                                        parsedArgs.mRemainingArgs, null /* classLoader */){

                                                        RuntimeInit.commonInit();
                                                        ZygoteInit.nativeZygoteInit(){
                                                            static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz){
                                                                gCurRuntime->onZygoteInit(){
                                                                    AppRuntime::onZygoteInit() {
                                                                        sp<ProcessState> proc = ProcessState::self();  // 初始化binder,这样app就可以使用binder机制类
                                                                        ALOGV("App process: starting thread pool.\n");
                                                                        proc->startThreadPool();
                                                                    }
                                                                }
                                                            }
                                                        }
                                                        return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,classLoader){
                                                            VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
                                                            // Remaining arguments are passed to the start class's static main
                                                            return findStaticMain(args.startClass, args.startArgs, classLoader){
                                                                // 找到app的main方法。
                                                                Class<?> cl = Class.forName(className, true, classLoader);
                                                                Method m = cl.getMethod("main", new Class[] { String[].class });
                                                                int modifiers = m.getModifiers();
                                                                return new MethodAndArgsCaller(m, argv);
                                                            }
                                                        }
                                                    }
                                                }
                                            } else {
                                                // In the parent. A pid < 0 indicates a failure and will be handled in
                                                // handleParentProc.
                                                IoUtils.closeQuietly(childPipeFd);
                                                childPipeFd = null;
                                                handleParentProc(pid, serverPipeFd);
                                                return null;
                                            }
                                        }
            
                                        // TODO (chriswailes): Is this extra check necessary?
                                        if (mIsForkChild) {
                                            // We're in the child. We should always have a command to run at
                                            // this stage if processCommand hasn't called "exec".
                                            if (command == null) {
                                                throw new IllegalStateException("command == null");
                                            }
                                            // 只有子进程才会退出循环。父进程（zygote）会永远陷入在循环中（没有return出口）
                                            return command;
                                        } else { 
                                            // 父进程（zygote）会永远陷入在循环中（没有return出口）

                                            // We're in the server - we should never have any commands to run.
                                            if (command != null) {
                                                throw new IllegalStateException("command != null");
                                            }
            
                                            // We don't know whether the remote side of the socket was closed or
                                            // not until we attempt to read from it from processCommand. This
                                            // shows up as a regular POLLIN event in our regular processing
                                            // loop.
                                            if (connection.isClosedByPeer()) {
                                                connection.closeSocket();
                                                peers.remove(pollIndex);
                                                socketFDs.remove(pollIndex);
                                            }
                                        }
                                    } else {
                                        // Either the USAP pool event FD or a USAP reporting pipe.
                
                                        // If this is the event FD the payload will be the number of USAPs removed.
                                        // If this is a reporting pipe FD the payload will be the PID of the USAP
                                        // that was just specialized.  The `continue` statements below ensure that
                                        // the messagePayload will always be valid if we complete the try block
                                        // without an exception.
                                        long messagePayload;
                
                                        try {
                                            byte[] buffer = new byte[Zygote.USAP_MANAGEMENT_MESSAGE_BYTES];
                                            int readBytes =
                                                    Os.read(pollFDs[pollIndex].fd, buffer, 0, buffer.length);
                
                                            if (readBytes == Zygote.USAP_MANAGEMENT_MESSAGE_BYTES) {
                                                DataInputStream inputStream =
                                                        new DataInputStream(new ByteArrayInputStream(buffer));
                
                                                messagePayload = inputStream.readLong();
                                            } else {
                                                Log.e(TAG, "Incomplete read from USAP management FD of size "
                                                        + readBytes);
                                                continue;
                                            }
                                        } catch (Exception ex) {
                                            if (pollIndex == usapPoolEventFDIndex) {
                                                Log.e(TAG, "Failed to read from USAP pool event FD: "
                                                        + ex.getMessage());
                                            } else {
                                                Log.e(TAG, "Failed to read from USAP reporting pipe: "
                                                        + ex.getMessage());
                                            }
                
                                            continue;
                                        }
                
                                        if (pollIndex > usapPoolEventFDIndex) {
                                            Zygote.removeUsapTableEntry((int) messagePayload);
                                        }
                
                                        usapPoolFDRead = true;
                                    }
                                }
                            }
                        }
                        
                        // 只有子进程才会从runSelectLoop返回，zygote进程会陷入死循环
                        // We're in the child process and have exited the select loop. Proceed to execute the command.
                        if (caller != null) {
                            // App的main方法
                            caller.run();
                        }
                    }
                }
            }
        } else if (className) {
            runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
        } 

    }



## SystemServer进程

file: com.android.internal.os.ZygoteInit.java

    public static void main(String[] argv) {
        ...
        if (startSystemServer) { // rc中有"--start-system-server"
            Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer){

                String[] args = {
                        "--setuid=1000",
                        "--setgid=1000",
                        "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,"
                                + "1024,1032,1065,3001,3002,3003,3005,3006,3007,3009,3010,3011,3012",
                        "--capabilities=" + capabilities + "," + capabilities,
                        "--nice-name=system_server",
                        "--runtime-args",
                        "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
                        "com.android.server.SystemServer",  // SystemServer的主类
                }; 
                ZygoteArguments parsedArgs;
        
                /* Request to fork the system server process */
                pid = Zygote.forkSystemServer(
                        parsedArgs.mUid, parsedArgs.mGid,
                        parsedArgs.mGids,
                        parsedArgs.mRuntimeFlags,
                        null,
                        parsedArgs.mPermittedCapabilities,
                        parsedArgs.mEffectiveCapabilities);
        
                if (pid == 0) {
                    // 这里已经是systemserver进程了
                    return handleSystemServerProcess(parsedArgs){
                        return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion, parsedArgs.mDisabledCompatChanges, parsedArgs.mRemainingArgs, cl){

                            RuntimeInit.commonInit();

                            ZygoteInit.nativeZygoteInit(){
                                static void com_android_internal_os_ZygoteInit_nativeZygoteInit(JNIEnv* env, jobject clazz){
                                    gCurRuntime->onZygoteInit(){
                                        AppRuntime::onZygoteInit() {
                                            sp<ProcessState> proc = ProcessState::self();  // 初始化binder,这样app就可以使用binder机制了
                                            ALOGV("App process: starting thread pool.\n");
                                            proc->startThreadPool();
                                        }
                                    }
                                }
                            }

                            return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,classLoader){
                                return findStaticMain(args.startClass, args.startArgs, classLoader){
                                    Class<?> cl = Class.forName(className, true, classLoader);
                                    Method m = cl.getMethod("main", new Class[] { String[].class });
                                    /*
                                     * This throw gets caught in ZygoteInit.main(), which responds
                                     * by invoking the exception's run() method. This arrangement
                                     * clears up all the stack frames that were required in setting
                                     * up the process.
                                     */
                                    return new MethodAndArgsCaller(m, argv);
                                    /*
                                    static class MethodAndArgsCaller implements Runnable {
                                        public void run() {
                                            mMethod.invoke(null, new Object[] { mArgs });
                                        }
                                    }
                                    */
                                }
                            }
                        }
                    }
                }

            }
    
            // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
            // child (system_server) process.
            if (r != null) {
                r.run(); // 执行systemserver的main函数

                // SystemServer的入口： com.android.server.SystemServer.java
                public static void main(String[] args) {
                    new SystemServer().run(){
                        ...
                        Slog.i(TAG, "Entered the Android system server!"); // logcat 中可以据此查看systemserver启动情况

                        // 设置时区
                        String timezoneProperty = SystemProperties.get("persist.sys.timezone");
                        if (!isValidTimeZoneId(timezoneProperty)) {
                            SystemProperties.set("persist.sys.timezone", "GMT");
                        }
                        // 设置语言、本地化设置
                        if (!SystemProperties.get("persist.sys.language").isEmpty()) {
                            final String languageTag = Locale.getDefault().toLanguageTag();
                            SystemProperties.set("persist.sys.locale", languageTag);
                            ...
                        }
                        // 其它设置（虚拟机运行库、内存参数、内存利用率的调整等）
                        ...
            
                        // 设置 system_server中的binder线程池的线程数
                        BinderInternal.setMaxThreads(sMaxBinderThreads);
            
                        // 设置线程的优先级较高
                        android.os.Process.setThreadPriority(android.os.Process.THREAD_PRIORITY_FOREGROUND);
            
                        // 开始looper
                        Looper.prepareMainLooper();
            
                        // 加载so库，Initialize native services.
                        System.loadLibrary("android_servers");
                        ...
            
                        // Initialize the system context.
                        createSystemContext(){
                            ActivityThread activityThread = ActivityThread.systemMain(){

                                ThreadedRenderer.initForSystemProcess();

                                /*ActivityThread 是应用程序的主线程类，启动应用最后执行到的就是 ActivityThread 的 main() 方法。
                                那么为什么 SystemServer 要初始化一个 ActivityThread 对象？实际上 SystemServer 不仅仅是一个单纯的后台进程，
                                它也是一个运行着 Service 组件的进程，很多系统对话框就是从 SystemServer 中显示出来的，因此 
                                SystemServer 本身也需要一个和 APK 应用类似的上下文环境，创建 ActivityThread 是获取这个环境的第一步。
                                */
                                ActivityThread thread = new ActivityThread();

                                /*ActivityThread 在 SystemServer 进程中与普通进程还是有区别的，
                                这里主要是通过 attach(boolen system,int seq) 方法的参数 system 来标识。
                                这里模仿一个 App 的启动。
                                */
                                thread.attach(true, 0){
                                    // 创建Instrumentation对象，一个ActivityThread对应一个，可以监视应用的生命周期
                                    mInstrumentation = new Instrumentation();
                                    mInstrumentation.basicInit(this);
                                    // 创建context对象
                                    ContextImpl context = ContextImpl.createAppContext(this, getSystemContext().mPackageInfo);
                                    // 创建 Application 对象，并调用其 onCreate 方法
                                    mInitialApplication = context.mPackageInfo.makeApplicationInner(true, null);
                                    mInitialApplication.onCreate();

                                    // 初始并设置 ViewRootImpl 的 ConfigChangedCallback 回调
                                    ViewRootImpl.ConfigChangedCallback configChangedCallback = (Configuration globalConfig) -> {
                                        // We need to apply this change to the resources immediately, because upon returning
                                        // the view hierarchy will be informed about it.
                                        if (mResourcesManager.applyConfigurationToResources(globalConfig, null /* compat */)) {
                                            mConfigurationController.updateLocaleListFromAppContext( mInitialApplication.getApplicationContext());
                                            // This actually changed the resources! Tell everyone about it.
                                            final Configuration updatedConfig = mConfigurationController.updatePendingConfiguration(globalConfig);
                                            if (updatedConfig != null) {
                                                sendMessage(H.CONFIGURATION_CHANGED, globalConfig);
                                                mPendingConfiguration = updatedConfig;
                                            }
                                        }
                                    };
                                    ViewRootImpl.addConfigCallback(configChangedCallback);

                                }

                                return thread;
                            }

                            mSystemContext = activityThread.getSystemContext(){
                                if (mSystemContext == null) {
                                    mSystemContext = ContextImpl.createSystemContext(this){
                                        LoadedApk packageInfo = new LoadedApk(mainThread){
                                            mActivityThread = activityThread;
                                            mApplicationInfo = new ApplicationInfo();
                                            mApplicationInfo.packageName = "android";
                                            mPackageName = "android";
                                            /*
                                            通过在aosp源码中所有AndroidManifest.xml文件中搜 package="android"
                                            可以定位到这个应用路径“frameworks/base/core/res/”。该模块的名字是 framework-res，
                                            主要包括一些 App 共用的系统资源，最终会被编译为 framework-res.apk，并预制到 /system/app 目录下。
                                            在 Zygote 启动过程中，preload 函数已经加载了 framework-res.apk 中的资源。所以我们的其他 App 可以直接使用系统资源。
                                            framework-res下的Android.bp：
                                            android_app {
                                                name: "framework-res",
                                                sdk_version: "core_platform",
                                                certificate: "platform",
                                                srcs: [":remote-color-resources-arsc"],
                                                ...
                                            }
                                            */
                                        }
                                        ContextImpl context = new ContextImpl(null, mainThread, packageInfo,
                                                ContextParams.EMPTY, null, null, null, null, null, 0, null, null);
                                        context.setResources(packageInfo.getResources());
                                        context.mResources.updateConfiguration(context.mResourcesManager.getConfiguration(),
                                                context.mResourcesManager.getDisplayMetrics());
                                        context.mContextType = CONTEXT_TYPE_SYSTEM_OR_SYSTEM_UI;
                                        return context;
                                    }
                                }
                                return mSystemContext;
                            }

                            mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
                    
                            final Context systemUiContext = activityThread.getSystemUiContext();
                            systemUiContext.setTheme(DEFAULT_SYSTEM_THEME);
                        }
            
                        // Create the system service manager.
                        mSystemServiceManager = new SystemServiceManager(mSystemContext);
                        mSystemServiceManager.setStartInfo(mRuntimeRestart, mRuntimeStartElapsedTime, mRuntimeStartUptime);
            
                        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
                        // Prepare the thread pool for init tasks that can be parallelized
                        SystemServerInitThreadPool tp = SystemServerInitThreadPool.start();
            
                        /* 启动一系列的Service
                        SystemServiceManager一个成员变量 ArrayList mServices。
                        SystemServiceManager 管理的其实就是 SystemService 的集合，
                        SystemService 是一个抽象类，如果我们想定义一个系统服务，就需要实现SystemService 这个抽象类。
                        */
                        // 启动引导服务
                        startBootstrapServices(t){
                            t.traceBegin("startBootstrapServices"); 

                            // 看门狗
                            t.traceBegin("StartWatchdog"); // 这些日志可以在logcat中查看，据此观测服务启动情况
                            Watchdog.getInstance().start();

                            mSystemServiceManager.startService(Installer.class){
                                ...
                                Constructor<T> constructor = serviceClass.getConstructor(Context.class);
                                service = constructor.newInstance(mContext);
                                service.onStart();
                                ...
                            }
                            ....

                            // 启动ActivityManagerService
                            ActivityTaskManagerService atm = mSystemServiceManager.startService(
                                    ActivityTaskManagerService.Lifecycle.class).getService();
                            mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                                    mSystemServiceManager, atm);
                            mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
                            mActivityManagerService.setInstaller(installer);
                    
                            // 电源管理服务
                            mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
                            ...

                        }

                        // 启动核心服务
                        startCoreServices(t){
                            // Service for system config
                            t.traceBegin("StartSystemConfigService");
                            mSystemServiceManager.startService(SystemConfigService.class);
                            
                            t.traceBegin("StartBatteryService");
                            // Tracks the battery level.  Requires LightService.
                            mSystemServiceManager.startService(BatteryService.class);
                            ...
                        }

                        startOtherServices(t){
                            // 各式各样的服务的初始化，大约1000行
                            ...

                            mActivityManagerService.systemReady(){
                                ...
                                mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady"); // 启动launcher
                            }

                            startSystemUi();
                        }

                        startApexServices(t);
            
                        // Loop forever. // SystemServer进程陷入死循环
                        Looper.loop();

                    }
                }

                return;
            }
        }
        ...
    }


## ServiceManager
ServiceManager是Android系统Binder进程间通信机制的守护进程，负责管理系统中的Binder对象
//============= init.rc
on init
    start logd
    ...
    start servicemanager

//============ frameworks/native/cmds/servicemanager/servicemanager.rc
service servicemanager /system/bin/servicemanager
class core animation
user system
group system readproc
critical
onrestart restart apexd
onrestart restart audioserver
onrestart restart gatekeeperd
onrestart class_restart --only-enabled main
onrestart class_restart --only-enabled hal
onrestart class_restart --only-enabled early_hal
task_profiles ServiceCapacityLow
shutdown critical

````

//========== frameworks/native/cmds/servicemanager/main.cpp
    int main(int argc, char** argv) {
        // 初始化binder
        const char* driver = argc == 2 ? argv[1] : "/dev/binder";
        sp<ProcessState> ps = ProcessState::initWithDriver(driver){
            gProcess = sp<ProcessState>::make(driver){ // 最终会调用ProcessState构造函数
                // 此为构造函数内部
                base::Result<int> opened = open_driver(driver);
                if (opened.ok()) {
                    // 调用mmap进行内存映射（作为Server端才需要）。此为Binder高效的关键
                    // mmap the binder, providing a chunk of virtual address space to receive transactions.
                    mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE,
                                    opened.value(), 0);
                }
            }
        }
        
        // 注册自己
        sp<ServiceManager> manager = sp<ServiceManager>::make(std::make_unique<Access>());
        if (!manager->addService("manager", manager, false /*allowIsolated*/, IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT).isOk()) {
            LOG(ERROR) << "Could not self register servicemanager";
        }
    
        // 保存自己作为BBinder对象到IPCThreadState（方便其他进程直接使用）
        IPCThreadState::self()->setTheContextObject(manager);
        // 向Binder驱动注册自己成为全局唯一的SM,handler=0
        ps->becomeContextManager(){
            int result = ioctl(mDriverFD, BINDER_SET_CONTEXT_MGR_EXT, &obj);
        }
    
        sp<Looper> looper = Looper::prepare(false /*allowNonCallbacks*/);
        // 将binder驱动的fd注册到Looper中监听，当驱动上报事件时回调handleEvent,处理相关事务。
        BinderCallback::setupTo(looper);
        // 通知驱动将当前线程加入Looper状态
        ClientCallbackCallback::setupTo(looper, manager);
    
        while(true) {
            looper->pollAll(-1); // 阻塞等待event到来，然后进行ioctl和驱动交互获取数据
        }
    
        // should not be reached
        return EXIT_FAILURE;
    }


````

## launcher

SystemServer进程启动后会拉起launcher：

    com.android.server.SystemServer#main(){
        startOtherServices(t){
            mActivityManagerService.systemReady(){
                ...
                mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady"){

                    //>>>>>>>>>>>>>>>>>>> file: com.android.server.wm.RootWindowContainer.java

                    mRootWindowContainer.startHomeOnAllDisplays(userId, reason){
                    startHomeOnDisplay(userId, reason, displayId){
                    startHomeOnTaskDisplayArea(int userId, String reason, TaskDisplayArea taskDisplayArea,boolean allowInstrumenting, boolean fromHomeKey) {
                        if (taskDisplayArea == getDefaultTaskDisplayArea()) {
                             // 构造一个Home intent
                            homeIntent = mService.getHomeIntent(){intent.addCategory(Intent.CATEGORY_HOME);} 
                            
                            // 通过PKMS从系统已安装的引用中，找到一个符合HomeItent的Activity
                            // packages/apps/Launcher3/src/com/android/launcher3/Launcher.java
                            aInfo = resolveHomeActivity(userId, homeIntent){
                                final ComponentName comp = homeIntent.getComponent();
                                ActivityInfo aInfo = null;
                                final String resolvedType = homeIntent.resolveTypeIfNeeded(mService.mContext.getContentResolver());
                                /*resolveIntent做了两件事：
                                1.通过queryIntentActivities来查找符合HomeIntent需求Activities
                                2.通过chooseBestActivity找到最符合Intent需求的Activity信息*/
                                final ResolveInfo info = AppGlobals.getPackageManager().resolveIntent(homeIntent, resolvedType, flags, userId);
                                if (info != null) {
                                    aInfo = info.activityInfo;
                                }
                        
                                aInfo = new ActivityInfo(aInfo);
                                aInfo.applicationInfo = mService.getAppInfoForUser(aInfo.applicationInfo, userId);
                                return aInfo;
                            }

                            // 对homeIntent一些设置
                            ...
                            

                            mService.getActivityStartController().startHomeActivity(homeIntent, aInfo, myReason,taskDisplayArea){

                                //>>>>>>>>>>>>>>>>>>>>>>>  com.android.server.wm.ActivityStartController.java

                                ..............
                                //一系列 setXXX() 方法传入启动所需的各种参数，最后的 execute() 是真正的启动逻辑
                                //最后执行 ActivityStarter的execute方法
                                mLastHomeActivityStartResult = obtainStarter(intent, "startHomeActivity: " + reason)
                                    .setOutActivity(tmpOutRecord)
                                    .setCallingUid(0)
                                    .setActivityInfo(aInfo)
                                    .setActivityOptions(options.toBundle())
                                    //执行 ActivityStarter的execute方法
                                    .execute(){

                                        //>>>>>>>>>>>>>>>>>>>>>>>>> com.android.server.wm.ActivityStarter.java

                                        res = executeRequest(mRequest){
                                            mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                                                request.voiceInteractor, startFlags, true /* doResume */, checkedOptions,
                                                inTask, inTaskFragment, balCode, intentGrants){

                                                result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                                                    startFlags, doResume, options, inTask, inTaskFragment, balCode,
                                                    intentGrants){
                                                    startResult = recycleTask(targetTask, targetTaskTop, reusedTask, intentGrants){
                                                        resumeTargetRootTaskIfNeeded(){
                                                            mRootWindowContainer.resumeFocusedTasksTopActivities(mTargetRootTask, null,mOptions, mTransientLaunch){

                                                                //>>>>>>>>>>>>>>>>>>> file: com.android.server.wm.RootWindowContainer.java

                                                                result = targetRootTask.resumeTopActivityUncheckedLocked(target, targetOptions,deferPause){
                                                                    //>>>>>>>>>>>>>>>>>>> com.android.server.wm.Task.java
                                                                    someActivityResumed = resumeTopActivityInnerLocked(prev, options, deferPause){
                                                                        resumed[0] = topFragment.resumeTopActivity(prev, options, deferPause){
                                                                            //>>>>>>>>>>>>>>>>> com.android.server.wm.ActivityTaskSupervisor.java
                                                                            mTaskSupervisor.startSpecificActivity(next, true, true){
                                                                                mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? HostingRecord.HOSTING_TYPE_TOP_ACTIVITY : HostingRecord.HOSTING_TYPE_ACTIVITY){
                                                                                    //>>>>>>>>>>>>>>com.android.server.wm.ActivityTaskManagerService.java
                                                                                    // Post message to start process to avoid possible deadlock of calling into AMS with the ATMS lock held.
                                                                                    final Message m = PooledLambda.obtainMessage(ActivityManagerInternal::startProcess{
                                                                                                //>>>>>>>>>>>>>> com.android.server.am.ActivityManagerService.java  
                                                                                                // 运行在SystemServer进程中的AMS服务
                                                                                                startProcessLocked(){
                                                                                                    return mProcessList.startProcessLocked(){
                                                                                                        //>>>>>>>>>>>>> com.android.server.am.ProcessList.java
                                                                                                        startProcessLocked(app, hostingRecord, zygotePolicyFlags, abiOverride){
                                                                                                            Process.ProcessStartResult startResult = startProcess(hostingRecord,
                                                                                                                entryPoint, app,    //######### entryPoint = "android.app.ActivityThread"; 所有app的main入口
                                                                                                                uid, gids, runtimeFlags, zygotePolicyFlags, mountExternal, seInfo,
                                                                                                                requiredAbi, instructionSet, invokeWith, startUptime){

                                                                                                                    startResult = Process.start(entryPoint,
                                                                                                                            app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                                                                                                                            app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                                                                                                                            app.info.dataDir, invokeWith, app.info.packageName, zygotePolicyFlags,
                                                                                                                            isTopApp, app.getDisabledCompatChanges(), pkgDataInfoMap,
                                                                                                                            allowlistedAppDataInfoMap, bindMountAppsData, bindMountAppStorageDirs,
                                                                                                                            new String[]{PROC_START_SEQ_IDENT + app.getStartSeq()}){
                                                                                                                        //>>>>>>>>>>>>>>>>> android.os.Process.java
                                                                                                                        return ZYGOTE_PROCESS.start(processClass, niceName, uid, gid, gids,
                                                                                                                                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                                                                                                                                    abi, instructionSet, appDataDir, invokeWith, packageName,
                                                                                                                                    zygotePolicyFlags, isTopApp, disabledCompatChanges,
                                                                                                                                    pkgDataInfoMap, whitelistedDataInfoMap, bindMountAppsData,
                                                                                                                                    bindMountAppStorageDirs, zygoteArgs){
                                                                                                                            //>>>>>>>>>>>>>>>>> android.os.ZygoteProcess.java
                                                                                                                            startViaZygote(){
                                                                                                                                return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi),
                                                                                                                                                                  zygotePolicyFlags,
                                                                                                                                                                  argsForZygote){
                                                                                                                                    return attemptZygoteSendArgsAndGetResult(zygoteState, msgStr){
                                                                                                                                        ...
                                                                                                                                        final BufferedWriter zygoteWriter = zygoteState.mZygoteOutputWriter;
                                                                                                                                        final DataInputStream zygoteInputStream = zygoteState.mZygoteInputStream;
                                                                                                                                        // 写socket,zygote侧epoll会收到
                                                                                                                                        zygoteWriter.write(msgStr);
                                                                                                                                        ...

                                                                                                                                        // zygote进程
                                                                                                                                        //>>>>>>>>>>>>>>>>>>>>>> com.android.internal.os.ZygoteServer.java
                                                                                                                                        // zygote主循环会收到AMS的请求
                                                                                                                                        caller=zygoteServer.runSelectLoop(String abiList) {
                                                                                                                                            while(true){
                                                                                                                                                ArrayList<FileDescriptor> socketFDs = new ArrayList<FileDescriptor>();
                                                                                                                                                ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
                                                                                                                                                try {
                                                                                                                                                    Os.poll(pollFDs, -1);  // poll 监听 fd
                                                                                                                                                } catch (ErrnoException ex) {
                                                                                                                                                    throw new RuntimeException("poll failed", ex);
                                                                                                                                                }
                                                                                                                                                boolean usapPoolFDRead = false;
                                                                                                                                    
                                                                                                                                                // poll 唤醒，开始收数据，处理数据
                                                                                                                                                while (--pollIndex >= 0) {
                                                                                                                                                    if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                                                                                                                                                        continue;
                                                                                                                                                    }
                                                                                                                                    
                                                                                                                                                    if (pollIndex == 0) { //第一个 zygote socket fd
                                                                                                                                                        // Zygote server socket
                                                                                                                                    
                                                                                                                                                        // 构建一个与客户端通信的 ZygoteConnection 类型的辅助对象
                                                                                                                                                        ZygoteConnection newPeer = acceptCommandPeer(abiList);
                                                                                                                                                        peers.add(newPeer);
                                                                                                                                                        // 把与客户端连接的 fd 存入到 socketFDs
                                                                                                                                                        // 下次循环，fd 会加入 pollFDs 中被监听
                                                                                                                                                        socketFDs.add(newPeer.getFileDescriptor());
                                                                                                                                    
                                                                                                                                                    } else if (pollIndex < usapPoolEventFDIndex) { // 与客户端连接的 fd
                                                                                                                                                        // Session socket accepted from the Zygote server socket
                                                                                                                                    
                                                                                                                                                        try {
                                                                                                                                                            // 读取客户端发来的数据，创建新的进程，把新进程的 main 函数包装为一个 Runnable 对象
                                                                                                                                                            ZygoteConnection connection = peers.get(pollIndex);
                                                                                                                                                            // fork一个子进程
                                                                                                                                                            final Runnable command = connection.processOneCommand(this);
                                                                                                                                    
                                                                                                                                                            // TODO (chriswailes): Is this extra check necessary?
                                                                                                                                                            // 子进程直接返回 Runnable
                                                                                                                                                            if (mIsForkChild) {  
                                                                                                                                                                // We're in the child. We should always have a command to run at this
                                                                                                                                                                // stage if processOneCommand hasn't called "exec".
                                                                                                                                                                if (command == null) {
                                                                                                                                                                    throw new IllegalStateException("command == null");
                                                                                                                                                                }
                                                                                                                                    
                                                                                                                                                                return command; // 子进程跳出循环，外面会执行command#run, 即android.app.ActivityThread#main()
                                                                                                                                                            } else {
                                                                                                                                                                // We're in the server - we should never have any commands to run.
                                                                                                                                                                if (command != null) {
                                                                                                                                                                    throw new IllegalStateException("command != null");
                                                                                                                                                                }
                                                                                                                                    
                                                                                                                                                                // We don't know whether the remote side of the socket was closed or
                                                                                                                                                                // not until we attempt to read from it from processOneCommand. This
                                                                                                                                                                // shows up as a regular POLLIN event in our regular processing loop.
                                                                                                                                                                if (connection.isClosedByPeer()) {
                                                                                                                                                                    connection.closeSocket();
                                                                                                                                                                    peers.remove(pollIndex);
                                                                                                                                                                    socketFDs.remove(pollIndex);
                                                                                                                                                                }
                                                                                                                                                            }
                                                                                                                                                            ...
                                                                                                                                                        }
                                                                                                                                                    } else {
                                                                                                                                                        // ......
                                                                                                                                                        
                                                                                                                                                    }
                                                                                                                                                }
                                                                                                                                    
                                                                                                                                               // ......
                                                                                                                                            }
                                                                                                                                        }

                                                                                                                                        if (caller != null) {
                                                                                                                                            // 这里已经是子进程，只有子进程才会走到这里
                                                                                                                                            // App的main方法，android.app.ActivityThread#main()
                                                                                                                                            caller.run();
                                                                                                                                        }

                                                                                                                                    }
                                                                                                                                }
                                                                                                                            }
                                                                                                                        }
                                                                                                                    }

                                                                                                                }
                                                                                                        }
                                                                                                    }
                                                                                                }
                                                                                            },

                                                                                            mAmInternal, activity.processName, activity.info.applicationInfo, knownToBeDead,
                                                                                            isTop, hostingType, activity.intent.getComponent());

                                                                                    mH.sendMessage(m);

                                                                                }
                                                                            }
                                                                        }
                                                                    }
                                                                }
                                                            }
                                                        }
                                                    }
                                                }

                                            }
                                        }
                                    }

                            }

                        }
                    }
                    }
                    }
                }
            }
        }
    }



## android.app.ActivityThread

    public static void main(String[] args) {

        Looper.prepareMainLooper(){
            prepare(false); // allowquit=false，表示不允许退出。普通的looper是允许退出的，主线程的不允许。
            synchronized (Looper.class) {
                if (sMainLooper != null) {
                    throw new IllegalStateException("The main Looper has already been prepared.");
                }
                sMainLooper = myLooper();
            }
        }

        ActivityThread thread = new ActivityThread();

        thread.attach(false, startSeq){
            // IActivityManager extends IInterface。binder机制中的用户接口定义
            final IActivityManager mgr = ActivityManager.getService();
            // binder机制中的proxy，远程过程调用（RPC）调到Stub（class ActivityManagerService extends IActivityManager.Stub）中对应的attachApplication
            mgr.attachApplication(mAppThread, startSeq){
                //>>>>>>>>>>>>>>>>> com.android.server.am.ActivityManagerService.java
                //#### SystemServer Process 的AMS
                attachApplicationLocked(thread, callingPid, callingUid, startSeq){
                    ...
                    // RPC回到ActivityThread
                    //#### App process
                    thread.bindApplication(){
                        //>>>>>>>>>>>>>>>>>>> android.app.ActivityThread.java
                        ...
                        sendMessage(H.BIND_APPLICATION, data);
                        // 消息接收：
                        public void handleMessage(Message msg) {
                            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
                            switch (msg.what) {
                                case BIND_APPLICATION:
                                    AppBindData data = (AppBindData)msg.obj;
                                    handleBindApplication(data){
                                        // 该app即为我们的Application实例
                                        Application app = data.info.makeApplicationInner(data.restrictedBackupMode, null){
                                            //>>>>>>>>>>>> frameworks/base/core/java/android/app/LoadedApk.java
                                            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
                                            app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext){
                                                //>>>>>>>>>>> frameworks/base/core/java/android/app/Instrumentation.java
                                                Application app = getFactory(context.getPackageName()).instantiateApplication(cl, className);
                                                app.attach(context){
                                                    //>>>>>>>>>>>>> frameworks/base/core/java/android/app/Application.java
                                                    attachBaseContext(context){ // 这里会调用子类的attachBaseContext，也就是Application#attachBaseContext()
                                                        //>>>>>>>>>>>>>> frameworks/base/core/java/android/content/ContextWrapper.java
                                                        if (mBase != null) {
                                                            throw new IllegalStateException("Base context already set");
                                                        }
                                                        mBase = base;
                                                    }
                                                    mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
                                                }
                                                return app;
                                            }
                                        }
                                        mInstrumentation.callApplicationOnCreate(app){
                                            app.onCreate();     // 这个就是我们Application#onCreate()回调。
                                        }
                                    }
                                    break;
                    }
                }
            }
        }


        // 陷入死循环，处理各种event
        Looper.loop();

    }





## low memory killer

AMS对进程的管理主要体现在两个方面：
- 进程优先级动态调整：实际是调整进程oom_adj的值。
- 进程LRU（Least Recently Used）列表动态更新：动态调整进程在mLruProcesses列表的位置

Framework层通过一定的规则调整进程的adj的值和内存空间阀值，然后通过socket发送给lmkd进程，
lmkd两种处理方式， 一种将阀值写入文件节点发送给内核的LowMemoryKiller，由内核进行杀进程处理，另一种是lmkd通过cgroup监控内存使用情况，自行计算杀掉进程。

lmkd通过epoll监听socket中是否有数据， 当接受的framework发送的socket命令之后，调用ctrl_cmmand_handler方法处理，显示解析socket中的命令和参数，根据对于的命令来调用不同的方法处理。
    cmd_target 调整最小内存阀值和adj值
    cmd_procprio 调整进程的adj值
    cmd_procremove 移除对应的进程

### 服务端——lmkd
在init.rc中触发启动：
on init
    ...
    # Start lmkd before any other services run so that it can register them
    write /proc/sys/vm/watermark_boost_factor 0
    chown root system /sys/module/lowmemorykiller/parameters/adj
    chmod 0664 /sys/module/lowmemorykiller/parameters/adj
    chown root system /sys/module/lowmemorykiller/parameters/minfree
    chmod 0664 /sys/module/lowmemorykiller/parameters/minfree
    start lmkd

lmkd服务定义（全局搜索"service .*lmkd"，过滤*.rc文件即可找到定义lmkd服务的rc文件）：
system/memory/lmkd/lmkd.rc
    service lmkd /system/bin/lmkd
    class core
    user lmkd
    group lmkd system readproc
    capabilities DAC_OVERRIDE KILL IPC_LOCK SYS_NICE SYS_RESOURCE
    critical
    socket lmkd seqpacket+passcred 0660 system system
    task_profiles ServiceCapacityLow
    on property:lmkd.reinit=1
    exec_background /system/bin/lmkd --reinit
    on property:
    ...

lmkd代码入口（全局搜索lmkd，过滤*.bp即可找到lmkd的构建脚本进而找到源码入口）：
cc_binary {
name: "lmkd",
    srcs: [
        "lmkd.cpp",
        "reaper.cpp",
        "watchdog.cpp",
    ],
    ...

system/memory/lmkd/lmkd.cpp

````
int main(int argc, char **argv) {

    // 读取prop配置参数
    update_props();

    // 初始化init epoll monitors事件监听
    int success = !init(void);
    
    if (success) {
        if (!use_inkernel_interface) {
            // 设置进程调度器
            if (sched_setscheduler(0, SCHED_FIFO, &param)) {
                ALOGW("set SCHED_FIFO failed %s", strerror(errno));
            }
        }
        
        // 循环处理事件
        mainloop(){
            while(1){
                ...
                /* Wait for events until the next polling timeout */
                // 阻塞等待，当psi阈值触发时会返回
                /*
                在Android10以及以后的版本，android变采用基于PSI 的LMK。
                PSI（Pressure Stall Information）是Facebook搞的一套东西并在2018 年开源。PSI提供了一种评估系统资源压力的方法。
                系统有三个基础资源：CPU、Memory 和 IO。PSI 主要也是对这三种资源进行监控。
                */
                nevents = epoll_wait(epollfd, events, maxevents, delay);
                ...
                
                /* Second pass to handle all other events */
                //获取对应psi事件等级的回调对象
                for (i = 0, evt = &events[0]; i < nevents; ++i, evt++) {
                    if (evt->data.ptr) {
                        取出存储的回调方法和参数，调用handler
                        handler_info = (struct event_handler_info*)evt->data.ptr;
                        call_handler(handler_info, &poll_params, evt->events) {
                            struct timespec curr_tm;
                        
                            watchdog.start();
                            poll_params->update = POLLING_DO_NOT_CHANGE;
                            
                            // 调用handler方法，对于psi来说这里的方法就是mp_event_psi
                            handler_info->handler(handler_info->data, events, poll_params){
                            
                                mp_event_psi(){
                                    ...
                                    
                                    /* Kill a process if necessary */
                                    if (kill_reason != NONE) {
                                        struct kill_info ki = {
                                            .kill_reason = kill_reason,
                                            .kill_desc = kill_desc,
                                            .thrashing = (int)thrashing,
                                            .max_thrashing = max_thrashing,
                                        };
                                
                                        /* Allow killing perceptible apps if the system is stalled */
                                        if (critical_stall) {
                                            min_score_adj = 0;
                                        }
                                        psi_parse_io(&psi_data);
                                        psi_parse_cpu(&psi_data);
                                        int pages_freed = find_and_kill_process(min_score_adj, &ki, &mi, &wi, &curr_tm, &psi_data){

                                            for (i = OOM_SCORE_ADJ_MAX; i >= min_score_adj; i--) {
                                                struct proc *procp;
                                        
                                                if (!choose_heaviest_task && i <= PERCEPTIBLE_APP_ADJ) {
                                                    /*
                                                     * If we have to choose a perceptible process, choose the heaviest one to
                                                     * hopefully minimize the number of victims.
                                                     */
                                                    choose_heaviest_task = true;
                                                }
                                        
                                                while (true) {
                                                    procp = choose_heaviest_task ?
                                                        proc_get_heaviest(i) : proc_adj_tail(i);
                                        
                                                    if (!procp)
                                                        break;
                                        
                                                    killed_size = kill_one_process(procp, min_score_adj, ki, mi, wi, tm, pd){
                                                        ...

                                                        mem_st = stats_read_memory_stat(per_app_memcg, pid, uid, rss_kb * 1024, swap_kb * 1024);
                                                    
                                                        start_wait_for_proc_kill(pidfd < 0 ? pid : pidfd);
                                                        kill_result = reaper.kill({ pidfd, pid, uid }, false);
                                                    
                                                    
                                                        last_kill_tm = *tm;
                                                    
                                                        inc_killcnt(procp->oomadj);
                                                    
                                                        return result;
                                                    }
                                                    
                                                    if (killed_size >= 0) {
                                                        if (!lmk_state_change_start) {
                                                            lmk_state_change_start = true;
                                                            stats_write_lmk_state_changed(STATE_START);
                                                        }
                                                        break;
                                                    }
                                                }
                                                if (killed_size) {
                                                    break;
                                                }
                                            }
                                        
                                            return killed_size;
                                        }
                                        ...
                                    }
                                    ...
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    
    ALOGI("exiting");
    return 0;
}


````

### 客户端——AMS

````

================ frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
// 创建到lmkd的连接
public ActivityManagerService(Context systemContext, ActivityTaskManagerService atm) {
    ...
    mProcessList.init(this, activeUids, mPlatformCompat){

        if (sKillHandler == null) {
            sKillThread = new ServiceThread(TAG + ":kill",
                    THREAD_PRIORITY_BACKGROUND, true /* allowIo */);
            sKillThread.start();
            sKillHandler = new KillHandler(sKillThread.getLooper());
            // 创建到lmkd的连接
            sLmkdConnection = new LmkdConnection(sKillThread.getLooper().getQueue(),
                    new LmkdConnection.LmkdConnectionListener() {
                        ...
                    }
            );
            
            // Start listening on incoming connections from zygotes.
            mSystemServerSocketForZygote = createSystemServerSocketForZygote();
            if (mSystemServerSocketForZygote != null) {
                sKillHandler.getLooper().getQueue().addOnFileDescriptorEventListener(
                        mSystemServerSocketForZygote.getFileDescriptor(),
                        EVENT_INPUT, this::handleZygoteMessages);
            }
            mAppExitInfoTracker.init(mService);
            mImperceptibleKillRunner = new ImperceptibleKillRunner(sKillThread.getLooper());
        }
    }
}

// 向lmkd写OomAdj值
// OomAdj值越大表示该进程被杀的可能性越大。
// 实测前台进程OomAdj=0,后台=11。查看方法： cat /proc/<pid>/oom_adj 

start|stop activity|service，前后台, ... 触发->{
    final boolean updateOomAdjLocked(ProcessRecord app, String oomAdjReason) {
      //============== frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
        return mOomAdjuster.updateOomAdjLocked(app, oomAdjReason){
          //============== frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java
            return updateOomAdjLSP(app, oomAdjReason){
                return performUpdateOomAdjLSP(app, oomAdjReason){
                    performUpdateOomAdjLSP(app, cachedAdj, topApp, SystemClock.uptimeMillis(), oomAdjReason){
                        return applyOomAdjLSP(app, false, now, SystemClock.elapsedRealtime(), oomAdjReason){
                            ProcessList.setOomAdj(app.getPid(), app.uid, state.getCurAdj()){
                              //============ frameworks/base/services/core/java/com/android/server/am/ProcessList.java  
                                writeLmkd(buf, null){
                                    return sLmkdConnection.exchange(buf, repl){
                                      //=============== frameworks/base/services/core/java/com/android/server/am/LmkdConnection.java
                                        return write(req){
                                            // 更新oomadj到lmdk
                                            //mLmkdOutputStream为跟lmkd关联的socket的stream对象，往里面写数据，lmkd侧的epoll会监听到
                                            mLmkdOutputStream.write(buf.array(), 0, buf.position()); 
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}    


// 更新配置，手机屏幕的尺寸和内存大小不一样，对应的最小内存阀值和adj值也不一样, 最终调用ProcessList的updateOomLevel方法向lmkd发送调整命令
public void updateOomLevelsForDisplay(int displayId) {
    synchronized(ActivityManagerService.this) {
        if (mWindowManager != null) {
            mProcessList.applyDisplaySize(mWindowManager);
        }
    }
}


// 更新进程在mLruProcesses列表的位置
各种场景下，常见的如四大组件生命周期变更，会触发{
    final void updateLruProcessLocked(ProcessRecord app, boolean activityChange,
            ProcessRecord client) {
        // 根据进程中四大组件的存在状态前后台状态等调整processrecord在list中的位置
        // list分3段：|other|service|activity| ，从右至左表示有activity的进程、有service的进程、其他进程
        mProcessList.updateLruProcessLocked(app, activityChange, client){
            ...
        }
    }
}
````

## 从launcher启动App
### 流程
1. **Launcher进程**

2. 点击应用图标(Launcher记录了每个shortcut的相关信息：WorkspaceItemInfo，其中包含了用来启动app的Intent，intent会设置FLAG_ACTIVITY_NEW_TASK）

    *packages/apps/Launcher3/src/com/android/launcher3/Launcher.java*
    
        onClick(){
        startAppShortcutOrInfoActivity{
        startActivitySafely{
        super.startActivitySafely(v, intent, item){
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK); // 目标Activity应该在其所属APP进程的新Task启动

3. 触发startActivity

        startActivity(Intent intent, @Nullable Bundle options) { 

        public void startActivityForResult(Intent intent, 
                    int requestCode,   // requestCode为-1，说明我们不需要activity的返回值
                    Bundle options) {
            super.startActivityForResult(intent, requestCode, options){

4. 最终调到Instrumentation.execStartActivity<br>
   Intrumentation 对象用来监控应用程序和系统之间的交互操作。由于启动 Activity 是应用程序与系统之间的交互操作，<br>
   因此调用它的成员函数 execStartActivity 来代替执行启动 Activity 的操作，以便它可以监控这个交互过程。<br>

                Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity(
                        this, // launcher
                        mMainThread.getApplicationThread(), // mMainThread是Launcher的ActivityThread实例，ApplicationThread它是一个Binder，用来RPC
                        mToken, this, intent, requestCode, options){

    **frameworks/base/core/java/android/app/Instrumentation.java**

                    /* ActivityTaskManager.getService()获取到一个binder包装对象IActivityTaskManager，
                     startActivity是RPC调用。查看调用的目标类可以先查找IActivityTaskManager的类继承结构。
                     public class ActivityTaskManagerService extends IActivityTaskManager.Stub 
                        public abstract static class Stub extends Binder implements IActivityTaskManager
                     ActivityTaskManagerService是IActivityTaskManager的子类，所以最终调用的是ActivityTaskManagerService的startActivity
                    */
                    int result = ActivityTaskManager.getService(){
5. **通过ActivityTaskManager获取到IActivityTaskManager#Stub#Proxy实例**

                            return IActivityTaskManagerSingleton.get(){
                                final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                                return IActivityTaskManager.Stub.asInterface(b);
                            }
                        }.startActivity(IApplicationThread caller, String callingPackage,
                                String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
                                String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
                                Bundle bOptions){
6. **最终通过Binder机制实现RPC调用到ActivityTaskManagerService#startActivity**

7. 切进程：**Launcher->SystemServer**

    **frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java**

                        startActivityAsUser(){
8. 获取到ActivityStarter，经过一系列配置最终调用其execute

                            return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                            ...
                            .execute(){
    *frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java*

                                if (mRequest.activityInfo == null) {
                                    // 从 PMS 中获取目标 Activity 信息，解析出的信息保存在 mRequest.activityInfo
                                    mRequest.resolveActivity(mSupervisor);
                                }

                                executeRequest(){
9. **经过一系列的check,最终创建ActivityRecord，调用startActivityUnchecked**

                                    ...
                                    final ActivityRecord r = new ActivityRecord.Builder(mService)
                                    .setCaller(callerApp)
                                     ...
                                    .build();

                                    startActivityUnchecked(r, sourceRecord, voiceSession,
                                             request.voiceInteractor, startFlags, true /* doResume */, checkedOptions,
                                             inTask, inTaskFragment, balCode, intentGrants){
                                        startActivityInner(){
10. 查找可用的任务栈，如果有则直接复用，如果没有，则新建一个任务栈。然后调用RootWindowContainer.resumeFocusedTasksTopActivities

                                             mRootWindowContainer.resumeFocusedTasksTopActivities(mTargetRootTask, mStartActivity, mOptions, mTransientLaunch){

     *frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java*

                                                 result = targetRootTask.resumeTopActivityUncheckedLocked(target, targetOptions,deferPause){
     *frameworks/base/services/core/java/com/android/server/wm/Task.java*

                                                     someActivityResumed = resumeTopActivityInnerLocked(prev, options, deferPause){

                                                         resumed[0] = topFragment.resumeTopActivity(prev, options, deferPause){
11. 最终调用到topFragment.resumeTopActivity

    */home/sissi/AndroidStudioProjects/aosp-13.0.0_r83/frameworks/base/services/core/java/com/android/server/wm/TaskFragment.java*
12. pause其他Activity（所以从一个activity A启动B时，A会先Pause）

                                                            taskDisplayArea.pauseBackTasks(next); // pause其他Activity

                                                            if (next.attachedToProcess()) {
13. **要启动的这个 Activity 已经存在，并且设置了像“singleInstance” 的启动模式，无需重新创建 Activity。**<br> 
    则先通过 ClientTransaction 添加了一个 NewIntentItem 的 callback，接下来通过 setLifecycleStateRequest 设置了一个 ResumeActivityItem 对象。

                                                            } else{

14. Activity不存在则继续走启动Activity流程

                                                                mTaskSupervisor.startSpecificActivity(next, true, true){
    *frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java*

                                                                    if (wpc != null && wpc.hasThread()) { // 进程已存在
15. <span id="hot_start"> <b>热启动：如果目标 Activity 所在的应用进程已经在运行则直接在该进程中启动该Activity</b> </span>

                                                                        realStartActivityLocked(r, wpc, andResume, checkConfig){
16. **创建一个activity执行事务ClientTransaction。**
    ClientTransaction 是包含了一系列要执行的事务项的事务。我们可以通过调用它的 addCallback方法来添加一个事务项。
    另外可以通过 ClientTransactionItem 的 setLifecycleStateRequest 方法设置 Activity 执行完后最终的生命周期状态，其参数的类型为 ActivityLifecycleItem。

                                                                            final ClientTransaction clientTransaction = ClientTransaction.obtain(proc.getThread(), r.token);

17. <span id="add_callback"> 添加一个事务项LaunchActivityItem </span>

                                                                            clientTransaction.addCallback(LaunchActivityItem.obtain(...));

18. 设置 Activity 启动后最终的生命周期状态

                                                                             // Activity 启动后最终的生命周期状态
                                                                             final ActivityLifecycleItem lifecycleItem;
                                                                             if (andResume) {
                                                                                 // 将最终生命周期设置为 Resume 状态
                                                                                 lifecycleItem = ResumeActivityItem.obtain(isTransitionForward);
                                                                             } else {
                                                                                 // 将最终生命周期设置为 Pause 状态
                                                                                 lifecycleItem = PauseActivityItem.obtain();
                                                                             }
                                                                             clientTransaction.setLifecycleStateRequest(lifecycleItem);
19. 开启事务

                                                                             mService.getLifecycleManager().scheduleTransaction(clientTransaction){
                                                                                transaction.schedule(){

                                                                                    // mClient为ApplicationThread,为App的Binder
                                                                                    mClient.scheduleTransaction(this); 
20. **通过Binder机制RPC调用ApplicationThread#scheduleTransaction**

21. 切进程**SystemServer->App**
    *frameworks/base/core/java/android/app/ActivityThread.java*

        ApplicationThread#scheduleTransaction(ClientTransaction transaction) {
            ActivityThread.this.scheduleTransaction(transaction){
                sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
                    ...
                    case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    mTransactionExecutor.execute(transaction){
22. 执行所有被添加进来的 ClientTransactionItem
    ClientTransactionItem 有多个实现类，这些实现类对应了 Activity 中不同的执行流程。
    例如在 Activity 启动时如果不需要重新创建 Activity ，则会通过 addCallback 添加了一个 NewIntentItem 来执行 Activity 的 onNewIntennt 方法。
    而当需要重新创建 Activity 时，则传入的是一个 LaunchActivityItem，用来创建并启动Activity。

    *frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java*

                        executeCallbacks(transaction){
                            final int size = callbacks.size();
                            for (int i = 0; i < size; ++i) { // 遍历所有的 ClientTransactionItem 并执行了 ClientTransactionItem 的 execute 
                                final ClientTransactionItem item = callbacks.get(i);
                                item.execute(mTransactionHandler, token, mPendingActions);
    // [前面](#add_callback)添加了LaunchActivityItem。item.execute会执行到LaunchActivityItem#execute

                                LaunchActivityItem#execute(ClientTransactionHandler client, IBinder token,PendingTransactionActions pendingActions){ 
                                    client.handleLaunchActivity(r, pendingActions, null /* customIntent */){ // 这里的 ClientTransactionHandler 就是 ActivityThread
    *frameworks/base/core/java/android/app/ActivityThread.java*

                                        WindowManagerGlobal.initialize();

                                        final Activity a = performLaunchActivity(r, customIntent){
23. 创建Activity的context（ContextImpl的实例，保存到ContextWrapper成员mBase）

                                            ContextImpl appContext = createBaseContextForActivity(r);
24. **创建Activity**
                                
                                            activity = mInstrumentation.newActivity(appContext.getClassLoader(), component.getClassName(), r.intent);
                                            activity.attach(){
25. 初始化activity一些成员（如mBase, PhoneWindow）

                                                attachBaseContext(context){mBase = base;}

                                                mWindow = new PhoneWindow(this, window, activityConfigCallback);
                                                ...
                                            }

                                            mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState){
                                                activity.performCreate(icicle, persistentState){
26. **activity#onCreate**

                                                    if (persistentState != null) {
                                                        onCreate(icicle, persistentState);
                                                    } else {
                                                        onCreate(icicle);
                                                    }
                                                }
                                            }
                                        }
                                    }
                                }
                                 
                             cycleToPath(r, postExecutionState, shouldExcludeLastTransition, transaction);
                            }
                        }
27. 将 Activity 的生命周期执行到指定的状态
    *frameworks/base/core/java/android/app/servertransaction/TransactionExecutor.java*

                        executeLifecycleState(transaction){
                            final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
                            // Cycle to the state right before the final requested state.
                            cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */, transaction){
                                performLifecycleSequence(r, path, transaction){
                                    final int size = path.size();
                                    for (int i = 0, state; i < size; i++) {
                                        state = path.get(i);
                                        switch (state) {
                                            case ON_CREATE:
                                                mTransactionHandler.handleLaunchActivity(r, mPendingActions,null /* customIntent */);
                                                break;
                                            case ON_START:
                                                mTransactionHandler.handleStartActivity(r, mPendingActions,null /* activityOptions */){
    *frameworks/base/core/java/android/app/ActivityThread.java*

                                                    activity.performStart("handleStartActivity"){
                                                        mInstrumentation.callActivityOnStart(this){
28. Activity#onStart

                                                            activity.onStart();
                                                        }
                                                    }
                                                    r.setState(ON_START);
                                                }
                                                break;
                                            case ON_RESUME:
                                                mTransactionHandler.handleResumeActivity(){
                                                    performResumeActivity(r, finalStateRequest, reason){
    *frameworks/base/core/java/android/app/ActivityThread.java*

                                                        r.activity.performResume(r.startsNotResumed, reason){
                                                            mInstrumentation.callActivityOnResume(this){
                                                                activity.onResume();
                                                            }
                                                        }
                                                        r.setState(ON_RESUME);
                                                    }

29. 将 DecorView 添加到 Window
    窗口的添加过程会完成 DecorView 的布局、测量与绘制。当完成窗口的添加后 Activity 的 View 才被显示出来，才有了宽高。
    这也是我们在 onResume 中获取不到 View 宽高的原因。

                                                    final Activity a = r.activity;
                                                    if (r.window == null && !a.mFinished && willBeVisible) {
                                                        // PhoneWindow
                                                        r.window = r.activity.getWindow();
                                                        // 获取 DecorView
                                                        View decor = r.window.getDecorView();
                                                        // 先设置 DecorView 不可见
                                                        decor.setVisibility(View.INVISIBLE);
                                                        // 获取 WindowManager
                                                        ViewManager wm = a.getWindowManager();
                                                        // 获取 Window 的属性参数
                                                        WindowManager.LayoutParams l = r.window.getAttributes();
                                                        a.mDecor = decor;
                                                        l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                                                        l.softInputMode |= forwardBit;
                                                        // ...
                                                        if (a.mVisibleFromClient) {
                                                            if (!a.mWindowAdded) {
                                                                a.mWindowAdded = true;
30. 通过 WindowManager 将 DecorView添加到窗口。（该过程也很复杂，此处不展开）

                                                                wm.addView(decor, l);
                                                            } else {
                                                                a.onWindowAttributesChanged(l);
                                                            }
                                                        }
                                                        ...
31. 设置可见
                                                        
                                                        if (r.activity.mVisibleFromClient) {
                                                            r.activity.makeVisible();
                                                        }
                                                    }

                                                }
                                                break;
                                            ....
                                        } 
                                    }
                                }
                            }


                            // Execute the final transition with proper parameters.
                            lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
                            lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
                        }
                    }
            }
        }

                                                                                }
                                                                             }

                                                                        }
                                                                    } else { // 进程不存在
31. ***冷启动：需要先创建进程（进入ATMS）***

                                                                        mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? HostingRecord.HOSTING_TYPE_TOP_ACTIVITY : HostingRecord.HOSTING_TYPE_ACTIVITY){
    *frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java*

32. <span id="obtain_message"> 通过handler.sendMessage的方式继续后续流程（为了防死锁，仍然在SystemServer进程中） </span> <br>
    SystemServer进程的主循环是Looper,类似App不断的从MessageQueue中读取Message处理

                                                                            // Post message to start process to avoid possible deadlock of calling into AMS with the ATMS lock held.
                                                                            final Message m = PooledLambda.obtainMessage(ActivityManagerInternal::startProcess{
33. <span id="start_process"> 启动进程 </span> （这里是message的callback,sendMessage后会执行到这里而不会走handler的handleMessage（进入AMS））

    *frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java*

                                                                                        // 运行在SystemServer进程中的AMS服务
                                                                                        startProcessLocked(){
                                                                                            return mProcessList.startProcessLocked(){
34. 进入ProcessList.startProcessLocked
    *frameworks/base/services/core/java/com/android/server/am/ProcessList.java*

                                                                                                // 创建ProcessRecord app
                                                                                                  ...

                                                                                                startProcessLocked(app, hostingRecord, zygotePolicyFlags, abiOverride){
35. ***指定App的入口类"android.app.ActivityThread"***

                                                                                                    final String entryPoint = "android.app.ActivityThread"; // 所有app的main入口

                                                                                                    Process.ProcessStartResult startResult = startProcess(hostingRecord,
                                                                                                        entryPoint, app,  
                                                                                                        uid, gids, runtimeFlags, zygotePolicyFlags, mountExternal, seInfo,
                                                                                                        requiredAbi, instructionSet, invokeWith, startUptime){

36. 根据收集的参数确定实际的启动进程方式<br>
    新进程都是从 zygote 衍生的，但实际上有三种不同类型的 zygote:<br>
    - regular zygote: 即常规的 zygote32/zygote64 进程，是所有 Android Java 应用的父进程；
    - app zygote: 应用 zygote 进程，与常规 zygote 创建的应用相比受到更多限制；
    - webview zygote: 辅助 zygote 进程，用于创建 isolated_app 进程来渲染不可信的 web 内容，具有最为严格的安全限制；

                                                                                                        if (regularZygote){ // 常规的 zygote32/zygote64 进程
                                                                                                            startResult = Process.start(entryPoint,
                                                                                                                    app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                                                                                                                    app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                                                                                                                    app.info.dataDir, invokeWith, app.info.packageName, zygotePolicyFlags,
                                                                                                                    isTopApp, app.getDisabledCompatChanges(), pkgDataInfoMap,
                                                                                                                    allowlistedAppDataInfoMap, bindMountAppsData, bindMountAppStorageDirs,
                                                                                                                    new String[]{PROC_START_SEQ_IDENT + app.getStartSeq()}){
    *frameworks/base/core/java/android/os/Process.java*

                                                                                                                return ZYGOTE_PROCESS.start(processClass, niceName, uid, gid, gids,
                                                                                                                            runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                                                                                                                            abi, instructionSet, appDataDir, invokeWith, packageName,
                                                                                                                            zygotePolicyFlags, isTopApp, disabledCompatChanges,
                                                                                                                            pkgDataInfoMap, whitelistedDataInfoMap, bindMountAppsData,
                                                                                                                            bindMountAppStorageDirs, zygoteArgs){
    *frameworks/base/core/java/android/os/ZygoteProcess.java*

37. ***常规方式启动进程，SystemServer进程的AMS通过zygoteWriter向ZygoteServer socket（Unix local socket相较网络通信的socket更高效）写数据请求启动进程（zygote进程的主循环runSelectLoop会收到该请求）***

                                                                                                                    startViaZygote(){
                                                                                                                        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi),
                                                                                                                                                          zygotePolicyFlags,
                                                                                                                                                          argsForZygote){
                                                                                                                            return attemptZygoteSendArgsAndGetResult(zygoteState, msgStr){
                                                                                                                                ...
                                                                                                                                final BufferedWriter zygoteWriter = zygoteState.mZygoteOutputWriter;
                                                                                                                                final DataInputStream zygoteInputStream = zygoteState.mZygoteInputStream;
                                                                                                                                // 写socket,zygote侧epoll会收到
                                                                                                                                zygoteWriter.write(msgStr);
                                                                                                                                ...
        
38. ***zygote进程正在循环等待事件（epoll）***
    ***Zygote进程***
    ***frameworks/base/core/java/com/android/internal/os/ZygoteServer.java***

                                                                                                                                // zygote主循环会收到AMS的请求
                                                                                                                                caller=zygoteServer.runSelectLoop(String abiList) {

                                                                                                                                    while(true){
                                                                                                                                        ArrayList<FileDescriptor> socketFDs = new ArrayList<FileDescriptor>();
                                                                                                                                        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
                                                                                                                                        try {
                                                                                                                                            Os.poll(pollFDs, -1);  // poll 监听 fd
                                                                                                                                        } catch (ErrnoException ex) {
                                                                                                                                            throw new RuntimeException("poll failed", ex);
                                                                                                                                        }
                                                                                                                                        boolean usapPoolFDRead = false;
                                                                                                                            
                                                                                                                                        // poll 唤醒，开始收数据，处理数据
                                                                                                                                        while (--pollIndex >= 0) {
                                                                                                                                            if ((pollFDs[pollIndex].revents & POLLIN) == 0) {
                                                                                                                                                continue;
                                                                                                                                            }
                                                                                                                            
                                                                                                                                            if (pollIndex == 0) { //第一个 zygote socket fd
                                                                                                                                                // Zygote server socket
                                                                                                                            
                                                                                                                                                // 构建一个与客户端通信的 ZygoteConnection 类型的辅助对象
                                                                                                                                                ZygoteConnection newPeer = acceptCommandPeer(abiList);
                                                                                                                                                peers.add(newPeer);
                                                                                                                                                // 把与客户端连接的 fd 存入到 socketFDs
                                                                                                                                                // 下次循环，fd 会加入 pollFDs 中被监听
                                                                                                                                                socketFDs.add(newPeer.getFileDescriptor());
                                                                                                                            
                                                                                                                                            } else if (pollIndex < usapPoolEventFDIndex) { // 与客户端连接的 fd
                                                                                                                                                // Session socket accepted from the Zygote server socket

39. Zygote收到了SystemServer启动进程的请求

                                                                                                                                                try {
                                                                                                                                                    // 读取客户端发来的数据，创建新的进程，把新进程的 main 函数包装为一个 Runnable 对象
                                                                                                                                                    ZygoteConnection connection = peers.get(pollIndex);
40. ***fork一个子进程（启动的Activity对应的进程）***

                                                                                                                                                    // fork一个子进程
                                                                                                                                                    final Runnable command = connection.processOneCommand(this){
    *frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java*

                                                                                                                                                       pid = Zygote.forkAndSpecialize();
                                                                                                                                                       if (pid == 0) { 
41. 进入fork出的App进程（注意尚未调用"android.app.ActivityThread#main()"）
    ***Activity对应的App进程***

                                                                                                                                                          // 在子进程中
                                                                                                                                                          return handleChildProc(parsedArgs, childPipeFd, parsedArgs.mStartChildZygote){
                                                                                                                                                              return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                                                                                                                                                                      parsedArgs.mDisabledCompatChanges,
                                                                                                                                                                      parsedArgs.mRemainingArgs, null /* classLoader */){
    *frameworks/base/core/java/com/android/internal/os/ZygoteInit.java*
                                                                                                                                                  
                                                                                                                                                                  RuntimeInit.commonInit();
                                                                                                                                                                  ZygoteInit.nativeZygoteInit();
                                                                                                                                                                  return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,classLoader){
42. ***通过反射找到"android.app.ActivityThread#main()"，封装为Runnable返回***

    *frameworks/base/core/java/com/android/internal/os/RuntimeInit.java*
                                                                                                                                                  
                                                                                                                                                                      return findStaticMain(args.startClass, args.startArgs, classLoader); // args.startClass即为“android.app.ActivityThread”
                                                                                                                                                                  }
                                                                                                                                                              }
                                                                                                                                                          }
                                                                                                                                                       }
                                                                                                                                  
                                                                                                                                                    }
                                                                                                                            
                                                                                                                                                    // 子进程直接返回 Runnable
                                                                                                                                                    if (mIsForkChild) {  
43. 子进程（App进程）跳出Zygote的runSelectLoop死循环，外面会执行command#run, 即android.app.ActivityThread#main()

                                                                                                                                                        return command; // 子进程跳出循环，外面会执行command#run, 即android.app.ActivityThread#main()
                                                                                                                                                    } 
                                                                                                                                                    ...
                                                                                                                                                }
                                                                                                                                            } 
                                                                                                                                        }
                                                                                                                                    }
                                                                                                                                }
        
                                                                                                                                if (caller != null) {

                                                                                                                                    caller.run(){
44. ***走到这里已经是子进程（App进程），执行android.app.ActivityThread#main()***
 
        public static void main(String[] args) {
    ***frameworks/base/core/java/android/app/ActivityThread.java***

45. ***创建主线程Looper***

        Looper.prepareMainLooper(){
            // allowquit=false，表示不允许退出。普通的looper是允许退出的，主线程的不允许。
            prepare(false){
                if (sThreadLocal.get() != null) {
                    throw new RuntimeException("Only one Looper may be created per thread");
                }
                sThreadLocal.set(new Looper(quitAllowed){ // ThreadLocal表明每个Thread只能有一个Looper且和别人的不一样
                    mQueue = new MessageQueue(quitAllowed);  // 每个Looper有且仅有一个MessageQueue
                    mThread = Thread.currentThread();
                });
            }
            synchronized (Looper.class) {
                if (sMainLooper != null) { // 每个应用只有一个mainLooper
                    throw new IllegalStateException("The main Looper has already been prepared.");
                }
                sMainLooper = myLooper();
            }
        }
46. ***创建ActivityThread，同时初始化了一些重要成员如ApplicationThread用于RPC,Looper和Handler用于消息投递。***

        ActivityThread thread = new ActivityThread(){
            // ApplicationThread extends IApplicationThread.Stub，用于Binder RPC。
            // ATMS 可以通过调用 ApplicationThread 的 Binder 客户端对象提供的接口，远程调用到 App 端，更新 Activity 的状态
            final ApplicationThread mAppThread = new ApplicationThread(); 
            final Looper mLooper = Looper.myLooper(); // 主线Looper
            final H mH = new H();
            ...
        }

47. ***App进程attach到ActivityManagerService***

        thread.attach(false, startSeq){
48. 获取proxy，通过binder机制（RPC）调到ActivityManagerService#attachApplication

            // IActivityManager extends IInterface。binder机制中的用户接口定义
            final IActivityManager mgr = ActivityManager.getService();
            // binder机制中的proxy，远程过程调用（RPC）调到Stub（class ActivityManagerService extends IActivityManager.Stub）中对应的attachApplication
            mgr.attachApplication(mAppThread, startSeq){

49. 切进程：*App->SystemServer*
    ***frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java***

                attachApplicationLocked(thread, callingPid, callingUid, startSeq){

                    ProcessRecord app;

50. AMS记录app相关信息方便管理

                    ...

51. ***RPC回到ActivityThread。此处的thread即为ActivityThread的ApplicationThread mAppThread，AMS用它主动和App沟通***

                    thread.bindApplication(){
52. 切进程：*SystemServer->App*
    
    *frameworks/base/core/java/android/app/ActivityThread.java*

53. 封装参数到AppBindData data，然后发送BIND_APPLICATION消息进行后续处理

54. <span id="cache_service"> 缓存一些服务到ServiceManager（getSystemService时用到）</span>

                        // Setup the service cache in the ServiceManager
                        ServiceManager.initServiceCache(services){sCache.putAll(services);}

                        ...
                        sendMessage(H.BIND_APPLICATION, data);

                        // 消息接收：
                        public void handleMessage(Message msg) {
                            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
                            switch (msg.what) {
                                case BIND_APPLICATION:
                                    AppBindData data = (AppBindData)msg.obj;
55. 处理BIND_APPLICATION消息

                                    handleBindApplication(data){

                                       if (data.debugMode != ApplicationThreadConstants.DEBUG_OFF) {
56. *如果开发者设置里面设置了应用启动时等待调试器，此处等待调试器*
                               
                                           if (data.debugMode == ApplicationThreadConstants.DEBUG_WAIT) {
                                               Slog.w(TAG, "Application " + data.info.getPackageName() + " is waiting for the debugger on port 8100...");
                               
                                               IActivityManager mgr = ActivityManager.getService();
                                               try {
                                                   mgr.showWaitingForDebugger(mAppThread, true);
                                               } catch (RemoteException ex) {
                                                   throw ex.rethrowFromSystemServer();
                                               }
                                               ...
                                       }

                                        // 该app即为我们的Application实例
                                        Application app = data.info.makeApplicationInner(data.restrictedBackupMode, null){
57. LoadedApk#makeApplicationInner
    *frameworks/base/core/java/android/app/LoadedApk.java*

58. 创建Application的context（ContextImpl的实例，保存到ContextWrapper成员mBase）

                                            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
                                            app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext){
    *frameworks/base/core/java/android/app/Instrumentation.java*

59. ***通过反射创建android.app.Application实例***

                                                Application app = getFactory(context.getPackageName()).instantiateApplication(cl, className);

                                                app.attach(context){
60. 回调Application#attachBaseContext(先于onCreate)
    *frameworks/base/core/java/android/app/Application.java*

                                                    attachBaseContext(context);

                                                    mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
                                                }
                                                return app;
                                            }
                                        }

                                        mInstrumentation.callApplicationOnCreate(app){
61. ***回调Application#onCreate()***

                                            app.onCreate(); 
                                        }
                                    }
                                    break;
                    }
62. thread.attach执行后继续在*SystemServer*进程执行
    *SystemServer进程*
    *frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java*

                    // See if the top visible activity is waiting to run in this process...
                    if (normalMode) {
                        mAtmInternal.attachApplication(app.getWindowProcessController()){
    *frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java*

                            mRootWindowContainer.attachApplication(wpc){
    *frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java*

                                process(WindowProcessController app) {
                                   mApp = app;
                                   for (int displayNdx = getChildCount() - 1; displayNdx >= 0; --displayNdx) {
                                       getChildAt(displayNdx).forAllRootTasks(this){
    *frameworks/base/services/core/java/com/android/server/wm/WindowContainer.java*

                                           task#forAllRootTasks(Consumer<Task> callback, boolean traverseTopToBottom){
    *frameworks/base/services/core/java/com/android/server/wm/Task.java*

                                               callback.accept(this){ // callback=RootWindowContainer#AttachApplicationHelper
    *frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java*

                                                   rootTask.forAllActivities(this){  // Task是WindowContainer的子类。Task : TaskFragment : WindowContainer
    *frameworks/base/services/core/java/com/android/server/wm/WindowContainer.java*

                                                       activityRecord.forAllActivities(Predicate<ActivityRecord> callback, boolean traverseTopToBottom){
    *frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java*

                                                           callback.test(this){ // callback=RootWindowContainer#AttachApplicationHelper
    *frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java*

                                                               mTaskSupervisor.realStartActivityLocked(){
63. **后续跟前面的“[15](#hot_start) 如果目标 Activity 所在的应用进程已经在运行则直接在该进程中启动该Activity”一样的流程了** 

    *frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java*

                                                               }
                                                           }
                                                       }
                                                   }
                                               }
                                           }
                                       }
                                   }
                                }
                            }
                        }
                    }
64. 如果有待启动的服务则启动这些服务
    （startService时若宿主进程不存在则会先记录下Service,然后启动进程，此时进程已经启动服务也可以继续启动了）

                    // Find any services that should be running in this process...
                    if (!badApp) {
                        mServices.attachApplicationLocked(app, processName);
                    }
                }
            }
        }

65. 主线程陷入Looper死循环，处理各种event

        Looper.loop();

    }


                                                                                                                                    }
                                                                                                                                }
        
                                                                                                                            }
                                                                                                                        }
                                                                                                                    }
                                                                                                                }
                                                                                                            }
                                                                                                        }
                                                                                                    }
                                                                                                }
                                                                                            }
                                                                                        }
                                                                                    },
        
                                                                                    mAmInternal, activity.processName, activity.info.applicationInfo, knownToBeDead,
                                                                                    isTop, hostingType, activity.intent.getComponent());


[refer to 17](#obtain_message)            

                                                                            /*handler发送消息先把消息投递到messageQueue,messageQueue和Looper一一对应，looper不断循环地从messageQueue中读取消息处理。
                                                                             取出消息m后，调用m.target.dispatchMessage(m)，该target即为投递m的handler。dispatchMessage会先判断m有没有设置callback,
                                                                             如果有就调用callback,结束。如果没有再判断有没有handler#callback,最后才会handleMessage。
                                                                             因为m前面设置了callback=ctivityManagerInternal::startProcess，所以此处sendMessage实际的效果是在handler对应的Thread执行ctivityManagerInternal::startProcess
                                                                            */
                                                                            mH.sendMessage(m); 
                                                                        }
                                                                    }
                                                                }
                                                            }
                                                        }
                                                    }
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }
            
                }

                if (ar != null) {
                    mMainThread.sendActivityResult(
                        mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                        ar.getResultData());
                }
            }
        }
    }



## getSystemService流程

1. call Activity#getSystemService(Context.POWER_SERVICE)
    *App进程*
    *frameworks/base/core/java/android/app/Activity.java*

        Activity#getSystemService(@ServiceName @NonNull String name) {
2. call ContextImpl#getSystemService

            ContextImpl#getSystemService(String name){ // Activity#mBase是ContextImpl实例，详见Activity的创建流程
                SystemServiceRegistry.getSystemService(this, name){

                }/* SystemServiceRegistry已经静态注册了许多服务，这些服务在使用时才被创建（调用createService）：
                   static {
                        registerService(Context.POWER_SERVICE, PowerManager.class,
                                new CachedServiceFetcher<PowerManager>() {
                            @Override
                            public PowerManager createService(ContextImpl ctx) throws ServiceNotFoundException {
                                IBinder powerBinder = ServiceManager.getServiceOrThrow(Context.POWER_SERVICE){
                                     final IBinder binder = getService(name){
3. 从缓存查找服务 [缓存](#cache_service)

                                         IBinder service = sCache.get(name);
                                         if (service != null) {
                                             return service;
                                         } else {
4. 若缓存没有则从native层获取

                                             return Binder.allowBlocking(rawGetService(name));
                                         }
                                     }
                                }
5. 客户拿到的XXXManager都是代理，真正的服务运行在SystemServer进程

                                IPowerManager powerService = IPowerManager.Stub.asInterface(powerBinder);
                                IBinder thermalBinder = ServiceManager.getServiceOrThrow(Context.THERMAL_SERVICE);
                                IThermalService thermalService = IThermalService.Stub.asInterface(thermalBinder);
                                return new PowerManager(ctx.getOuterContext(), powerService, thermalService,
                                        ctx.mMainThread.getHandler());
                            }});
                
                        ...
                 */
            }
        }


## startService流程

````

//============= 调用方进程
 //============= frameworks/base/core/java/android/app/ContextImpl.java

   public ComponentName startService(Intent service) {

    warnIfCallingFromSystemProcess();
    
    return startServiceCommon(service, false, mUser){
        validateServiceIntent(service); // API 21开始intent必须是显示
        service.prepareToLeaveProcess(this); // 跨进程准备
        ComponentName cn = ActivityManager.getService() // 获取IActivityManager#Stub#Proxy实例
        .startService( // 通过Binder机制RPC调用到服务端IActivityManager.Stub#startService，即SystemServer进程的ActivityManagerService#startService
                mMainThread.getApplicationThread(),  // 调用方的Binder Stub，供SystemServer RPC回调过来
                service,
                service.resolveTypeIfNeeded(getContentResolver()), 
                requireForeground, // 是否前台服务方式启动（startForegroundService时为true,此处为false）
                getOpPackageName(), getAttributionTag(), user.getIdentifier());
                
                
            //>============= SystemServer进程
              //============= frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
              
                public ComponentName startService(IApplicationThread caller, Intent service,
                    String resolvedType, boolean requireForeground, String callingPackage,
                    String callingFeatureId, int userId)
                    throws TransactionTooLargeException {
                    enforceNotIsolatedCaller("startService"); // 隔离进程（如android:isolatedProcess=true的service）不允许调用startService
                    enforceAllowedToStartOrBindServiceIfSdkSandbox(service); // 如果caller是sdk沙盒进程（运行在SDK Runtime）确保其有权限启动服务
                    // Refuse possible leaked file descriptors
                    if (service != null && service.hasFileDescriptors() == true) { // 禁止通过intent传递文件描述符
                        throw new IllegalArgumentException("File descriptors passed in Intent");
                    }
                    ...
                    
                    final int callingPid = Binder.getCallingPid();
                    final int callingUid = Binder.getCallingUid();
                    final long origId = Binder.clearCallingIdentity();
                    ComponentName res;
                    try {
                        res = mServices.startServiceLocked(caller, service,
                                resolvedType, callingPid, callingUid,
                                requireForeground, callingPackage, callingFeatureId, userId){
                                
                          //================== frameworks/base/services/core/java/com/android/server/am/ActiveServices.java     
                               
                            //判断调用方是否为前台    
                            final boolean callerFg;
                            if (caller != null) {
                                final ProcessRecord callerApp = mAm.getRecordForAppLOSP(caller);
                                callerFg = callerApp.mState.getSetSchedGroup() != ProcessList.SCHED_GROUP_BACKGROUND;
                            } else {
                                callerFg = true;
                            }
                            
                            //查找待启动Service。其中会进行ServiceRecord的创建（如果缓存中没有），
                            // 还会进行各种校验，如目标Service是否是export的，是否带权限且caller是否具备了该权限
                            // 如果，校验失败则会返回serviceRecord为null
                            ServiceLookupResult res = retrieveServiceLocked(service, null, resolvedType, callingPackage,
                                        callingPid, callingUid, userId, true, callerFg, false, false);
                            ...
                            
                            // 设置前台服务的一些限制
                            ServiceRecord r = res.record;
                            setFgsRestrictionLocked(callingPackage, callingPid, callingUid, service, r, userId,
                                    allowBackgroundActivityStarts);
                            ...
                            
                            //Service所在应用未启动或处在后台
                            final boolean bgLaunch = !mAm.isUidActiveLOSP(r.appInfo.uid);
                    
                            // If the app has strict background restrictions, we treat any bg service
                            // start analogously to the legacy-app forced-restrictions case, regardless
                            // of its target SDK version.
                            //检查Service所在应用后台启动限制
                            boolean forcedStandby = false;
                            if (bgLaunch && appRestrictedAnyInBackground(r.appInfo.uid, r.packageName)) {
                                if (DEBUG_FOREGROUND_SERVICE) {
                                    Slog.d(TAG, "Forcing bg-only service start only for " + r.shortInstanceName
                                            + " : bgLaunch=" + bgLaunch + " callerFg=" + callerFg);
                                }
                                forcedStandby = true;
                            }
                    
                            // 如果是启动前台服务（startForegroundService()）检查是否允许
                            if (fgRequired) {
                                logFgsBackgroundStart(r);
                                if (r.mAllowStartForeground == REASON_DENIED && isBgFgsRestrictionEnabled(r)) {
                                    String msg = "startForegroundService() not allowed due to "
                                            + "mAllowStartForeground false: service "
                                            + r.shortInstanceName;
                                    ...
                                    return null;
                                }
                            }
                    
                            // If this is a direct-to-foreground start, make sure it is allowed as per the app op.
                            boolean forceSilentAbort = false;
                            if (fgRequired) { //作为前台服务启动
                                final int mode = mAm.getAppOpsManager().checkOpNoThrow(
                                        AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName);
                                switch (mode) {
                                     //默认和允许都可以作为前台服务启动
                                    case AppOpsManager.MODE_ALLOWED:
                                    case AppOpsManager.MODE_DEFAULT:
                                        // All okay.
                                        break;
                                    //不允许的话，回退到作为普通后台服务启动
                                    case AppOpsManager.MODE_IGNORED:
                                        // Not allowed, fall back to normal start service, failing siliently
                                        // if background check restricts that.
                                        Slog.w(TAG, "startForegroundService not allowed due to app op: service "
                                                + service + " to " + r.shortInstanceName
                                                + " from pid=" + callingPid + " uid=" + callingUid
                                                + " pkg=" + callingPackage);
                                        fgRequired = false;
                                        forceSilentAbort = true;
                                        break;
                                    default:
                                    //错误的话直接返回，由上层抛出SecurityException异常
                                        return new ComponentName("!!", "foreground not allowed as per app op");
                                }
                            }
                    
                            // If this isn't a direct-to-foreground start, check our ability to kick off an
                            // arbitrary service
                            if (forcedStandby || (!r.startRequested && !fgRequired)) { // 如果不是startService方式启动且非前台服务
                                // Before going further -- if this app is not allowed to start services in the
                                // background, then at this point we aren't going to let it period.
                                //服务是否允许在后台启动
                                final int allowed = mAm.getAppStartModeLOSP(r.appInfo.uid, r.packageName,
                                        r.appInfo.targetSdkVersion, callingPid, false, false, forcedStandby);
                                //如果不允许，则无法启动服务
                                if (allowed != ActivityManager.APP_START_MODE_NORMAL) {
                                    if (allowed == ActivityManager.APP_START_MODE_DELAYED || forceSilentAbort) {
                                        //静默的停止启动
                                        return null;
                                    }
                                    if (forcedStandby) {
                                        // This is an O+ app, but we might be here because the user has placed
                                        // it under strict background restrictions.  Don't punish the app if it's
                                        // trying to do the right thing but we're denying it for that reason.
                                        if (fgRequired) {
                                            if (DEBUG_BACKGROUND_CHECK) {
                                                Slog.v(TAG, "Silently dropping foreground service launch due to FAS");
                                            }
                                            return null;
                                        }
                                    }
                                    //明确的告知不允许启动，上层抛出异常
                                    UidRecord uidRec = mAm.mProcessList.getUidRecordLOSP(r.appInfo.uid);
                                    return new ComponentName("?", "app is in background uid " + uidRec);
                                }
                            }
                            
                            //对于targetSdk 26以下（Android 8.0以下）的应用来说，不需要作为前台服务启动
                            if (r.appInfo.targetSdkVersion < Build.VERSION_CODES.O && fgRequired) {
                                if (DEBUG_BACKGROUND_CHECK || DEBUG_FOREGROUND_SERVICE) {
                                    Slog.i(TAG, "startForegroundService() but host targets "
                                            + r.appInfo.targetSdkVersion + " - not requiring startForeground()");
                                }
                                fgRequired = false;
                            }
                    
                            // The package could be frozen (meaning it's doing surgery), defer the actual
                            // start until the package is unfrozen.
                            if (deferServiceBringupIfFrozenLocked(r, service, callingPackage, callingFeatureId,
                                    callingUid, callingPid, fgRequired, callerFg, userId, allowBackgroundActivityStarts,
                                    backgroundActivityStartsToken, false, null)) {
                                return null;
                            }
                    
                            //如果待启动的Service需要相应权限，则需要用户手动确认权限后，再进行启动
                            if (!requestStartTargetPermissionsReviewIfNeededLocked(r, callingPackage, callingFeatureId,
                                    callingUid, service, callerFg, userId, false, null)) {
                                return null;
                            }
                    
                            final ComponentName realResult =
                                    startServiceInnerLocked(r, service, callingUid, callingPid, fgRequired, callerFg,
                                    allowBackgroundActivityStarts, backgroundActivityStartsToken){
                                NeededUriGrants neededGrants = mAm.mUgmInternal.checkGrantUriPermissionFromIntent(
                                        service, callingUid, r.packageName, r.userId);
                                if (unscheduleServiceRestartLocked(r, callingUid, false)) {
                                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "START SERVICE WHILE RESTART PENDING: " + r);
                                }
                                final boolean wasStartRequested = r.startRequested;
                                r.lastActivity = SystemClock.uptimeMillis();
                                r.startRequested = true;   //表示Service是由startService方式所启动的
                                r.delayedStop = false;
                                r.fgRequired = fgRequired;  //是否作为前台服务启动
                                r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                                        service, neededGrants, callingUid)); //构造启动参数
                        
                                if (fgRequired) {     //作为前台服务启动
                                    // 记录、监控
                                    ...
                                }
                        
                                final ServiceMap smap = getServiceMapLocked(r.userId);
                                boolean addToStarting = false;
                                //对于后台启动的非前台服务，需要判断其是否需要延迟启动
                                if (!callerFg && !fgRequired && r.app == null
                                        && mAm.mUserController.hasStartedUserState(r.userId)) {
                                    ...
                                       
                                    //如果当前正在后台启动的Service数大于等于允许同时在后台启动的最大服务数
                                    //将这个Service设置为延迟启动
                                    if (smap.mStartingBackground.size() >= mMaxStartingBackground) {
                                        // Something else is starting, delay!
                                        smap.mDelayedStartList.add(r);
                                        r.delayed = true;
                                        return r.name;
                                    }
                                    //添加到正在启动服务列表中
                                    addToStarting = true;

                                } 
                                ...
                                
                                //如果允许Service后台启动Activity，则将其加入到白名单中
                                if (allowBackgroundActivityStarts) {
                                    r.allowBgActivityStartsOnServiceStart(backgroundActivityStartsToken);
                                }
                                
                                ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting,
                                        callingUid, wasStartRequested){
                                    ...
                                    
                                    r.callStart = false;
                                    final int uid = r.appInfo.uid;
                                    final String packageName = r.name.getPackageName();
                                    final String serviceName = r.name.getClassName();
                                    FrameworkStatsLog.write(FrameworkStatsLog.SERVICE_STATE_CHANGED, uid, packageName,
                                            serviceName, FrameworkStatsLog.SERVICE_STATE_CHANGED__STATE__START);
                                    mAm.mBatteryStatsService.noteServiceStartRunning(uid, packageName, serviceName);
                                    //拉起服务，如果服务未启动，则会启动服务并调用其onCreate和onStartCommand方法
                                    //如果服务已启动，由于之前构造了启动参数，则会直接调用其onStartCommand方法
                                    String error = bringUpServiceLocked(r, service.getFlags(), callerFg,
                                            false /* whileRestarting */,
                                            false /* permissionsReviewRequired */,
                                            false /* packageFrozen */,
                                            true /* enqueueOomAdj */){
                                            
                                        //如果Service所在的进程存在，并且其IApplicationThread也存在
                                        //说明服务已启动（因为在启动服务时，会给ServiceRecord.app赋值，并且app.thread不为null说明进程没有被杀死）
                                        //此时直接拉起Service.onStartCommand方法                               
                                        if (r.app != null && r.app.getThread() != null) {
                                            sendServiceArgsLocked(r, execInFg, false);
                                            return null;
                                        }
                                
                                        if (!whileRestarting && mRestartingServices.contains(r)) {
                                            //如果服务正在重启中，则什么都不做，直接返回
                                            return null;
                                        }
                                
                                        //Service正在启动，将其从重启中服务列表中移除，并清除其重启中状态
                                        if (mRestartingServices.remove(r)) {
                                            clearRestartingIfNeededLocked(r);
                                        }
                                
                                        //走到这里，需要确保此服务不再被视为延迟启动，同时将其从延迟启动服务列表中移除
                                        if (r.delayed) {
                                            getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
                                            r.delayed = false;
                                        }
                                
                                        //确保Service所在的用户已启动
                                        if (!mAm.mUserController.hasStartedUserState(r.userId)) {
                                            String msg = "Unable to launch app "
                                                    + r.appInfo.packageName + "/"
                                                    + r.appInfo.uid + " for service "
                                                    + r.intent.getIntent() + ": user " + r.userId + " is stopped";
                                            Slog.w(TAG, msg);
                                            bringDownServiceLocked(r, enqueueOomAdj);
                                            return msg;
                                        }
                                
                                        // Report usage if binding is from a different package except for explicitly exempted
                                        // bindings
                                        if (!r.appInfo.packageName.equals(r.mRecentCallingPackage)
                                                && !r.isNotAppComponentUsage) {
                                            mAm.mUsageStatsService.reportEvent(
                                                    r.packageName, r.userId, UsageEvents.Event.APP_COMPONENT_USED);
                                        }
                                
                                        //Service即将启动，Service所属的App不该为stopped状态
                                        //将App状态置为unstopped，设置休眠状态为false
                                        AppGlobals.getPackageManager().setPackageStoppedState(
                                                r.packageName, false, r.userId);
                                
                                        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0; //服务所在进程是否为隔离进程
                                        final String procName = r.processName;
                                        HostingRecord hostingRecord = new HostingRecord(
                                                HostingRecord.HOSTING_TYPE_SERVICE, r.instanceName,
                                                r.definingPackageName, r.definingUid, r.serviceInfo.processName,
                                                getHostingRecordTriggerType(r));
                                        ProcessRecord app;
                                
                                        if (!isolated) {
                                            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid);
                                            if (app != null) {
                                                final IApplicationThread thread = app.getThread();
                                                final int pid = app.getPid();
                                                final UidRecord uidRecord = app.getUidRecord();
                                                if (thread != null) { // service所在进程已启动
                                                    try {
                                                        //将App添加至进程中运行的包列表中
                                                        app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode,
                                                                mAm.mProcessStats);
                                                        //接着启动Service
                                                        realStartServiceLocked(r, app, thread, pid, uidRecord, execInFg,enqueueOomAdj){
                                                            ...
                                                            
                                                            //为ServiceRecord设置所属进程                                                
                                                            r.setProcess(app, thread, pid, uidRecord);
                                                            r.restartTime = r.lastActivity = SystemClock.uptimeMillis();
                                                    
                                                            //在此进程中将Service记录为运行中
                                                            //返回值为此Service是否之前未启动
                                                            final ProcessServiceRecord psr = app.mServices;
                                                            final boolean newService = psr.startService(r);
                                                            //记录Service执行操作并设置超时回调
                                                            //前台服务超时时间为20s，后台服务超时时间为200s，超时则报ANR
                                                            bumpServiceExecutingLocked(r, execInFg, "create", null /* oomAdjReason */){
                                                                scheduleServiceTimeoutLocked(r.app){
                                                                    Message msg = mAm.mHandler.obtainMessage(ActivityManagerService.SERVICE_TIMEOUT_MSG);
                                                                    mAm.mHandler.sendMessageDelayed(msg, proc.mServices.shouldExecServicesFg()
                                                                        ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT){
                                                                    
                                                                    public void handleMessage(Message msg) {
                                                                        switch (msg.what) {
                                                                        case SERVICE_TIMEOUT_MSG: {
                                                                            mServices.serviceTimeout((ProcessRecord) msg.obj){
                                                                              //=============== frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
                                                                                mAm.mAnrHelper.appNotResponding(proc, anrMessage){ 
                                                                                  // 所有类型的ANR（service/provider/broadcast/input）最终都会走到这里
                                                                                    //============ frameworks/base/services/core/java/com/android/server/am/AnrHelper.java
                                                                                      ...
                                                                                        ...
                                                                                        //============== frameworks/base/services/core/java/com/android/server/am/AppErrors.java
                                                                                            void handleShowAnrUi(Message msg) {
                                                                                                Slog.d(TAG, "ANR delay completed. Showing ANR dialog for package: "+ packageName);
                                                                                                // 弹出ANR对话框
                                                                                                errState.getDialogController().showAnrDialogs(data);
                                                                                            }
                                                                                }
                                                                            }
                                                                        } break;
                                                                    }
                                                                }
                                                            }
                                                            //更新进程优先级
                                                            mAm.updateLruProcessLocked(app, false, null);
                                                            updateServiceForegroundLocked(psr, /* oomAdj= */ false);
                                                            // Force an immediate oomAdjUpdate, so the client app could be in the correct process state
                                                            // before doing any service related transactions
                                                            mAm.enqueueOomAdjTargetLocked(app);
                                                            mAm.updateOomAdjLocked(app, OomAdjuster.OOM_ADJ_REASON_START_SERVICE);
                                                    
                                                            boolean created = false;
                                                            try {
                                                    
                                                                final int uid = r.appInfo.uid;
                                                                final String packageName = r.name.getPackageName();
                                                                final String serviceName = r.name.getClassName();
                                                                FrameworkStatsLog.write(FrameworkStatsLog.SERVICE_LAUNCH_REPORTED, uid, packageName,
                                                                        serviceName);
                                                                mAm.mBatteryStatsService.noteServiceStartLaunch(uid, packageName, serviceName);
                                                                mAm.notifyPackageUse(r.serviceInfo.packageName, PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
                                                                
                                                                //回到App进程，调度创建Service                     
                                                                thread.scheduleCreateService(r, r.serviceInfo,
                                                                        mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                                                                        app.mState.getReportedProcState()){
                                                                    //================ 目标Service所在进程      
                                                                      //================ frameworks/base/core/java/android/app/ActivityThread.java      
                                                                        updateProcessState(processState, false);     //更新进程状态
                                                                        CreateServiceData s = new CreateServiceData();
                                                                        s.token = token; // 此处的token即为ServiceRecord extends IBinder
                                                                        s.info = info;
                                                                        s.compatInfo = compatInfo;
                                                            
                                                                        sendMessage(H.CREATE_SERVICE, s){
                                                                            handleMessage(Message msg) {
                                                                                switch (msg.what) {
                                                                                    ...
                                                                                    case CREATE_SERVICE:
                                                                                    handleCreateService((CreateServiceData)msg.obj){
                                                                                        // If we are getting ready to gc after going to the background, well
                                                                                        // we are back active so skip it.
                                                                                        unscheduleGcIdler();     //此时不要进行GC
                                                                                
                                                                                        LoadedApk packageInfo = getPackageInfoNoCheck(
                                                                                                data.info.applicationInfo, data.compatInfo);
                                                                                        Service service = null;
                                                                                        try {
                                                                                            //创建或获取Application（到了这里进程的初始化应该都完成了，所以是直接获取Application）
                                                                                            Application app = packageInfo.makeApplicationInner(false, mInstrumentation);
                                                                                
                                                                                            final java.lang.ClassLoader cl;
                                                                                            if (data.info.splitName != null) {
                                                                                                cl = packageInfo.getSplitClassLoader(data.info.splitName);
                                                                                            } else {
                                                                                                cl = packageInfo.getClassLoader();
                                                                                            }
                                                                                            //通过AppComponentFactory反射创建Service实例
                                                                                            service = packageInfo.getAppFactory()
                                                                                                    .instantiateService(cl, data.info.name, data.intent);
                                                                                            ContextImpl context = ContextImpl.getImpl(service
                                                                                                    .createServiceBaseContext(this, packageInfo));
                                                                                            if (data.info.splitName != null) {
                                                                                                context = (ContextImpl) context.createContextForSplit(data.info.splitName);
                                                                                            }
                                                                                            if (data.info.attributionTags != null && data.info.attributionTags.length > 0) {
                                                                                                final String attributionTag = data.info.attributionTags[0];
                                                                                                context = (ContextImpl) context.createAttributionContext(attributionTag);
                                                                                            }
                                                                                            // Service resources must be initialized with the same loaders as the application
                                                                                            // context.
                                                                                            context.getResources().addLoaders(
                                                                                                    app.getResources().getLoaders().toArray(new ResourcesLoader[0]));
                                                                                
                                                                                            context.setOuterContext(service);
                                                                                            //初始化
                                                                                            service.attach(context, this, data.info.name, data.token, app,
                                                                                                    ActivityManager.getService());
                                                                                            //执行onCreate回调
                                                                                            service.onCreate();
                                                                                            // 记录运行中的service
                                                                                            mServicesData.put(data.token, data);
                                                                                            mServices.put(data.token, // 此处的token即为ServiceRecord（继承了IBinder）
                                                                                            service);
                                                                                            try {
                                                                                                //Service相关任务执行完成
                                                                                                //这一步中会把之前的启动超时定时器取消
                                                                                                ActivityManager.getService().serviceDoneExecuting(
                                                                                                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0){
                                                                                                  //======== SystemServer进程
                                                                                                    //========== frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java  
                                                                                                        mServices.serviceDoneExecutingLocked((ServiceRecord) token, type, startId, res, false){
                                                                                                          //============== frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
                                                                                                            serviceDoneExecutingLocked(r, inDestroying, inDestroying, enqueueOomAdj){
                                                                                                                if (psr.numberOfExecutingServices() == 0) {
                                                                                                                    // 删除启动服务超时（超时会触发ANR）
                                                                                                                    mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
                                                                                                                }
                                                                                                            }
                                                                                                        }
                                                                                                }
                                                                                            } catch (RemoteException e) {
                                                                                                throw e.rethrowFromSystemServer();
                                                                                            }
                                                                                        } catch (Exception e) {
                                                                                            if (!mInstrumentation.onException(service, e)) {
                                                                                                throw new RuntimeException(
                                                                                                    "Unable to create service " + data.info.name
                                                                                                    + ": " + e.toString(), e);
                                                                                            }
                                                                                        }
                                                                                    }
    
                                                                                    break;
                                                                                    ...
                                                                                }
                                                                                ...
                                                                            }
                                                                        }
                                                                }
                                                                        
                                                                //显示前台服务通知        
                                                                r.postNotification();
                                                                created = true;
                                                            } catch (DeadObjectException e) {
                                                                Slog.w(TAG, "Application dead when creating service " + r);
                                                                //杀死进程
                                                                mAm.appDiedLocked(app, "Died when creating service");
                                                                throw e;
                                                            } finally {
                                                                if (!created) {
                                                                    // Keep the executeNesting count accurate.
                                                                    final boolean inDestroying = mDestroyingServices.contains(r);
                                                                    serviceDoneExecutingLocked(r, inDestroying, inDestroying, false);
                                                    
                                                                    // Cleanup.
                                                                    if (newService) {
                                                                        psr.stopService(r);
                                                                        r.setProcess(null, null, 0, null);
                                                                    }
                                                    
                                                                    // Retry.
                                                                    if (!inDestroying) {
                                                                        scheduleServiceRestartLocked(r, false);
                                                                    }
                                                                }
                                                            }
                                                    
                                                            if (r.allowlistManager) {  //允许管理白名单，如省电模式白名单
                                                                psr.mAllowlistManager = true;
                                                            }
                                                    
                                                            //执行Service.onBind方法（通过bindService启动的情况下）
                                                            requestServiceBindingsLocked(r, execInFg);
                                                            
                                                            //更新是否有与Service建立连接的Activity
                                                            updateServiceClientActivitiesLocked(psr, null, true);
                                                    
                                                            if (newService && created) {
                                                                psr.addBoundClientUidsOfNewService(r);
                                                            }
                                                    
                                                            // If the service is in the started state, and there are no
                                                            // pending arguments, then fake up one so its onStartCommand() will
                                                            // be called.
                                                            //如果Service已经启动，并且没有启动项，则构建一个假的启动参数供onStartCommand使用
                                                            if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
                                                                r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                                                                        null, null, 0));
                                                            }
                                                            
                                                            //拉起Service.onStartCommand方法
                                                            sendServiceArgsLocked(r, execInFg, true){
                                                                final int N = r.pendingStarts.size();
                                                                if (N == 0) {     //如果待启动项列表中没有内容则直接返回
                                                                    return;
                                                                }
                                                        
                                                                ArrayList<ServiceStartArgs> args = new ArrayList<>();
                                                        
                                                                while (r.pendingStarts.size() > 0) {
                                                                    ServiceRecord.StartItem si = r.pendingStarts.remove(0);
                                                                    if (si.intent == null && N > 1) {
                                                                        //如果在多个启动项中有假启动项，则跳过假启动项
                                                                        //但如果这个假启动项是唯一的启动项则不要跳过它，这是为了支持onStartCommand(null)的情况
                                                                        continue;
                                                                    }
                                                                    si.deliveredTime = SystemClock.uptimeMillis();
                                                                    r.deliveredStarts.add(si);
                                                                    si.deliveryCount++;
                                                                    //处理Uri权限
                                                                    if (si.neededGrants != null) {
                                                                        mAm.mUgmInternal.grantUriPermissionUncheckedFromIntent(si.neededGrants,
                                                                                si.getUriPermissionsLocked());
                                                                    }
                                                                    //授权访问权限
                                                                    mAm.grantImplicitAccess(r.userId, si.intent, si.callingId,
                                                                            UserHandle.getAppId(r.appInfo.uid)
                                                                    );
                                                                    
                                                                    //记录Service执行操作并设置超时回调
                                                                    //前台服务超时时间为20s，后台服务超时时间为200s
                                                                    bumpServiceExecutingLocked(r, execInFg, "start", null /* oomAdjReason */);
                                                                    
                                                                    //如果是以前台服务的方式启动的Service（startForegroundService），并且之前没有设置启动前台服务超时回调
                                                                    if (r.fgRequired && !r.fgWaiting) {
                                                                        //如果当前服务还没成为前台服务，设置启动前台服务超时回调
                                                                        //在10s内需要调用Service.startForeground成为前台服务，否则停止服务
                                                                        //注：Android 11这个超时时间是10s，在后面的Android版本中这个时间有变化
                                                                        if (!r.isForeground) {
                                                                            scheduleServiceForegroundTransitionTimeoutLocked(r);
                                                                        } else {
                                                                            r.fgRequired = false;
                                                                        }
                                                                    }
                                                                    int flags = 0;
                                                                    if (si.deliveryCount > 1) {
                                                                        flags |= Service.START_FLAG_RETRY;
                                                                    }
                                                                    if (si.doneExecutingCount > 0) {
                                                                        flags |= Service.START_FLAG_REDELIVERY;
                                                                    }        
                                                                    //添加启动项
                                                                    args.add(new ServiceStartArgs(si.taskRemoved, si.id, flags, si.intent));
                                                                }
                                                        
                                                                if (!oomAdjusted) {
                                                                    mAm.enqueueOomAdjTargetLocked(r.app);
                                                                    mAm.updateOomAdjPendingTargetsLocked(OomAdjuster.OOM_ADJ_REASON_START_SERVICE);
                                                                }
                                                                //构建出一个支持Binder跨进程传输大量数据的列表来传输启动参数数据
                                                                ParceledListSlice<ServiceStartArgs> slice = new ParceledListSlice<>(args);
                                                                slice.setInlineCountLimit(4);
                                                                
                                                                //回到App进程，调度启动Service
                                                                r.app.getThread().scheduleServiceArgs(r, slice){
                                                                    //================ 目标Service所在进程      
                                                                      //================ frameworks/base/core/java/android/app/ActivityThread.java    
                                                                        ...
                                                                        sendMessage(H.SERVICE_ARGS, s){
                                                                            handleMessage(Message msg) {
                                                                                handleServiceArgs((ServiceArgsData)msg.obj){
                                                                                    CreateServiceData createData = mServicesData.get(data.token);
                                                                                    Service s = mServices.get(data.token);
                                                                                    if (s != null) {
                                                                                        //Intent跨进程处理
                                                                                        if (data.args != null) {
                                                                                            data.args.setExtrasClassLoader(s.getClassLoader());
                                                                                            data.args.prepareToEnterProcess(isProtectedComponent(createData.info),
                                                                                                    s.getAttributionSource());
                                                                                        }
                                                                                        int res;
                                                                                        if (!data.taskRemoved) {
                                                                                            //调用service#onStartCommand
                                                                                            res = s.onStartCommand(data.args, data.flags, data.startId);
                                                                                        } else {
                                                                                            //用户从最近任务列表移除Task栈时调用
                                                                                            s.onTaskRemoved(data.args);
                                                                                            res = Service.START_TASK_REMOVED_COMPLETE;
                                                                                        }
                                                                                        
                                                                                        //确保其他异步任务执行完成
                                                                                        QueuedWork.waitToFinish();
                                                                            
                                                                                        try {
                                                                                            //Service相关任务执行完成
                                                                                            //这一步会根据onStartCommand的返回值，调整Service死亡重建策略
                                                                                            //同时会把之前的启动超时定时器取消
                                                                                            ActivityManager.getService().serviceDoneExecuting(
                                                                                                    data.token, SERVICE_DONE_EXECUTING_START, data.startId, res){
                                                                                                //======== SystemServer进程
                                                                                                    serviceDoneExecutingLocked(ServiceRecord r, int type, int startId, int res,
                                                                                                            boolean enqueueOomAdj) {
                                                                                                      //============== frameworks/base/services/core/java/com/android/server/am/ActiveServices.java  
                                                                                                        boolean inDestroying = mDestroyingServices.contains(r);
                                                                                                        if (r != null) {
                                                                                                            if (type == ActivityThread.SERVICE_DONE_EXECUTING_START) {
                                                                                                                // This is a call from a service start...  take care of
                                                                                                                // book-keeping.
                                                                                                                r.callStart = true;
                                                                                                                switch (res) {
                                                                                                                    case Service.START_STICKY_COMPATIBILITY:
                                                                                                                    case Service.START_STICKY: {
                                                                                                                        // We are done with the associated start arguments.
                                                                                                                        r.findDeliveredStart(startId, false, true);
                                                                                                                        // Don't stop if killed.
                                                                                                                        r.stopIfKilled = false;  // 服务被杀需要重新拉起
                                                                                                                        break;
                                                                                                                    }
                                                                                                        ... 
                                                                                                        
                                                                                                            if (psr.numberOfExecutingServices() == 0) {
                                                                                                                // 删除启动服务超时（超时会触发ANR）
                                                                                                                mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);
                                                                                                            }           
                                                                                                            
                                                                                            }
                                                                                        } catch (RemoteException e) {
                                                                                            throw e.rethrowFromSystemServer();
                                                                                        }
                                                                                    }
                                                                                }
                                                                            }
                                                                        }
                                                                }
                                                                ...
                                                            }
                                                    
                                                            //走到这里，需要确保此服务不再被视为延迟启动，同时将其从延迟启动服务列表中移除
                                                            if (r.delayed) {
                                                                getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
                                                                r.delayed = false;
                                                            }
                                                    
                                                            //Service被要求stop，停止服务
                                                            if (r.delayedStop) {
                                                                // Oh and hey we've already been asked to stop!
                                                                r.delayedStop = false;
                                                                if (r.startRequested) {
                                                                    stopServiceLocked(r, enqueueOomAdj);
                                                                }
                                                            }
                                                        }
    
                                                        return null;
                                                    } catch (TransactionTooLargeException e) {
                                                        throw e;
                                                    } catch (RemoteException e) {
                                                        Slog.w(TAG, "Exception when starting service " + r.shortInstanceName, e);
                                                    }
                                
                                                    // If a dead object exception was thrown -- fall through to
                                                    // restart the application.
                                                }
                                            }
                                        } else { //隔离进程
                                            // If this service runs in an isolated process, then each time
                                            // we call startProcessLocked() we will get a new isolated
                                            // process, starting another process if we are currently waiting
                                            // for a previous process to come up.  To deal with this, we store
                                            // in the service any current isolated process it is running in or
                                            // waiting to have come up.
                                            app = r.isolationHostProc;
                                            //辅助zygote进程，用于创建isolated_app进程来渲染不可信的web内容，具有最为严格的安全限制
                                            if (WebViewZygote.isMultiprocessEnabled()
                                                    && r.serviceInfo.packageName.equals(WebViewZygote.getPackageName())) {
                                                hostingRecord = HostingRecord.byWebviewZygote(r.instanceName, r.definingPackageName,
                                                        r.definingUid, r.serviceInfo.processName);
                                            }
                                            //应用zygote进程，与常规zygote创建的应用相比受到更多限制
                                            if ((r.serviceInfo.flags & ServiceInfo.FLAG_USE_APP_ZYGOTE) != 0) {
                                                hostingRecord = HostingRecord.byAppZygote(r.instanceName, r.definingPackageName,
                                                        r.definingUid, r.serviceInfo.processName);
                                            }
                                        }
                                
                                        //如果Service所在进程尚未启动
                                        if (app == null && !permissionsReviewRequired && !packageFrozen) {
                                            //启动App进程
                                            if (r.isSdkSandbox) { // 如果目标进程是sdk runtime沙盒进程
                                                final int uid = Process.toSdkSandboxUid(r.sdkSandboxClientAppUid);
                                                app = mAm.startSdkSandboxProcessLocked(procName, r.appInfo, true, intentFlags,
                                                        hostingRecord, ZYGOTE_POLICY_FLAG_EMPTY, uid, r.sdkSandboxClientAppPackage);
                                                r.isolationHostProc = app;
                                            } else { // 普通进程
                                                //启动service所在进程，进程启动后，会在pendingServices中拉起该服务
                                                app = mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                                                        hostingRecord, ZYGOTE_POLICY_FLAG_EMPTY, false, isolated);
                                            }
                                
                                            if (isolated) {  //如果是隔离进程，将这次启动的进程记录保存下来
                                                r.isolationHostProc = app;
                                            }
                                        }
                                
                                        if (r.fgRequired) {  //对于要启动的前台服务，加入到临时白名单，暂时绕过省电模式
                                            mAm.tempAllowlistUidLocked(r.appInfo.uid,
                                                    mAm.mConstants.mServiceStartForegroundTimeoutMs, REASON_SERVICE_LAUNCH,
                                                    "fg-service-launch",
                                                    TEMPORARY_ALLOW_LIST_TYPE_FOREGROUND_SERVICE_ALLOWED,
                                                    r.mRecentCallingUid);
                                        }
                                
                                        if (!mPendingServices.contains(r)) {
                                            //将启动的服务添加到mPendingServices列表中
                                            //如果服务进程尚未启动，进程在启动的过程中会检查此列表并启动需要启动的Service
                                            mPendingServices.add(r);
                                        }
                                
                                        //Service被要求stop，停止服务
                                        if (r.delayedStop) {     
                                            // Oh and hey we've already been asked to stop!
                                            r.delayedStop = false;
                                            if (r.startRequested) {
                                                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                                                        "Applying delayed stop (in bring up): " + r);
                                                stopServiceLocked(r, enqueueOomAdj);
                                            }
                                        }
                                
                                        return null;
                                    }
                                            
                                    /* Will be a no-op if nothing pending */
                                    mAm.updateOomAdjPendingTargetsLocked(OomAdjuster.OOM_ADJ_REASON_START_SERVICE);
                                    if (error != null) {
                                        return new ComponentName("!!", error);
                                    }
                                    ...
                            
                                    if (r.startRequested && addToStarting) { //对于后台启动服务的情况
                                        boolean first = smap.mStartingBackground.size() == 0; //是否为第一个后台启动的服务
                                        smap.mStartingBackground.add(r);     //添加到正在后台启动服务列表中
                                        //设置后台启动服务超时时间（默认15秒）
                                        r.startingBgTimeout = SystemClock.uptimeMillis() + mAm.mConstants.BG_START_TIMEOUT;
                                        ...
                                        //如果为第一个后台启动的服务，则代表后面暂时没有正在后台启动的服务了
                                        //此时将之前设置为延迟启动的服务调度出来后台启动
                                        if (first) {
                                            smap.rescheduleDelayedStartsLocked();
                                        }
                                    } else if (callerFg || r.fgRequired) { //对于调用方进程为前台或作为前台服务启动的情况
                                        //将此Service从正在后台启动服务列表和延迟启动服务列表中移除
                                        //如果正在后台启动服务列表中存在此服务的话，将之前设置为延迟启动的服务调度出来后台启动
                                        smap.ensureNotStartingBackgroundLocked(r);
                                    }
                            
                                    return r.name;
                                }
    
                                return cmp;
                            }
                            if (res.aliasComponent != null
                                    && !realResult.getPackageName().startsWith("!")
                                    && !realResult.getPackageName().startsWith("?")) {
                                return res.aliasComponent;
                            } else {
                                return realResult;
                            }
                        }
                    } finally {
                        Binder.restoreCallingIdentity(origId);
                    }
                    return res;
                }
            
            
        //通过AMS层返回的ComponentName.packageName来判断是否出错以及错误类型        
        if (cn != null) {
            if (cn.getPackageName().equals("!")) {
                throw new SecurityException(
                        "Not allowed to start service " + service
                        + " without permission " + cn.getClassName());
            } else if (cn.getPackageName().equals("!!")) 
            ...
        }
        ...
        
        return cn;
    }
}
    
````





1. Activity#startService
   *APP A进程*
    *frameworks/base/core/java/android/app/ContextImpl.java*

       ContextImpl#startService{

       return startServiceCommon(service, false, mUser){

2. 通过Binder远程调用ActivityManagerService#startService

        ComponentName cn = ActivityManager.getService().startService(
        mMainThread.getApplicationThread(), service,
        service.resolveTypeIfNeeded(getContentResolver()), requireForeground,
        getOpPackageName(), getAttributionTag(), user.getIdentifier()){

    *SystemServer#AMS*
   *frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java*

            res = mServices.startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid,
                    requireForeground, callingPackage, callingFeatureId, userId){
3. ActiveServices#startServiceLocked
   *frameworks/base/services/core/java/com/android/server/am/ActiveServices.java*

               final ComponentName realResult = startServiceInnerLocked(r, service, callingUid, callingPid, fgRequired, callerFg,
               allowBackgroundActivityStarts, backgroundActivityStartsToken){
                    ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting,
                            callingUid, wasStartRequested){

4. <span id="bringUpService"> bringUpServiceLocked </span>

                        String error = bringUpServiceLocked(r, service.getFlags(), callerFg,
                                false /* whileRestarting */,
                                false /* permissionsReviewRequired */,
                                false /* packageFrozen */,
                                true /* enqueueOomAdj */){

                            final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
                    
                            if (!isolated) {
5. 如果不需要以独立进程的形式启动

                                app = mAm.getProcessRecordLocked(procName, r.appInfo.uid);
                                if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                                            + " app=" + app);
                                if (app != null) {
                                    final IApplicationThread thread = app.getThread();
                                    final int pid = app.getPid();
                                    final UidRecord uidRecord = app.getUidRecord();
                                    if (thread != null) {
6. 且该Service宿主进程已存在

                                        app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode,
                                                mAm.mProcessStats);
                                        realStartServiceLocked(r, app, thread, pid, uidRecord, execInFg,
                                                enqueueOomAdj){
7. 通过Binder调App B的scheduleCreateService

                                            thread.scheduleCreateService(r, r.serviceInfo,
                                                    mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                                                    app.mState.getReportedProcState()){
   *App B进程*
    *frameworks/base/core/java/android/app/ActivityThread.java*

                                                sendMessage(H.CREATE_SERVICE, s);
                                                ...
                                                case CREATE_SERVICE:
                                                    handleCreateService((CreateServiceData)msg.obj){
8. 反射创建Service

                                                        service = packageInfo.getAppFactory()
                                                                .instantiateService(cl, data.info.name, data.intent);
9. attach设置一些service的成员（如mBase,mThread,mApplication）
                                            
                                                        service.attach(context, this, data.info.name, data.token, app,
                                                                ActivityManager.getService());
10. 回调Service#onCreate

                                                         service.onCreate();
                                                     }
                                                     break;
                                             }

     *SystemServer#AMS*

                                             sendServiceArgsLocked(r, execInFg, true){
                                                 r.app.getThread().scheduleServiceArgs(r, slice){
11. 通过Binder调用ApplicationThread#scheduleServiceArgs
    *App B*

                                                    sendMessage(H.SERVICE_ARGS, s);
                                                    ...
                                                    case SERVICE_ARGS:
                                                    handleServiceArgs((ServiceArgsData)msg.obj){
                                                        if (!data.taskRemoved) {
12. 如果任务未被用户清理掉（从最近任务列表删除）则回调Service#onStartCommand

                                                            res = s.onStartCommand(data.args, data.flags, data.startId);
                                                        } else {
13. 如果任务被清理则回调onTaskRemoved（TODO 如果service运行中用户划走任务会触发回调吗）

                                                            s.onTaskRemoved(data.args);
                                                            res = Service.START_TASK_REMOVED_COMPLETE;
                                                        }
14. 根据onStartCommand的返回值调整service的死亡重建策略

                                                        ActivityManager.getService().serviceDoneExecuting(
                                                                data.token, SERVICE_DONE_EXECUTING_START, data.startId, res);
                                                    }
                                                }
                                            }

                                        }
15. 返回，不走后面流程

                                         return null;

                                     }
                                 }
                             } else {
16. 如果该service需要以独立进程运行
                                 // TODO

                             }
        
17. 如果Service宿主进程不存在则先[启动进程](#start_process)

                             app = mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                                 hostingRecord, ZYGOTE_POLICY_FLAG_EMPTY, false, isolated);

                         }
                     }
                 }
             }
 
         }





## bindService流程

````

  //============ 调用方进程
    //============ frameworks/base/core/java/android/app/ContextImpl.java
        public boolean bindService(Intent service, int flags, Executor executor, ServiceConnection conn) {
            return bindServiceCommon(service, conn, flags, null, null, executor, getUser()){
                //获取LoadedApk$ServiceDispatcher$IServiceConnection
                //这个类是用来后续连接建立完成后发布连接，回调ServiceConnection各种方法的
                IServiceConnection sd;
                ...
                
                if (mPackageInfo != null) {
                    if (executor != null) {
                        sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), executor, flags);
                    } else {
                        sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
                    }
                } else {
                    throw new RuntimeException("Not supported in system context");
                }
                
                validateServiceIntent(service);
            
                IBinder token = getActivityToken();   //获取ActivityRecord的远程Binder对象
                // 跨进程准备
                service.prepareToLeaveProcess(this);
                //跨进程使用AMS绑定Service
                int res = ActivityManager.getService().bindServiceInstance(
                        mMainThread.getApplicationThread(), getActivityToken(), service,
                        service.resolveTypeIfNeeded(getContentResolver()),
                        sd, flags, instanceName, getOpPackageName(), user.getIdentifier()){
                  //============= SystemServer进程
                    //============= frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
                        enforceNotIsolatedCaller("bindService");    // 隔离进程（如android:isolatedProcess=true的service）不允许调用bindService
                        enforceAllowedToStartOrBindServiceIfSdkSandbox(service); // 如果caller是sdk沙盒进程（运行在SDK Runtime）确保其有权限绑定服务
                        // Refuse possible leaked file descriptors
                        if (service != null && service.hasFileDescriptors() == true) {
                            throw new IllegalArgumentException("File descriptors passed in Intent");
                        }
                        ...

                            return mServices.bindServiceLocked(caller, token, service, resolvedType, connection,
                                    flags, instanceName, isSdkSandboxService, sdkSandboxClientAppUid,
                                    sdkSandboxClientAppPackage, callingPackage, userId){
                              //============= frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
                                final int callingPid = Binder.getCallingPid();
                                final int callingUid = Binder.getCallingUid();
                                final ProcessRecord callerApp = mAm.getRecordForAppLOSP(caller);
                        
                                ActivityServiceConnectionsHolder<ConnectionRecord> activity = null;
                                if (token != null) {//token不为空表示是从Activity发起的，token实际为ActivityRecord的远程Binder对象
                                    activity = mAm.mAtmInternal.getServiceConnectionsHolder(token);  //获取Activity与Service的连接记录
                                    if (activity == null) {  //ServiceConnectionsHolder为null说明调用方Activity不在栈中，直接异常返回
                                        Slog.w(TAG, "Binding with unknown activity: " + token);
                                        return 0;
                                    }
                                }
                        
                                int clientLabel = 0;
                                PendingIntent clientIntent = null;
                                final boolean isCallerSystem = callerApp.info.uid == Process.SYSTEM_UID;
                                ...
                                
                                //像对待Activity一样对待该Service
                                //需要校验调用方应用是否具有MANAGE_ACTIVITY_STACKS权限
                                if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                                    mAm.enforceCallingPermission(android.Manifest.permission.MANAGE_ACTIVITY_TASKS,
                                            "BIND_TREAT_LIKE_ACTIVITY");
                                }
                                // 针对flags的校验
                                ...
                        
                                final boolean callerFg = callerApp.mState.getSetSchedGroup()
                                        != ProcessList.SCHED_GROUP_BACKGROUND;
                                final boolean isBindExternal = (flags & Context.BIND_EXTERNAL_SERVICE) != 0;
                                final boolean allowInstant = (flags & Context.BIND_ALLOW_INSTANT) != 0;
                        
                                ServiceLookupResult res = retrieveServiceLocked(service, instanceName,
                                        isSdkSandboxService, sdkSandboxClientAppUid, sdkSandboxClientAppPackage,
                                        resolvedType, callingPackage, callingPid, callingUid, userId, true, callerFg,
                                        isBindExternal, allowInstant);
                        
                                ServiceRecord s = res.record;
                        
                                // The package could be frozen (meaning it's doing surgery), defer the actual
                                // binding until the package is unfrozen.
                                boolean packageFrozen = deferServiceBringupIfFrozenLocked(s, service, callingPackage, null,
                                        callingUid, callingPid, false, callerFg, userId, false, null, true, connection);
                        
                                // If permissions need a review before any of the app components can run,
                                // we schedule binding to the service but do not start its process, then
                                // we launch a review activity to which is passed a callback to invoke
                                // when done to start the bound service's process to completing the binding.
                                //如果需要用户手动确认授权则先不启动service进程，而是先弹出授权确认框
                                boolean permissionsReviewRequired = !packageFrozen
                                        && !requestStartTargetPermissionsReviewIfNeededLocked(s, callingPackage, null,
                                                callingUid, service, callerFg, userId, true, connection){
                                    if (mAm.getPackageManagerInternal().isPermissionsReviewRequired(r.packageName, r.userId)) {
                            
                                        // Show a permission review UI only for starting/binding from a foreground app
                                        if (!callerFg) { //只有调用方进程在前台才可以显示授权弹窗
                                            return false;
                                        }
                            
                                        final Intent intent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
                                        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                                                | Intent.FLAG_ACTIVITY_MULTIPLE_TASK
                                                | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
                                        intent.putExtra(Intent.EXTRA_PACKAGE_NAME, r.packageName);
                            
                                        if (isBinding) {
                                            RemoteCallback callback = new RemoteCallback(
                                                    new RemoteCallback.OnResultListener() {
                                                        @Override
                                                        public void onResult(Bundle result) {
                                                            final long identity = Binder.clearCallingIdentity();
                                                            try {
                                                                if (!mPendingServices.contains(r)) {
                                                                    return;
                                                                }
                                                                // If there is still a pending record, then the service
                                                                // binding request is still valid, so hook them up. We
                                                                // proceed only if the caller cleared the review requirement
                                                                // otherwise we unbind because the user didn't approve.
                                                                if (!mAm.getPackageManagerInternal()
                                                                        .isPermissionsReviewRequired(r.packageName,
                                                                            r.userId)) {
                                                                    //拉起服务，如果服务未创建，则会创建服务并调用其onCreate方法
                                                                    //如果服务已创建则什么都不会做
                                                                    bringUpServiceLocked(r,
                                                                            service.getFlags(),
                                                                            callerFg,
                                                                            false /* whileRestarting */,
                                                                            false /* permissionsReviewRequired */,
                                                                            false /* packageFrozen */,
                                                                            true /* enqueueOomAdj */);
                                                                } else {
                                                                    //无相应权限则解绑Service
                                                                    unbindServiceLocked(connection);
                                                                }
                                                            } finally {
                                                                Binder.restoreCallingIdentity(identity);
                                                            }
                                                        }
                                                    });
                                            intent.putExtra(Intent.EXTRA_REMOTE_CALLBACK, callback);
                                        } else { // Starting a service
                                            IIntentSender target = mAm.mPendingIntentController.getIntentSender(
                                                    ActivityManager.INTENT_SENDER_SERVICE, callingPackage, callingFeatureId,
                                                    callingUid, userId, null, null, 0, new Intent[]{service},
                                                    new String[]{service.resolveType(mAm.mContext.getContentResolver())},
                                                    PendingIntent.FLAG_CANCEL_CURRENT | PendingIntent.FLAG_ONE_SHOT
                                                    | PendingIntent.FLAG_IMMUTABLE, null);
                                            intent.putExtra(Intent.EXTRA_INTENT, new IntentSender(target));
                                        }
                                        
                                        //弹出授权弹窗
                                        mAm.mHandler.post(new Runnable() {
                                            @Override
                                            public void run() {
                                                mAm.mContext.startActivityAsUser(intent, new UserHandle(userId));
                                            }
                                        });
                            
                                        return false;
                                    }
                        
                                    return  true;
                                }
                            
                        
                                final long origId = Binder.clearCallingIdentity();
                        
                                try {
                                    //取消之前的Service重启任务（如果有）
                                    if (unscheduleServiceRestartLocked(s, callerApp.info.uid, false)) {
                                        if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "BIND SERVICE WHILE RESTART PENDING: "
                                                + s);
                                    }
                        
                                    //建立调用方与服务方之间的关联
                                    final boolean wasStartRequested = s.startRequested;
                                    final boolean hadConnections = !s.getConnections().isEmpty();
                                    mAm.startAssociationLocked(callerApp.uid, callerApp.processName,
                                            callerApp.mState.getCurProcState(), s.appInfo.uid, s.appInfo.longVersionCode,
                                            s.instanceName, s.processName);
                                    // Once the apps have become associated, if one of them is caller is ephemeral
                                    // the target app should now be able to see the calling app
                                    mAm.grantImplicitAccess(callerApp.userId, service,
                                            callerApp.uid, UserHandle.getAppId(s.appInfo.uid));
                        
                                    //查询App绑定信息
                                    AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
                                    //创建连接信息
                                    ConnectionRecord c = new ConnectionRecord(b, activity,
                                            connection, flags, clientLabel, clientIntent,
                                            callerApp.uid, callerApp.processName, callingPackage, res.aliasComponent);
                        
                                    IBinder binder = connection.asBinder();
                                    s.addConnection(binder, c);
                                    b.connections.add(c);
                                    if (activity != null) {
                                        activity.addConnection(c);
                                    }
                                    final ProcessServiceRecord clientPsr = b.client.mServices;
                                    clientPsr.addConnection(c);
                                    //建立关联
                                    c.startAssociationIfNeeded();
                                    if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
                                        clientPsr.setHasAboveClient(true);
                                    }
                                    if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {
                                        s.allowlistManager = true;
                                    }
                                    if ((flags & Context.BIND_ALLOW_BACKGROUND_ACTIVITY_STARTS) != 0) {
                                        s.setAllowedBgActivityStartsByBinding(true); //允许Service后台启动Activity
                                    }
                        
                                    if ((flags & Context.BIND_NOT_APP_COMPONENT_USAGE) != 0) {
                                        s.isNotAppComponentUsage = true;
                                    }
                        
                                    if (s.app != null && s.app.mState != null
                                            && s.app.mState.getCurProcState() <= PROCESS_STATE_TOP
                                            && (flags & Context.BIND_ALMOST_PERCEPTIBLE) != 0) {
                                        s.lastTopAlmostPerceptibleBindRequestUptimeMs = SystemClock.uptimeMillis();
                                    }
                        
                                    if (s.app != null) {        //更新是否有与Service建立连接的Activity
                                        updateServiceClientActivitiesLocked(s.app.mServices, c, true);
                                    }
                                    //更新连接列表
                                    ArrayList<ConnectionRecord> clist = mServiceConnections.get(binder);
                                    if (clist == null) {
                                        clist = new ArrayList<>();
                                        mServiceConnections.put(binder, clist);
                                    }
                                    clist.add(c);
                        
                                    boolean needOomAdj = false;
                                    if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                                        s.lastActivity = SystemClock.uptimeMillis();
                                        needOomAdj = true;
                                        //拉起服务，如果服务未创建，则会创建服务并调用其onCreate方法
                                        //如果服务已创建则什么都不会做
                                        if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                                                permissionsReviewRequired, packageFrozen, true) != null) {
                                            mAm.updateOomAdjPendingTargetsLocked(OomAdjuster.OOM_ADJ_REASON_BIND_SERVICE);
                                            return 0;
                                        }
                                    }
                                    // 更新前台服务限制策略
                                    setFgsRestrictionLocked(callingPackage, callingPid, callingUid, service, s, userId,
                                            false);
                        
                                    if (s.app != null) {
                                        ProcessServiceRecord servicePsr = s.app.mServices;
                                        if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                                            servicePsr.setTreatLikeActivity(true);
                                        }
                                        if (s.allowlistManager) {
                                            servicePsr.mAllowlistManager = true;
                                        }
                                        // This could have made the service more important.
                                        mAm.updateLruProcessLocked(s.app, (callerApp.hasActivitiesOrRecentTasks()
                                                    && servicePsr.hasClientActivities())
                                                || (callerApp.mState.getCurProcState() <= PROCESS_STATE_TOP
                                                    && (flags & Context.BIND_TREAT_LIKE_ACTIVITY) != 0),
                                                b.client);
                                        needOomAdj = true;
                                        mAm.enqueueOomAdjTargetLocked(s.app);
                                    }
                                    if (needOomAdj) {
                                        mAm.updateOomAdjPendingTargetsLocked(OomAdjuster.OOM_ADJ_REASON_BIND_SERVICE);
                                    }
                        
                                    FrameworkStatsLog.write(SERVICE_REQUEST_EVENT_REPORTED, s.appInfo.uid, callingUid,
                                            ActivityManagerService.getShortAction(service.getAction()),
                                            SERVICE_REQUEST_EVENT_REPORTED__REQUEST_TYPE__BIND, false,
                                            s.app == null || s.app.getThread() == null
                                            ? SERVICE_REQUEST_EVENT_REPORTED__PROC_START_TYPE__PROCESS_START_TYPE_COLD
                                            : (wasStartRequested || hadConnections
                                            ? SERVICE_REQUEST_EVENT_REPORTED__PROC_START_TYPE__PROCESS_START_TYPE_HOT
                                            : SERVICE_REQUEST_EVENT_REPORTED__PROC_START_TYPE__PROCESS_START_TYPE_WARM));
                        
                                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bind " + s + " with " + b
                                            + ": received=" + b.intent.received
                                            + " apps=" + b.intent.apps.size()
                                            + " doRebind=" + b.intent.doRebind);
                        
                                    if (s.app != null && b.intent.received) {
                                        // Service is already running, so we can immediately
                                        // publish the connection.
                        
                                        // If what the client try to start/connect was an alias, then we need to
                                        // pass the alias component name instead to the client.
                                        final ComponentName clientSideComponentName =
                                                res.aliasComponent != null ? res.aliasComponent : s.name;
                                        try {
                                            //如果服务之前就已经在运行，即Service.onBind方法已经被执行，返回的IBinder对象也已经被保存
                                            //调用LoadedApk$ServiceDispatcher$InnerConnection.connected方法
                                            //回调ServiceConnection.onServiceConnected方法
                                            c.conn.connected(clientSideComponentName, b.intent.binder, false){
                                              //============= frameworks/base/core/java/android/app/LoadedApk.java 
                                                LoadedApk#ServiceDispatcher#InnerConnection#connected(ComponentName name, IBinder service, boolean dead){
                                                    LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                                                    if (sd != null) {
                                                        sd.connected(name, service, dead){
                                                            mActivityThread.post(new RunConnection(name, service, 0, dead){
                                                                public void run() {
                                                                    if (mCommand == 0) {
                                                                        doConnected(mName, mService, mDead){
                                                                            ServiceDispatcher.ConnectionInfo old;
                                                                            ServiceDispatcher.ConnectionInfo info;
                                                                
                                                                            // A new service is being connected... set it all up.
                                                                            info = new ConnectionInfo();
                                                                            info.binder = service;
                                                                            info.deathMonitor = new DeathMonitor(name, service);
                                                                            try {
                                                                                service.linkToDeath(info.deathMonitor, 0);
                                                                                mActiveConnections.put(name, info);
                                                                            } catch (RemoteException e) {
                                                                                // This service was dead before we got it...  just
                                                                                // don't do anything with it.
                                                                                mActiveConnections.remove(name);
                                                                                return;
                                                                            }
                                                                
                                                                            if (old != null) {
                                                                                old.binder.unlinkToDeath(old.deathMonitor, 0);
                                                                            }
                                                                
                                                                            // If there was an old service, it is now disconnected.
                                                                            if (old != null) {
                                                                                mConnection.onServiceDisconnected(name);
                                                                            }
                                                                            if (dead) {
                                                                                mConnection.onBindingDied(name);
                                                                            } else {
                                                                                // If there is a new viable service, it is now connected.
                                                                                if (service != null) {
                                                                                    mConnection.onServiceConnected(name, service); // 回调ServiceConnection#onServiceConnected()
                                                                                } else {
                                                                                    // The binding machinery worked, but the remote returned null from onBind().
                                                                                    mConnection.onNullBinding(name);
                                                                                }
                                                                            }
                                                                        }
                                                                    } else if (mCommand == 1) {
                                                                        doDeath(mName, mService);
                                                                    }
                                                                }
                                                            });
                                                        }
                                                    }
                                                }
                                            
                                            }
                                        } catch (Exception e) {
                                            Slog.w(TAG, "Failure sending service " + s.shortInstanceName
                                                    + " to connection " + c.conn.asBinder()
                                                    + " (in " + c.binding.client.processName + ")", e);
                                        }
                        
                                        // If this is the first app connected back to this binding,
                                        // and the service had previously asked to be told when
                                        // rebound, then do so.
                                        if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                                            requestServiceBindingLocked(s, b.intent, callerFg, true);
                                        }
                                    } else if (!b.intent.requested) {             
                                        //当服务解绑，调用到Service.onUnbind方法时返回true，此时doRebind变量就会被赋值为true
                                        //此时，当再次建立连接时，服务会回调Service.onRebind方法
                                        requestServiceBindingLocked(s, b.intent, callerFg, false){
                                            if (r.app == null || r.app.getThread() == null) {
                                                // If service is not currently running, can't yet bind.
                                                return false;
                                            }
                                            if ((!i.requested || rebind) && i.apps.size() > 0) {
                                                bumpServiceExecutingLocked(r, execInFg, "bind",
                                                        OomAdjuster.OOM_ADJ_REASON_BIND_SERVICE);
                                                        
                                                r.app.getThread().scheduleBindService(r, i.intent.getIntent(), rebind,
                                                        r.app.mState.getReportedProcState()){
                                                  //=============== service所在进程
                                                    //=============== frameworks/base/core/java/android/app/ActivityThread.java
                                                        ...
                                                        sendMessage(H.BIND_SERVICE, s){
                                                            handleMessage(Message msg) {
                                                                ...
                                                                handleBindService((BindServiceData)msg.obj){
                                                                    CreateServiceData createData = mServicesData.get(data.token);
                                                                    Service s = mServices.get(data.token);
                                                                    if (s != null) {
                                                                        data.intent.setExtrasClassLoader(s.getClassLoader());
                                                                        data.intent.prepareToEnterProcess(isProtectedComponent(createData.info),
                                                                                s.getAttributionSource());
                                                                        try {
                                                                            if (!data.rebind) {
                                                                                //正常情况下回调Service.onBind方法，获得控制Service的IBinder对象
                                                                                // 回调Service#onBind()
                                                                                // 从SystemServer进程RPC调用scheduleBindService切到Service所在进程,
                                                                                // 没做多少事又立马RPC调用publishService回到SystemServer进程好像没啥必要，
                                                                                // 主要是为了在Service所在进程回调onBind（不能从SystemServer进程直接回调onBind）
                                                                                IBinder binder = s.onBind(data.intent);
                                                                                ActivityManager.getService().publishService(data.token, data.intent, binder){
                                                                                  //=========== SystemServer进程
                                                                                    //============== frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
                                                                                        mServices.publishServiceLocked((ServiceRecord)token, intent, service){
                                                                                            final long origId = Binder.clearCallingIdentity();
                                                                                            try {
                                                                                                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "PUBLISHING " + r
                                                                                                        + " " + intent + ": " + service);
                                                                                                if (r != null) {
                                                                                                    Intent.FilterComparison filter
                                                                                                            = new Intent.FilterComparison(intent);
                                                                                                    IntentBindRecord b = r.bindings.get(filter);
                                                                                                    if (b != null && !b.received) {
                                                                                                        b.binder = service;
                                                                                                        b.requested = true;
                                                                                                        b.received = true;
                                                                                                        ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections = r.getConnections();
                                                                                                        //遍历所有与此服务绑定的客户端连接
                                                                                                        for (int conni = connections.size() - 1; conni >= 0; conni--) {
                                                                                                            ArrayList<ConnectionRecord> clist = connections.valueAt(conni);
                                                                                                            for (int i=0; i<clist.size(); i++) {
                                                                                                                ConnectionRecord c = clist.get(i);
                                                                                                                if (!filter.equals(c.binding.intent.intent)) {
                                                                                                                    if (DEBUG_SERVICE) Slog.v(
                                                                                                                            TAG_SERVICE, "Not publishing to: " + c);
                                                                                                                    if (DEBUG_SERVICE) Slog.v(
                                                                                                                            TAG_SERVICE, "Bound intent: " + c.binding.intent.intent);
                                                                                                                    if (DEBUG_SERVICE) Slog.v(
                                                                                                                            TAG_SERVICE, "Published intent: " + intent);
                                                                                                                    continue;
                                                                                                                }
                                                                                                                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Publishing to: " + c);
                                                                                                                // If what the client try to start/connect was an alias, then we need to
                                                                                                                // pass the alias component name instead to the client.
                                                                                                                final ComponentName clientSideComponentName =
                                                                                                                        c.aliasComponent != null ? c.aliasComponent : r.name;
                                                                                                                //调用LoadedApk$ServiceDispatcher$IServiceConnection.connected方法
                                                                                                                //回调ServiceConnection.onServiceConnected方法
                                                                                                                c.conn.connected(clientSideComponentName, service, false);
                                                                                                            }
                                                                                                        }
                                                                                                    }
                                                                                    
                                                                                                    serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false, false);
                                                                                                }
                                                                                            } finally {
                                                                                                Binder.restoreCallingIdentity(origId);
                                                                                            }
                                                                                        }
                                                                                }
                                                                            } else {                   
                                                                                //当服务解绑，调用到Service.onUnbind方法时返回true，此时doRebind变量就会被赋值为true
                                                                                //此时，当再次建立连接时，服务会回调Service.onRebind方法
                                                                                // 回调Service#onRebind()
                                                                                s.onRebind(data.intent);
                                                                                ActivityManager.getService().serviceDoneExecuting(
                                                                                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                                                                            }
                                                                        } catch (RemoteException ex) {
                                                                            throw ex.rethrowFromSystemServer();
                                                                        }
                                                                    }
                                                                }
                                                            }
                                                        }
                                                }
                                                if (!rebind) {
                                                    i.requested = true;
                                                }
                                                i.hasBound = true;
                                                i.doRebind = false;
                                            }
                                            return true;
                                        }
                                    }
                        
                                    maybeLogBindCrossProfileService(userId, callingPackage, callerApp.info.uid);
                                    //将此Service从正在后台启动服务列表和延迟启动服务列表中移除
                                    //如果正在后台启动服务列表中存在此服务的话，将之前设置为延迟启动的服务调度出来后台启动
                                    getServiceMapLocked(s.userId).ensureNotStartingBackgroundLocked(s);
                        
                                } finally {
                                    Binder.restoreCallingIdentity(origId);
                                }
                        
                                notifyBindingServiceEventLocked(callerApp, callingPackage);
                        
                                return 1;
                            }
    
                }

                if (res < 0) {
                    throw new SecurityException(
                            "Not allowed to bind to service " + service);
                }
                return res != 0;
            }
        }
        
````



## stopService流程

````

  //============ 调用方进程
    //============ frameworks/base/core/java/android/app/ContextImpl.java
        public boolean stopService(Intent service) {
            return stopServiceCommon(service, mUser){
                validateServiceIntent(service);
                service.prepareToLeaveProcess(this);
                int res = ActivityManager.getService().stopService(
                    mMainThread.getApplicationThread(), service,
                    service.resolveTypeIfNeeded(getContentResolver()), user.getIdentifier()){
                    //============= SystemServer进程
                      //============= frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
                        enforceNotIsolatedCaller("stopService");  // 隔离进程禁止调用stopService
                        // Refuse possible leaked file descriptors
                        if (service != null && service.hasFileDescriptors() == true) {
                            throw new IllegalArgumentException("File descriptors passed in Intent");
                        }
                
                        return mServices.stopServiceLocked(caller, service, resolvedType, userId){
                            final ProcessRecord callerApp = mAm.getRecordForAppLOSP(caller);
                            // If this service is active, make sure it is stopped.
                            ServiceLookupResult r = retrieveServiceLocked(service, null, resolvedType, null,
                                    Binder.getCallingPid(), Binder.getCallingUid(), userId, false, false, false, false);
                            final long origId = Binder.clearCallingIdentity();
                            try {
                                stopServiceLocked(r.record, false){
                                    if (service.delayed) {
                                        // If service isn't actually running, but is being held in the
                                        // delayed list, then we need to keep it started but note that it
                                        // should be stopped once no longer delayed.
                                        if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "Delaying stop of pending: " + service);
                                        service.delayedStop = true;
                                        return;
                                    }
                            
                                    final int uid = service.appInfo.uid;
                                    final String packageName = service.name.getPackageName();
                                    final String serviceName = service.name.getClassName();
                                    mAm.mBatteryStatsService.noteServiceStopRunning(uid, packageName, serviceName);
                                    service.startRequested = false; //启动标志设置为false
                                    service.callStart = false;
                            
                                    bringDownServiceIfNeededLocked(service, false, false, enqueueOomAdj){
                                        //检查此服务是否还被需要
                                        //在用stopSelf停止服务的这种情况下
                                        //检查的就是是否有auto-create的连接（flag为BIND_AUTO_CREATE）
                                        //如有则不能停止服务
                                        if (isServiceNeededLocked(r, knowConn, hasConn){
                                            // Are we still explicitly being asked to run?
                                            if (r.startRequested) {
                                                return true;
                                            }
                                    
                                            // Is someone still bound to us keeping us running?
                                            if (!knowConn) {
                                                hasConn = r.hasAutoCreateConnections();
                                            }
                                            if (hasConn) {
                                                return true;
                                            }
                                    
                                            return false;
                                        }) {
                                            return;
                                        }
                                
                                        // Are we in the process of launching?
                                        // 不要停止正在启动中的Service
                                        if (mPendingServices.contains(r)) {
                                            return;
                                        }
                                
                                        bringDownServiceLocked(r, enqueueOomAdj){
                                        
                                            // Report to all of the connections that the service is no longer
                                            // available.
                                            ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections = r.getConnections();
                                            for (int conni = connections.size() - 1; conni >= 0; conni--) {
                                                ArrayList<ConnectionRecord> c = connections.valueAt(conni);
                                                for (int i=0; i<c.size(); i++) {
                                                    ConnectionRecord cr = c.get(i);
                                                    // There is still a connection to the service that is
                                                    // being brought down.  Mark it as dead.
                                                    cr.serviceDead = true;
                                                    cr.stopAssociation();
                                                    final ComponentName clientSideComponentName =
                                                            cr.aliasComponent != null ? cr.aliasComponent : r.name;
                                                    try {
                                                        cr.conn.connected(r.name, null, true);
                                                    } catch (Exception e) {
                                                        Slog.w(TAG, "Failure disconnecting service " + r.shortInstanceName
                                                              + " to connection " + c.get(i).conn.asBinder()
                                                              + " (in " + c.get(i).binding.client.processName + ")", e);
                                                    }
                                                }
                                            }
                                    
                                            boolean oomAdjusted = false;
                                            // Tell the service that it has been unbound.
                                            if (r.app != null && r.app.getThread() != null) {
                                                for (int i = r.bindings.size() - 1; i >= 0; i--) {
                                                    IntentBindRecord ibr = r.bindings.valueAt(i);
                                                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Bringing down binding " + ibr
                                                            + ": hasBound=" + ibr.hasBound);
                                                    if (ibr.hasBound) {
                                                        try {
                                                            oomAdjusted |= bumpServiceExecutingLocked(r, false, "bring down unbind",
                                                                    OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
                                                            ibr.hasBound = false;
                                                            ibr.requested = false;
                                                            r.app.getThread().scheduleUnbindService(r,
                                                                    ibr.intent.getIntent());
                                                        } catch (Exception e) {
                                                            Slog.w(TAG, "Exception when unbinding service "
                                                                    + r.shortInstanceName, e);
                                                            serviceProcessGoneLocked(r, enqueueOomAdj);
                                                            break;
                                                        }
                                                    }
                                                }
                                            }
                                    
                                            // Check to see if the service had been started as foreground, but being
                                            // brought down before actually showing a notification.  That is not allowed.
                                            //如果此Service是以前台服务的形式启动，并且当前还尚未成为前台服务，这种情况不被允许，直接令App崩溃，杀死应用
                                            if (r.fgRequired) {
                                                Slog.w(TAG_SERVICE, "Bringing down service while still waiting for start foreground: "
                                                        + r);
                                                r.fgRequired = false;
                                                r.fgWaiting = false;
                                                synchronized (mAm.mProcessStats.mLock) {
                                                    ServiceState stracker = r.getTracker();
                                                    if (stracker != null) {
                                                        stracker.setForeground(false, mAm.mProcessStats.getMemFactorLocked(),
                                                                SystemClock.uptimeMillis());
                                                    }
                                                }
                                                mAm.mAppOpsService.finishOperation(AppOpsManager.getToken(mAm.mAppOpsService),
                                                        AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName, null);
                                                mAm.mHandler.removeMessages(
                                                        ActivityManagerService.SERVICE_FOREGROUND_TIMEOUT_MSG, r);
                                                if (r.app != null) {
                                                    // 直接令App崩溃，杀死应用
                                                    Message msg = mAm.mHandler.obtainMessage(
                                                            ActivityManagerService.SERVICE_FOREGROUND_CRASH_MSG);
                                                    SomeArgs args = SomeArgs.obtain();
                                                    args.arg1 = r.app;
                                                    args.arg2 = r.toString();
                                                    args.arg3 = r.getComponentName();
                                    
                                                    msg.obj = args;
                                                    mAm.mHandler.sendMessage(msg);
                                                }
                                            }
                                            
                                            r.destroyTime = SystemClock.uptimeMillis();
                                            if (LOG_SERVICE_START_STOP) {
                                                EventLogTags.writeAmDestroyService(
                                                        r.userId, System.identityHashCode(r), (r.app != null) ? r.app.getPid() : -1);
                                            }
                                    
                                            final ServiceMap smap = getServiceMapLocked(r.userId);
                                            ServiceRecord found = smap.mServicesByInstanceName.remove(r.instanceName);
                                    
                                            // Note when this method is called by bringUpServiceLocked(), the service is not found
                                            // in mServicesByInstanceName and found will be null.
                                            if (found != null && found != r) {
                                                // This is not actually the service we think is running...  this should not happen,
                                                // but if it does, fail hard.
                                                smap.mServicesByInstanceName.put(r.instanceName, found);
                                                throw new IllegalStateException("Bringing down " + r + " but actually running "
                                                        + found);
                                            }
                                            smap.mServicesByIntent.remove(r.intent);
                                            r.totalRestartCount = 0;
                                            unscheduleServiceRestartLocked(r, 0, true);
                                    
                                            // Also make sure it is not on the pending list.
                                            for (int i=mPendingServices.size()-1; i>=0; i--) {
                                                if (mPendingServices.get(i) == r) {
                                                    mPendingServices.remove(i);
                                                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Removed pending: " + r);
                                                }
                                            }
                                            if (mPendingBringups.remove(r) != null) {
                                                if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Removed pending bringup: " + r);
                                            }
                                    
                                            //关闭前台服务通知
                                            cancelForegroundNotificationLocked(r);
                                            
                                            final boolean exitingFg = r.isForeground;
                                            if (exitingFg) {
                                                decActiveForegroundAppLocked(smap, r);
                                                synchronized (mAm.mProcessStats.mLock) {
                                                    ServiceState stracker = r.getTracker();
                                                    if (stracker != null) {
                                                        stracker.setForeground(false, mAm.mProcessStats.getMemFactorLocked(),
                                                                SystemClock.uptimeMillis());
                                                    }
                                                }
                                                mAm.mAppOpsService.finishOperation(
                                                        AppOpsManager.getToken(mAm.mAppOpsService),
                                                        AppOpsManager.OP_START_FOREGROUND, r.appInfo.uid, r.packageName, null);
                                                unregisterAppOpCallbackLocked(r);
                                                r.mFgsExitTime = SystemClock.uptimeMillis();
                                                logFGSStateChangeLocked(r,
                                                        FrameworkStatsLog.FOREGROUND_SERVICE_STATE_CHANGED__STATE__EXIT,
                                                        r.mFgsExitTime > r.mFgsEnterTime
                                                                ? (int) (r.mFgsExitTime - r.mFgsEnterTime) : 0,
                                                        FGS_STOP_REASON_STOP_SERVICE);
                                                mAm.updateForegroundServiceUsageStats(r.name, r.userId, false);
                                            }
                                    
                                            r.isForeground = false;
                                            r.mFgsNotificationWasDeferred = false;
                                            dropFgsNotificationStateLocked(r);
                                            r.foregroundId = 0;
                                            r.foregroundNoti = null;
                                            resetFgsRestrictionLocked(r);
                                            // Signal FGS observers *after* changing the isForeground state, and
                                            // only if this was an actual state change.
                                            if (exitingFg) {
                                                signalForegroundServiceObserversLocked(r);
                                            }
                                    
                                            // Clear start entries.
                                            r.clearDeliveredStartsLocked();
                                            r.pendingStarts.clear();
                                            smap.mDelayedStartList.remove(r);
                                    
                                            if (r.app != null) {
                                                mAm.mBatteryStatsService.noteServiceStopLaunch(r.appInfo.uid, r.name.getPackageName(),
                                                        r.name.getClassName());
                                                stopServiceAndUpdateAllowlistManagerLocked(r);
                                                if (r.app.getThread() != null) {
                                                    // Bump the process to the top of LRU list
                                                    mAm.updateLruProcessLocked(r.app, false, null);
                                                    updateServiceForegroundLocked(r.app.mServices, false);
                                                    try {
                                                        oomAdjusted |= bumpServiceExecutingLocked(r, false, "destroy",
                                                                oomAdjusted ? null : OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
                                                        mDestroyingServices.add(r);
                                                        r.destroying = true;
                                                        r.app.getThread().scheduleStopService(r);
                                                    } catch (Exception e) {
                                                        Slog.w(TAG, "Exception when destroying service "
                                                                + r.shortInstanceName, e);
                                                        serviceProcessGoneLocked(r, enqueueOomAdj);
                                                    }
                                                } else {
                                                    if (DEBUG_SERVICE) Slog.v(
                                                        TAG_SERVICE, "Removed service that has no process: " + r);
                                                }
                                            } else {
                                                if (DEBUG_SERVICE) Slog.v(
                                                    TAG_SERVICE, "Removed service that is not running: " + r);
                                            }
                                    
                                            if (!oomAdjusted) {
                                                mAm.enqueueOomAdjTargetLocked(r.app);
                                                if (!enqueueOomAdj) {
                                                    mAm.updateOomAdjPendingTargetsLocked(OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
                                                }
                                            }
                                            if (r.bindings.size() > 0) {
                                                r.bindings.clear();
                                            }
                                    
                                            if (r.restarter instanceof ServiceRestarter) { //对于主动停止的Service，不需要重启
                                                //将ServiceRestarter内的ServiceRecord变量设为null，避免其后续重启
                                               ((ServiceRestarter)r.restarter).setService(null);
                                            }
                                    
                                            synchronized (mAm.mProcessStats.mLock) {
                                                final int memFactor = mAm.mProcessStats.getMemFactorLocked();
                                                if (r.tracker != null) {
                                                    final long now = SystemClock.uptimeMillis();
                                                    r.tracker.setStarted(false, memFactor, now);
                                                    r.tracker.setBound(false, memFactor, now);
                                                    if (r.executeNesting == 0) {
                                                        r.tracker.clearCurrentOwner(r, false);
                                                        r.tracker = null;
                                                    }
                                                }
                                            }
                                    
                                            smap.ensureNotStartingBackgroundLocked(r);
                                            updateNumForegroundServicesLocked();
                                        }
    
                                    }
                                }
                            } finally {
                                Binder.restoreCallingIdentity(origId);
                            }
                    
                            return 0;
                        }
                }
                    
                if (res < 0) {
                    throw new SecurityException(
                            "Not allowed to stop service " + service);
                }
                return res != 0;
            }
        }

````


## unbindService流程

````
  //============ 调用方进程
    //============ frameworks/base/core/java/android/app/ContextImpl.java
        public void unbindService(ServiceConnection conn) {
            //将ServiceDispatcher的状态设置为forgotten，之后便不再会回调ServiceConnection任何方法
            IServiceConnection sd = mPackageInfo.forgetServiceDispatcher(
                    getOuterContext(), conn);
            // 远程调用AMS#unbindService        
            ActivityManager.getService().unbindService(sd){
              // SystemServer进程
                return mServices.unbindServiceLocked(connection){
                  //============= frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
                    IBinder binder = connection.asBinder();
                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "unbindService: conn=" + binder);
                    ArrayList<ConnectionRecord> clist = mServiceConnections.get(binder);
                    if (clist == null) {
                        Slog.w(TAG, "Unbind failed: could not find connection for "
                              + connection.asBinder());
                        return false;
                    }
            
                    final long origId = Binder.clearCallingIdentity();
                    try {
                        while (clist.size() > 0) {
                            ConnectionRecord r = clist.get(0);
                            removeConnectionLocked(r, null, null, true){
                            
                                IBinder binder = c.conn.asBinder();
                                AppBindRecord b = c.binding;
                                ServiceRecord s = b.service;
                                //这里的clist是ServiceRecord中的列表，和上一个方法中的clist不是一个对象
                                ArrayList<ConnectionRecord> clist = s.getConnections().get(binder);
                                if (clist != null) {
                                    clist.remove(c);
                                    if (clist.size() == 0) {
                                        s.removeConnection(binder);
                                    }
                                }
                                b.connections.remove(c);
                                c.stopAssociation();
                                if (c.activity != null && c.activity != skipAct) {
                                    c.activity.removeConnection(c);
                                }
                                if (b.client != skipApp) {
                                    final ProcessServiceRecord psr = b.client.mServices;
                                    psr.removeConnection(c);
                                    if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
                                        psr.updateHasAboveClientLocked();
                                    }
                                    // If this connection requested allowlist management, see if we should
                                    // now clear that state.
                                    if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {
                                        s.updateAllowlistManager();
                                        if (!s.allowlistManager && s.app != null) {
                                            updateAllowlistManagerLocked(s.app.mServices);
                                        }
                                    }
                                    // And do the same for bg activity starts ability.
                                    if ((c.flags & Context.BIND_ALLOW_BACKGROUND_ACTIVITY_STARTS) != 0) {
                                        s.updateIsAllowedBgActivityStartsByBinding();
                                    }
                                    // And for almost perceptible exceptions.
                                    if ((c.flags & Context.BIND_ALMOST_PERCEPTIBLE) != 0) {
                                        psr.updateHasTopStartedAlmostPerceptibleServices();
                                    }
                                    if (s.app != null) {
                                        updateServiceClientActivitiesLocked(s.app.mServices, c, true);
                                    }
                                }
                                
                                //将连接从mServiceConnections列表中移除
                                //这个clist才是和上一个方法是同一个对象
                                clist = mServiceConnections.get(binder);
                                if (clist != null) {
                                    clist.remove(c);
                                    if (clist.size() == 0) {
                                        mServiceConnections.remove(binder);
                                    }
                                }
                        
                                mAm.stopAssociationLocked(b.client.uid, b.client.processName, s.appInfo.uid,
                                        s.appInfo.longVersionCode, s.instanceName, s.processName);
                        
                        
                                //如果调用方App没有其他连接和Service绑定
                                //则将整个AppBindRecord移除
                                if (b.connections.size() == 0) {
                                    b.intent.apps.remove(b.client);
                                }
                        
                                if (!c.serviceDead) {
                                    //如果服务端进程存活并且没有其他连接绑定了，同时服务还处在绑定关系中（尚未回调过Service.onUnbind）
                                    if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Disconnecting binding " + b.intent
                                            + ": shouldUnbind=" + b.intent.hasBound);
                                    if (s.app != null && s.app.getThread() != null && b.intent.apps.size() == 0
                                            && b.intent.hasBound) {
                                        try {
                                            bumpServiceExecutingLocked(s, false, "unbind",
                                                    OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
                                            if (b.client != s.app && (c.flags&Context.BIND_WAIVE_PRIORITY) == 0
                                                    && s.app.mState.getSetProcState() <= PROCESS_STATE_HEAVY_WEIGHT) {
                                                // If this service's process is not already in the cached list,
                                                // then update it in the LRU list here because this may be causing
                                                // it to go down there and we want it to start out near the top.
                                                mAm.updateLruProcessLocked(s.app, false, null);
                                            }
                                            //标记为未绑定
                                            b.intent.hasBound = false;
                                            // Assume the client doesn't want to know about a rebind;
                                            // we will deal with that later if it asks for one.
                                            b.intent.doRebind = false;
                                            //回到App进程，调度执行Service的unbind操作
                                            s.app.getThread().scheduleUnbindService(s, b.intent.intent.getIntent()){
                                                ...
                                                sendMessage(H.UNBIND_SERVICE, s){
                                                    handleUnbindService((BindServiceData)msg.obj){
                                                        CreateServiceData createData = mServicesData.get(data.token);
                                                        Service s = mServices.get(data.token);
                                                        if (s != null) {
                                                            data.intent.setExtrasClassLoader(s.getClassLoader());
                                                            data.intent.prepareToEnterProcess(isProtectedComponent(createData.info),
                                                                    s.getAttributionSource());
                                                            //回调Service.onUnbind方法，如果返回值为true
                                                            //当再次建立连接时，服务会回调Service.onRebind方法
                                                            boolean doRebind = s.onUnbind(data.intent);
                                                            if (doRebind) { // 根据onUnbind的返回值设置不同的策略
                                                                ActivityManager.getService().unbindFinished(
                                                                        data.token, data.intent, doRebind);
                                                            } else {
                                                                ActivityManager.getService().serviceDoneExecuting(
                                                                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                                                            }
                                                        }
                                                    }
                                                    schedulePurgeIdler();
                                                }
                                            }
                                            
                                        } catch (Exception e) {
                                            Slog.w(TAG, "Exception when unbinding service " + s.shortInstanceName, e);
                                            serviceProcessGoneLocked(s, enqueueOomAdj);
                                        }
                                    }
                        
                                    // If unbound while waiting to start and there is no connection left in this service,
                                    // remove the pending service
                                    if (s.getConnections().isEmpty()) {
                                        mPendingServices.remove(s);
                                        mPendingBringups.remove(s);
                                    }
                        
                                    if ((c.flags&Context.BIND_AUTO_CREATE) != 0) {
                                        //是否有其他含有BIND_AUTO_CREATE标记的连接
                                        boolean hasAutoCreate = s.hasAutoCreateConnections();
                                        if (!hasAutoCreate) {
                                            if (s.tracker != null) {
                                                synchronized (mAm.mProcessStats.mLock) {
                                                    s.tracker.setBound(false, mAm.mProcessStats.getMemFactorLocked(),
                                                            SystemClock.uptimeMillis());
                                                }
                                            }
                                        }
                                        bringDownServiceIfNeededLocked(s, true, hasAutoCreate, enqueueOomAdj){
                                            if (isServiceNeededLocked(r, knowConn, hasConn)) {
                                                return;
                                            }
                                    
                                            // Are we in the process of launching?
                                            if (mPendingServices.contains(r)) {
                                                return;
                                            }
                                    
                                            bringDownServiceLocked(r, enqueueOomAdj); // 参见stopService流程中的
                                        }
                                    }
                                }
                            }
                            
                            if (clist.size() > 0 && clist.get(0) == r) {
                                // In case it didn't get removed above, do it now.
                                Slog.wtf(TAG, "Connection " + r + " not removed for binder " + binder);
                                clist.remove(0);
                            }
            
                            final ProcessRecord app = r.binding.service.app;
                            if (app != null) {
                                final ProcessServiceRecord psr = app.mServices;
                                if (psr.mAllowlistManager) {
                                    updateAllowlistManagerLocked(psr);
                                }
                                // This could have made the service less important.
                                if ((r.flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                                    psr.setTreatLikeActivity(true);
                                    mAm.updateLruProcessLocked(app, true, null);
                                }
                                mAm.enqueueOomAdjTargetLocked(app);
                            }
                        }
                        //更新顶层应用进程优先级
                        mAm.updateOomAdjPendingTargetsLocked(OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
            
                    } finally {
                        Binder.restoreCallingIdentity(origId);
                    }
            
                    return true;
                }
            }
        }

````


## bindService
App A bind到App B的Service

1. ContextImpl#bindService
    *App A进程*

    ContextImpl#bindService{

        bindServiceCommon(service, conn, flags, null, null, executor, getUser()){

2. 获取LoadedApk$ServiceDispatcher$IServiceConnection，用来后续连接建立完成后发布连接回调ServiceConnection的各种方法
<span id="conn"> ServiceConnection </span>

            IServiceConnection sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);

3. 通过Binder调用ActivityManagerService#bindServiceInstance

            int res = ActivityManager.getService().bindServiceInstance(
                    mMainThread.getApplicationThread(), getActivityToken(), service,
                    service.resolveTypeIfNeeded(getContentResolver()),
                    sd, flags, instanceName, getOpPackageName(), user.getIdentifier()){
    *frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java*

                return mServices.bindServiceLocked(caller, token, service, resolvedType, 
                        connection, // 前面getServiceDispatcher创建的
                        flags, instanceName, isSdkSandboxService, sdkSandboxClientAppUid,
                        sdkSandboxClientAppPackage, callingPackage, userId){
4. 查找目标service信息
    *frameworks/base/services/core/java/com/android/server/am/ActiveServices.java*

                    ServiceLookupResult res = retrieveServiceLocked(service, instanceName,
                            isSdkSandboxService, sdkSandboxClientAppUid, sdkSandboxClientAppPackage,
                            resolvedType, callingPackage, callingPid, callingUid, userId, true, callerFg,
                            isBindExternal, allowInstant);
                    ServiceRecord s = res.record;

5. 建立调用方与服务方关联

                    mAm.startAssociationLocked(callerApp.uid, callerApp.processName,
                            callerApp.mState.getCurProcState(), s.appInfo.uid, s.appInfo.longVersionCode,
                            s.instanceName, s.processName);
        
                    // bindService时指定了BIND_AUTO_CREATE，如：
                    // bindService(new Intent(context, RtcService.class), serviceConnection, Context.BIND_AUTO_CREATE);
                    if ((flags&Context.BIND_AUTO_CREATE) != 0) {
6. [bringUpService](#bringUpService)
        
                        bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                                permissionsReviewRequired, packageFrozen, true) {
        
                        }
                    }
        
                    if (s.app != null && b.intent.received) {
7. 服务已经在运行，即onBind已经执行过，则直接回调ServiceConnection.onServiceConnected

                        c.conn.connected(clientSideComponentName, b.intent.binder, false);
                    } else if (!b.intent.requested) {
8. 如果服务是因这次绑定而创建则执行Service.onBind；发布Service回调ServiceConnection.onServiceConnected

                        requestServiceBindingLocked(s, b.intent, callerFg, false);
                    }

}
}
}
   
    }


## stopService

frameworks/base/core/java/android/app/ContextImpl.java

    ContextImpl#stopService(Intent service){
        stopServiceCommon(service, mUser){
            int res = ActivityManager.getService().stopService(
                       mMainThread.getApplicationThread(), service,
                       service.resolveTypeIfNeeded(getContentResolver()), user.getIdentifier()){
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

                return mServices.stopServiceLocked(caller, service, resolvedType, userId){
frameworks/base/services/core/java/com/android/server/am/ActiveServices.java

                    stopServiceLocked(r.record, false){

                        service.startRequested = false;  

                        bringDownServiceIfNeededLocked(service, false, false, enqueueOomAdj){

如果服务仍然有用则不销毁，直接返回

                             if (isServiceNeededLocked(r, knowConn, hasConn){
                                 // Are we still explicitly being asked to run?
                                 if (r.startRequested) { // 是否被startService且未被stopservice（若stop了则会被置为false）
                                     return true;
                                 }
                         
                                 // Is someone still bound to us keeping us running?
                                 if (!knowConn) {
                                     hasConn = r.hasAutoCreateConnections(); // 是否仍然有bindService(Context.BIND_AUTO_CREATE)尚未unbind
                                 }
                                 if (hasConn) {
                                     return true;
                                 }
                         
                                 return false;
                             }) {
                                 return;
                             }


                             // Are we in the process of launching?
                             if (mPendingServices.contains(r)) { // 不要停止正在启动中的服务
                                 return;
                             }

                            bringDownServiceLocked(r, enqueueOomAdj);
                        }

                    }
                }

            }
        }
    }


## 进程被杀流程

## Java异常崩溃流程
崩溃情形之一——Java异常，处理的前半部分（主要工作是打印崩溃信息，弹一个对话框供用户选择重启，查看appinfo等），后半部分在AMS侧，binder讣告。
若用户设置了Thread.setDefaultUncaughtExceptionHandler则不会走这里，但后半部分始终会走。

````
 Java异常处理器设置：
   //=========frameworks/base/core/java/com/android/internal/os/ZygoteServer.java
   runSelectLoop(String abiList){
    ...
    final Runnable command = connection.processCommand(this, multipleForksOK){
      //============= frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java
        ...
      if (pid == 0) {
      //============= App进程
        return handleChildProc(parsedArgs, childPipeFd, parsedArgs.mStartChildZygote){
            ...
            return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                        parsedArgs.mDisabledCompatChanges,
                        parsedArgs.mRemainingArgs, null /* classLoader */){
                ...
                RuntimeInit.commonInit(){
                  //=========== frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
                    ...
                    // 设置Java异常预处理（打印崩溃日志。即便用户设置了自己的异常处理器，预处理器依然会先运行，所以崩溃日志始终会打印出来）
                    LoggingHandler loggingHandler = new LoggingHandler();
                    RuntimeHooks.setUncaughtExceptionPreHandler(loggingHandler){
                        Thread.setUncaughtExceptionPreHandler(uncaughtExceptionHandler);
                    }
                    
                    // 设置java异常默认处理器
                    Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler(loggingHandler){
                    
                        public void uncaughtException(Thread t, Throwable e) {
                            try {
                                // 打印崩溃信息
                                ensureLogging(t, e);
                
                                // Don't re-enter -- avoid infinite loops if crash-reporting crashes.
                                if (mCrashing) return;
                                mCrashing = true;
                
                                // Try to end profiling. If a profiler is running at this point, and we kill the
                                // process (below), the in-memory buffer will be lost. So try to stop, which will
                                // flush the buffer. (This makes method trace profiling useful to debug crashes.)
                                if (ActivityThread.currentActivityThread() != null) {
                                    ActivityThread.currentActivityThread().stopProfiling();
                                }
                
                                // Bring up crash dialog, wait for it to be dismissed
                                ActivityManager.getService().handleApplicationCrash(
                                        mApplicationObject, new ApplicationErrorReport.ParcelableCrashInfo(e)){
                                  //=============== SystemServer进程      
                                    //================ frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
                                        ProcessRecord r = findAppProcess(app, "Crash");
                                        final String processName = app == null ? "system_server"
                                                : (r == null ? "unknown" : r.processName);
                                
                                        handleApplicationCrashInner("crash", r, processName, crashInfo){
                                            ...
                                            mAppErrors.crashApplication(r, crashInfo){
                                              //================= frameworks/base/services/core/java/com/android/server/am/AppErrors.java  
                                                crashApplicationInner(r, crashInfo, callingPid, callingUid){
                                                
                                                    boolean handled = handleAppCrashInActivityController(r, crashInfo, shortMsg, longMsg, stackTrace,
                                                            timeMillis, callingPid, callingUid){
                                                            
                                                        Runnable killCrashingAppCallback = () -> {
                                                            if (Build.IS_DEBUGGABLE
                                                                    && "Native crash".equals(crashInfo.exceptionClassName)) {
                                                                Slog.w(TAG, "Skip killing native crashed app " + name
                                                                        + "(" + pid + ") during testing");
                                                            } else {
                                                                Slog.w(TAG, "Force-killing crashed app " + name + " at watcher's request");
                                                                if (r != null) {
                                                                    if (!makeAppCrashingLocked(r, shortMsg, longMsg, stackTrace, null)) {
                                                                        r.killLocked("crash", ApplicationExitInfo.REASON_CRASH, true);
                                                                    }
                                                                } else {
                                                                    // Huh.
                                                                    Process.killProcess(pid);
                                                                    ProcessList.killProcessGroup(uid, pid);
                                                                    mService.mProcessList.noteAppKill(pid, uid,
                                                                            ApplicationExitInfo.REASON_CRASH,
                                                                            ApplicationExitInfo.SUBREASON_UNKNOWN,
                                                                            "crash");
                                                                }
                                                            }
                                                        }
                                                
                                                        return mService.mAtmInternal.handleAppCrashInActivityController(
                                                                name, pid, shortMsg, longMsg, timeMillis, crashInfo.stackTrace, killCrashingAppCallback){
                                                          //============= frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
                                                          
                                                            Runnable targetRunnable = null;
                                                            synchronized (mGlobalLock) {
                                                                if (mController == null) {
                                                                    return false;
                                                                }
                                                
                                                                try {
                                                                    if (!mController.appCrashed(processName, pid, shortMsg, longMsg, timeMillis,
                                                                            stackTrace){
                                                                      //========== frameworks/base/services/core/java/com/android/server/am/ActivityManagerShellCommand.java        
                                                                            mPw.println("** ERROR: PROCESS CRASHED");
                                                                            mPw.println("processName: " + processName);
                                                                            mPw.println("processPid: " + pid);
                                                                            mPw.println("shortMsg: " + shortMsg);
                                                                            mPw.println("longMsg: " + longMsg);
                                                                            mPw.println("timeMillis: " + timeMillis);
                                                                            mPw.println("stack:");
                                                                            mPw.print(stackTrace);
                                                                            mPw.println("#");
                                                                            mPw.flush();
                                                                            int result = waitControllerLocked(pid, STATE_CRASHED);
                                                                            return result == RESULT_CRASH_KILL ? false : true;
                                                                        }
                                                                        ) {
                                                                        targetRunnable = killCrashingAppCallback;
                                                                    }
                                                                } catch (RemoteException e) {
                                                                    mController = null;
                                                                    Watchdog.getInstance().setActivityController(null);
                                                                }
                                                            }
                                                            if (targetRunnable != null) {
                                                                targetRunnable.run();
                                                                return true;
                                                            }
                                                            return false;
                                                        }
                                                    }
    
                                                    /**
                                                     * If crash is handled by instance of {@link android.app.IActivityController},
                                                     * finish now and don't show the app error dialog.
                                                     */
                                                    if (handled) {
                                                        return;
                                                    }
                                                    
                                                    AppErrorResult result = new AppErrorResult();
                                                    AppErrorDialog.Data data = new AppErrorDialog.Data();
                                                    data.result = result;
                                                    data.proc = r;
                                                                                            
                                                    boolean bad=!makeAppCrashingLocked(r, shortMsg, longMsg, stackTrace, data){
                                                        return handleAppCrashLSPB(app, "force-crash" /*reason*/, shortMsg, longMsg,
                                                                stackTrace, data){
                                                            // 开发者选项是否打开了“允许后台应用显示‘ANR’对话框”                                                        
                                                            final boolean showBackground = Settings.Secure.getIntForUser(mContext.getContentResolver(),
                                                                    Settings.Secure.ANR_SHOW_BACKGROUND, 0,
                                                                    mService.mUserController.getCurrentUserId()) != 0;
                                                    
                                                            // Bump up the crash count of any services currently running in the proc.
                                                            // 是否存在允许重启的服务（重启次数未超限且为前台服务）
                                                            boolean tryAgain = app.mServices.incServiceCrashCountLocked(now);
                                                    
                                                            final boolean quickCrash = crashTime != null
                                                                    && now < crashTime + ActivityManagerConstants.MIN_CRASH_INTERVAL;
                                                            if (quickCrash || isProcOverCrashLimitLBp(app, now)) {
                                                                // bad进程——短时间内（2分钟）crash多次或1天内crash次数达上限
                                                                mService.mAtmInternal.onHandleAppCrash(proc);
                                                                if (!persistent) {
                                                                    // We don't want to start this process again until the user
                                                                    // explicitly does so...  but for persistent process, we really
                                                                    // need to keep it running.  If a persistent process is actually
                                                                    // repeatedly crashing, then badness for everyone.
                                                                    if (!isolated) {
                                                                        // XXX We don't have a way to mark isolated processes
                                                                        // as bad, since they don't have a persistent identity.
                                                                        markBadProcess(processName, app.uid,
                                                                                new BadProcessInfo(now, shortMsg, longMsg, stackTrace));
                                                                        mProcessCrashTimes.remove(processName, app.uid);
                                                                        mProcessCrashCounts.remove(processName, app.uid);
                                                                    }
                                                                    errState.setBad(true);
                                                                    app.setRemoved(true);
                                                                    final AppStandbyInternal appStandbyInternal =
                                                                            LocalServices.getService(AppStandbyInternal.class);
                                                                    if (appStandbyInternal != null) {
                                                                        appStandbyInternal.restrictApp(
                                                                                // Sometimes the processName is the same as the package name, so use
                                                                                // that if we don't have the ApplicationInfo object.
                                                                                // AppStandbyController will just return if it can't find the app.
                                                                                app.info != null ? app.info.packageName : processName,
                                                                                userId, UsageStatsManager.REASON_SUB_FORCED_SYSTEM_FLAG_BUGGY);
                                                                    }
                                                                    // Don't let services in this process be restarted and potentially
                                                                    // annoy the user repeatedly.  Unless it is persistent, since those
                                                                    // processes run critical code.
                                                                    mService.mProcessList.removeProcessLocked(app, false, tryAgain,
                                                                            ApplicationExitInfo.REASON_CRASH, "crash");
                                                                    mService.mAtmInternal.resumeTopActivities(false /* scheduleIdle */);
                                                                    if (!showBackground) {
                                                                        return false;
                                                                    }
                                                                }
                                                                mService.mAtmInternal.resumeTopActivities(false /* scheduleIdle */);
                                                            } else {
                                                                final int affectedTaskId = mService.mAtmInternal.finishTopCrashedActivities(
                                                                                proc, reason);
                                                                if (data != null) {
                                                                    data.taskId = affectedTaskId;
                                                                }
                                                                if (data != null && crashTimePersistent != null
                                                                        && now < crashTimePersistent + ActivityManagerConstants.MIN_CRASH_INTERVAL) {
                                                                    data.repeating = true;
                                                                }
                                                            }
                                                    
                                                            if (data != null && tryAgain) {
                                                                // 可重启
                                                                data.isRestartableForService = true;
                                                            }
                                                    
                                                            // If the crashing process is what we consider to be the "home process" and it has been
                                                            // replaced by a third-party app, clear the package preferred activities from packages
                                                            // with a home activity running in the process to prevent a repeatedly crashing app
                                                            // from blocking the user to manually clear the list.
                                                            if (proc.isHomeProcess() && proc.hasActivities() && (app.info.flags & FLAG_SYSTEM) == 0) {
                                                                proc.clearPackagePreferredForHomeActivities();
                                                            }
                                                    
                                                            if (!isolated) {
                                                                // XXX Can't keep track of crash times for isolated processes,
                                                                // because they don't have a persistent identity.
                                                                mProcessCrashTimes.put(processName, uid, now);
                                                                mProcessCrashTimesPersistent.put(processName, uid, now);
                                                                updateProcessCrashCountLBp(processName, uid, now);
                                                            }
                                                    
                                                            if (errState.getCrashHandler() != null) {
                                                                mService.mHandler.post(errState.getCrashHandler());
                                                            }
                                                            return true;
                                                        }
                                                    }
                                                    
                                                    // If we can't identify the process or it's already exceeded its crash quota,
                                                    // quit right away without showing a crash dialog.
                                                    if (r == null || bad) {
                                                        // bad进程（频繁崩溃）
                                                        return;
                                                    }
                                                    
                                                    // 弹崩溃对话框
                                                    final Message msg = Message.obtain();
                                                    msg.what = ActivityManagerService.SHOW_ERROR_UI_MSG;
                                        
                                                    taskId = data.taskId;
                                                    msg.obj = data;
                                                    mService.mUiHandler.sendMessage(msg){
                                                        final class UiHandler extends Handler {
                                                        public UiHandler() {
                                                            super(com.android.server.UiThread.get().getLooper(), null, true);
                                                        }
                                                
                                                        @Override
                                                        public void handleMessage(Message msg) {
                                                            switch (msg.what) {
                                                                case SHOW_ERROR_UI_MSG: {
                                                                    mAppErrors.handleShowAppErrorUi(msg){
                                                                        if (isBackground && !showBackground) {
                                                                            // 崩溃app在后台且设置中不允许后台弹崩溃框
                                                                            if (res != null) {
                                                                                res.set(AppErrorDialog.BACKGROUND_USER);
                                                                            }
                                                                            return;
                                                                        }
                                                                        Long crashShowErrorTime = null;
                                                                        synchronized (mBadProcessLock) {
                                                                            //各种是否能显示崩溃弹框的条件收集
                                                                            ...
                                                                            
                                                                            if ((mService.mAtmInternal.canShowErrorDialogs() || showBackground)
                                                                                    && !crashSilenced && !shouldThottle
                                                                                    && (showFirstCrash || showFirstCrashDevOption || data.repeating)) {
                                                                                Slog.i(TAG, "Showing crash dialog for package " + packageName + " u" + userId);
                                                                                // 弹崩溃对话框
                                                                                errState.getDialogController().showCrashDialogs(data);
                                                                            } 
                                                                        }
                                                                    }
                                                                    
                                                                    ensureBootCompleted();
                                                                } break;
                                                            ...
                                                        ...
                                                    }
                                                    
                                                    //进入阻塞等待，直到用户选择crash对话框"退出"或者"退出并报告"
                                                    int res = result.get();
                                                    
                                                    // 根据崩溃弹框用户的选择做处理
                                                    switch (res) {
                                                        case AppErrorDialog.MUTE:
                                                        case AppErrorDialog.RESTART:
                                                        case AppErrorDialog.FORCE_QUIT:
                                                        case AppErrorDialog.APP_INFO:
                                                        case AppErrorDialog.FORCE_QUIT_AND_REPORT:
                                                    }    
                                                    
                                                }  
                                            }
                                        }
                                }
                            } catch (Throwable t2) {
                                ...
                            } finally {
                                // Try everything to make sure this process goes away.
                                // 崩溃弹框选择后（或静默没有弹框）最终杀死进程。（随后会触发binder的讣告机制）
                                Process.killProcess(Process.myPid());
                                System.exit(10);
                            }
                        }
                        
                    });
                }
            }
        }
    }
}

````


## Native crash流程

````

监听器设置：
  //========= frameworks/base/services/java/com/android/server/SystemServer.java
    private void startOtherServices(@NonNull TimingsTraceAndSlog t) {
        mActivityManagerService.systemReady(() -> {
            mActivityManagerService.startObservingNativeCrashes(){
                final NativeCrashListener ncl = new NativeCrashListener(this){
                  //======================== frameworks/base/services/core/java/com/android/server/am/NativeCrashListener.java
                    public void run() {

                        // 创建服务端socket供crash_dump上报崩溃信息
                        FileDescriptor serverFd = Os.socket(AF_UNIX, SOCK_STREAM, 0);
                        final UnixSocketAddress sockAddr = UnixSocketAddress.createFileSystem(
                                DEBUGGERD_SOCKET_PATH);
                        Os.bind(serverFd, sockAddr);
                        Os.listen(serverFd, 1);
                        Os.chmod(DEBUGGERD_SOCKET_PATH, 0777);
            
                        while (true) { // 循环处理crash_dump的上报
                            FileDescriptor peerFd = null;
                            if (MORE_DEBUG) Slog.v(TAG, "Waiting for debuggerd connection");
                            peerFd = Os.accept(serverFd, null /* peerAddress */);
                            if (MORE_DEBUG) Slog.v(TAG, "Got debuggerd socket " + peerFd);
                            if (peerFd != null) {
                                // 收到native crash
                                consumeNativeCrashData(peerFd){
                                    ...
                                    // 通知给AMS
                                    final String reportString = new String(os.toByteArray(), "UTF-8");
                                    (new NativeCrashReporter(pr, signal, reportString)).start(){
                                        mAm.handleApplicationCrashInner("native_crash", mApp, mApp.processName, ci){
                                            // 后续处理流程同“Java异常崩溃流程”
                                        }
                                    }
                                }
                            }
                            ...
                        }
                    }
                
                }
                ncl.start();
            }

````


## ANR流程

````

// 四大组件回调都有超时时限，Service启动流程已分析了超时机制的设置，所有类型的ANR最终都会触发AnrHelper#appNotResponding

//============== frameworks/base/services/core/java/com/android/server/am/AnrHelper.java

void appNotResponding(ProcessRecord anrProcess, String activityShortComponentName,
            ApplicationInfo aInfo, String parentShortComponentName,
            WindowProcessController parentProcess, boolean aboveSystem, String annotation){
    final int incomingPid = anrProcess.mPid;
    synchronized (mAnrRecords) {
        if (incomingPid == 0) {
            // Extreme corner case such as zygote is no response to return pid for the process.
            Slog.i(TAG, "Skip zero pid ANR, process=" + anrProcess.processName);
            return;
        }
        if (mProcessingPid == incomingPid) {
            Slog.i(TAG, "Skip duplicated ANR, pid=" + incomingPid + " " + annotation);
            return;
        }
        for (int i = mAnrRecords.size() - 1; i >= 0; i--) {
            if (mAnrRecords.get(i).mPid == incomingPid) {
                Slog.i(TAG, "Skip queued ANR, pid=" + incomingPid + " " + annotation);
                return;
            }
        }
        mAnrRecords.add(new AnrRecord(anrProcess, activityShortComponentName, aInfo,
                parentShortComponentName, parentProcess, aboveSystem, annotation));
    }
    startAnrConsumerIfNeeded();
}

````


## binder讣告机制

````
讣告设置：
ActivityThread.main(){
  //======== APP进程
    attach(){
        ActivityManager.getService().attachApplication(mAppThread, startSeq){
          //======== SystemServer进程，AMS
            // 设置讣告
            AppDeathRecipient adr = new AppDeathRecipient(app, pid, thread);
            thread.asBinder().linkToDeath(adr, 0);
            app.setDeathRecipient(adr);
        }
    }
}

讣告回调（app崩溃/被杀，先走java crash/native crash/anr等的前置处理，前置处理主要是杀掉进程，后置处理（讣告）主要是做AMS侧的清理）：
    
    AppDeathRecipient#binderDied() {
      //=========== frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
        appDiedLocked(mApp, mPid, mAppThread, true, null){
            ...
            handleAppDiedLocked(app, pid, false, true, fromBinderDied){
                boolean kept = cleanUpApplicationRecordLocked(app, pid, restarting, allowRestart, -1,
                        false /*replacingPid*/, fromBinderDied){

                    // 清理provider, 并判断是否存在正在工作的contentprovider且有客户端已和它建立了连接，若有则restart=true
                    restart = app.onCleanupApplicationRecordLSP(mProcessStats, allowRestart,
                            fromBinderDied || app.isolated /* unlinkDeath */){
                        mErrorState.onCleanupApplicationRecordLSP(){
                            // 消隐crash/anr等对话框
                            getDialogController().clearAllErrorDialogs();
                        }
                        
                        if (unlinkDeath) {
                            unlinkDeathRecipient();
                        }
                        ...
                        mState.onCleanupApplicationRecordLSP();
                        mServices.onCleanupApplicationRecordLocked();
                        mReceivers.onCleanupApplicationRecordLocked();

                        return mProviders.onCleanupApplicationRecordLocked(allowRestart){
                            ...
                            // 如果存在正在执行中的provider且有client连接则需要重启
                            if (!alwaysRemove && inLaunching && cpr.hasConnectionOrHandle()) {
                                // We left the provider in the launching list, need to
                                // restart it.
                                restart = true;
                            }
                        }
                    }
                    
                    ...
                    
                    // 清理Receiver
                    skipCurrentReceiverLocked(app);
                    
                    // 清理service（可能重启service）
                    mServices.killServicesLocked(app, allowRestart){
                        ...
                        // Now do remaining service cleanup.
                        for (int i = psr.numberOfRunningServices() - 1; i >= 0; i--) {
                            ServiceRecord sr = psr.getRunningServiceAt(i);
                            ...
                            // Any services running in the application may need to be placed
                            // back in the pending list.
                            if (allowRestart && sr.crashCount >= mAm.mConstants.BOUND_SERVICE_MAX_CRASH_RETRY
                                    && (sr.serviceInfo.applicationInfo.flags
                                        &ApplicationInfo.FLAG_PERSISTENT) == 0) {
                                bringDownServiceLocked(sr, true);
                            } else if (!allowRestart
                                    || !mAm.mUserController.isUserRunning(sr.userId, 0)) {
                                bringDownServiceLocked(sr, true);
                            } else { // 尝试重启
                                final boolean scheduled = scheduleServiceRestartLocked(sr, true /* allowCancel */){
                                    ...
                                    // 过滤一些不需要重启的情形
                                    ...
                                    
                                    if (allowCancel) {
                                        final boolean shouldStop = r.canStopIfKilled(canceled){
                                            // 是否需要重启service。stopIfKilled即为onstartcommand的返回值
                                            return startRequested && (stopIfKilled || isStartCanceled) && pendingStarts.isEmpty();
                                        }
                                        if (shouldStop && !r.hasAutoCreateConnections()) {
                                            // Nothing to restart.
                                            return false;
                                        }
                                        reason = (r.startRequested && !shouldStop) ? "start-requested" : "connection";
                                    } else {
                                        reason = "always";
                                    }
                                    
                                    ...
                                    // 计算重启延时
                                    r.restartCount = x;
                                    r.restartDelay = x;
                                    r.nextRestartTime = now;
                                    ...
                                    
                                    cancelForegroundNotificationLocked(r);
                                    
                                    // 重启service
                                    performScheduleRestartLocked(r, "Scheduling", reason, now){
                                        ...
                                        mAm.mHandler.postAtTime(r.restarter, r.nextRestartTime){ //restarter是创建service时(ActiveServices#startServiceLocked)构造的
                                            private class ServiceRestarter implements Runnable {
                                                ...
                                                public void run() {
                                                    performServiceRestartLocked(mService){
                                                        ...
                                                        // 启动service
                                                        bringUpServiceLocked(r, r.intent.getIntent().getFlags(), r.createdFromFg, true, false,false, true);
                                                    }
                                                }
                                            }
                                        }
                                        ...
                                    }
                                    
                                    return true;
                                }
                                ...
                            }
                        }
                        ...
                        
                        psr.stopAllExecutingServices();
                    }
            
                    mProcessList.scheduleDispatchProcessDiedLocked(pid, app.info.uid);
            
                    ...
                    
                    if (!app.isPersistent() || app.isolated) {
                        if (!replacingPid) {
                            mProcessList.removeProcessNameLocked(app.processName, app.uid, app);
                        }
                        mAtmInternal.clearHeavyWeightProcessIfEquals(app.getWindowProcessController());
                    } else if (!app.isRemoved()) {
                        if (mPersistentStartingProcesses.indexOf(app) < 0) {
                            // Persistent进程需要保持运行（重启）
                            mPersistentStartingProcesses.add(app);
                            restart = true;
                        }
                    }
                    ...
                    
                    if (restart && allowRestart && !app.isolated) {
                        ...
                        // 重启app
                        mProcessList.startProcessLocked(app, new HostingRecord(
                                HostingRecord.HOSTING_TYPE_RESTART, app.processName),
                                ZYGOTE_POLICY_FLAG_EMPTY);
                        return true;
                    } else if (pid > 0 && pid != MY_PID) {
                        // Goodbye!
                        removePidLocked(pid, app);
                        ...
                    }
                    return false;
                }

                if (!kept && !restarting) {
                    removeLruProcessLocked(app);
                    if (pid > 0) {
                        ProcessList.remove(pid);
                    }
                }
        
                mAppProfiler.onAppDiedLocked(app);
        
                // 处理activity
                mAtmInternal.handleAppDied(app.getWindowProcessController(), restarting){
                  //=========== frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
                    ...
                    if (!restarting && hasVisibleActivities) {
                        deferWindowLayout();
                        try {
                            if (!mRootWindowContainer.resumeFocusedTasksTopActivities()) { // 详见Activity启动流程的resumeFocusedTasksTopActivities
                                // If there was nothing to resume, and we are not already restarting
                                // this process, but there is a visible activity that is hosted by the
                                // process...then make sure all visible activities are running, taking
                                // care of restarting this process.
                                mRootWindowContainer.ensureActivitiesVisible(null, 0,
                                        !PRESERVE_WINDOWS);
                            }
                        } finally {
                            continueWindowLayout();
                        }
                    }
                    ...
                }
            }
        }
    }

````


## 清空最近任务列表（单独删除某一个Task流程类似）

````
最近任务列表在launcher中实现：

//=========== packages/apps/Launcher3/src/com/android/launcher3/anim/AnimatorListeners.java

    public void onAnimationEnd(Animator anim) {
        if (!mListenerCalled) {
            mListenerCalled = true;
            mListener.accept(anim instanceof ValueAnimator
                    ? ((ValueAnimator) anim).getAnimatedFraction() > SUCCESS_TRANSITION_PROGRESS
                    : true){
                ...
                // mListener设置的地方
                //=============== packages/apps/Launcher3/quickstep/src/com/android/quickstep/views/RecentsView.java    
                mPendingAnimation.addEndListener(isSuccess -> {
                    if (isSuccess) {
                        // Remove all the task views now
                        finishRecentsAnimation(true /* toRecents */, false /* shouldPip */, () -> {
                            UI_HELPER_EXECUTOR.getHandler().postDelayed(
                                ActivityManagerWrapper.getInstance()::removeAllRecentTasks{
                                  //============ frameworks/base/packages/SystemUI/shared/src/com/android/systemui/shared/system/ActivityManagerWrapper.java
                                    getService().removeAllVisibleRecentTasks(){
                                      //========== SystemServer进程
                                      //=========== frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
                                        
                                        getRecentTasks().removeAllVisibleTasks(mAmInternal.getCurrentUserId()){
                                          //================== frameworks/base/services/core/java/com/android/server/wm/RecentTasks.java
                                            Set<Integer> profileIds = getProfileIds(userId);
                                            for (int i = mTasks.size() - 1; i >= 0; --i) {
                                                final Task task = mTasks.get(i);
                                                if (!profileIds.contains(task.mUserId)) continue;
                                                if (isVisibleRecentTask(task)) {
                                                    mTasks.remove(i);
                                                    notifyTaskRemoved(task, true /* wasTrimmed */, true /* killProcess */){
                                                        for (int i = 0; i < mCallbacks.size(); i++) {
                                                            mCallbacks.get(i).onRecentTaskRemoved(task, wasTrimmed, killProcess){
                                                              //=========== frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java
                                                                if (wasTrimmed) {
                                                                    // Task was trimmed from the recent tasks list -- remove the active task record as well
                                                                    // since the user won't really be able to go back to it
                                                                    removeTaskById(task.mTaskId, killProcess, false /* removeFromRecents */,
                                                                            "recent-task-trimmed"){
                                                                        final Task task =
                                                                                mRootWindowContainer.anyTaskForId(taskId, MATCH_ATTACHED_TASK_OR_RECENT_TASKS);
                                                                        if (task != null) {
                                                                            removeTask(task, killProcess, removeFromRecents, reason){
                                                                            
                                                                                // 清理activity
                                                                                task.removeActivities(reason, false /* excludingTaskOverlay */);
                                                                                
                                                                                cleanUpRemovedTaskLocked(task, killProcess, removeFromRecents){

                                                                                    // Find any running services associated with this app and stop if needed.
                                                                                    // 清理服务
                                                                                    final Message msg = PooledLambda.obtainMessage(ActivityManagerInternal::cleanUpServices{
                                                                                        ...
                                                                                      //================= frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
                                                                                        ArrayList<ServiceRecord> services = new ArrayList<>();
                                                                                        //获得此用户下所有的活动Service
                                                                                        ArrayMap<ComponentName, ServiceRecord> alls = getServicesLocked(userId);
                                                                                        for (int i = alls.size() - 1; i >= 0; i--) {
                                                                                            ServiceRecord sr = alls.valueAt(i);
                                                                                            if (sr.packageName.equals(component.getPackageName())) {
                                                                                                services.add(sr);
                                                                                            }
                                                                                        }
                                                                                
                                                                                        // Take care of any running services associated with the app.
                                                                                        boolean needOomAdj = false;
                                                                                        for (int i = services.size() - 1; i >= 0; i--) {
                                                                                            ServiceRecord sr = services.get(i);
                                                                                            if (sr.startRequested) {
                                                                                                if ((sr.serviceInfo.flags&ServiceInfo.FLAG_STOP_WITH_TASK) != 0) {
                                                                                                    //如果在manifest里设置了stopWithTask，那么会直接停止Service
                                                                                                    Slog.i(TAG, "Stopping service " + sr.shortInstanceName + ": remove task");
                                                                                                    needOomAdj = true;
                                                                                                    stopServiceLocked(sr, true);
                                                                                                } else {
                                                                                                    //如果没有设置stopWithTask的话，则会回调Service.onTaskRemoved方法
                                                                                                    sr.pendingStarts.add(new ServiceRecord.StartItem(sr, true,
                                                                                                            sr.getLastStartId(), baseIntent, null, 0));
                                                                                                    if (sr.app != null && sr.app.getThread() != null) {
                                                                                                        // We always run in the foreground, since this is called as
                                                                                                        // part of the "remove task" UI operation.
                                                                                                        try {
                                                                                                            sendServiceArgsLocked(sr, true, false){
                                                                                                              //========== frameworks/base/core/java/android/app/ActivityThread.java
                                                                                                                private void handleServiceArgs(ServiceArgsData data) {
                                                                                                                    ...
                                                                                                                    int res;
                                                                                                                    if (!data.taskRemoved) {
                                                                                                                        //正常情况调用
                                                                                                                        res = s.onStartCommand(data.args, data.flags, data.startId);
                                                                                                                    } else {
                                                                                                                        //用户关闭Task栈时调用
                                                                                                                        s.onTaskRemoved(data.args);
                                                                                                                        res = Service.START_TASK_REMOVED_COMPLETE;
                                                                                                                    }
                                                                                                                    ...
                                                                                                                }
                                                                                                            }    

                                                                                                        } catch (TransactionTooLargeException e) {
                                                                                                            // Ignore, keep going.
                                                                                                        }
                                                                                                    }
                                                                                                }
                                                                                            }
                                                                                        }
                                                                                        if (needOomAdj) {
                                                                                            mAm.updateOomAdjPendingTargetsLocked(OomAdjuster.OOM_ADJ_REASON_UNBIND_SERVICE);
                                                                                        }
                                                                                    
                                                                                    },
                                                                                            mService.mAmInternal, task.mUserId, component, new Intent(task.getBaseIntent()));
                                                                                    mService.mH.sendMessage(msg);
                                                                            
                                                                                    if (!killProcess) {
                                                                                        return;
                                                                                    }
                                                                            
                                                                                    // Determine if the process(es) for this task should be killed.
                                                                                    final String pkg = component.getPackageName();
                                                                                    ArrayList<Object> procsToKill = new ArrayList<>();
                                                                                    ArrayMap<String, SparseArray<WindowProcessController>> pmap =
                                                                                            mService.mProcessNames.getMap();
                                                                                    for (int i = 0; i < pmap.size(); i++) {
                                                                            
                                                                                        SparseArray<WindowProcessController> uids = pmap.valueAt(i);
                                                                                        for (int j = 0; j < uids.size(); j++) {
                                                                                            WindowProcessController proc = uids.valueAt(j);
                                                                                            // check一些不能杀进程的场景
                                                                                            if (proc.mUserId != task.mUserId) {
                                                                                                // Don't kill process for a different user.
                                                                                                continue;
                                                                                            }
                                                                                            if (proc == mService.mHomeProcess) {
                                                                                                // Don't kill the home process along with tasks from the same package.
                                                                                                continue;
                                                                                            }
                                                                                            if (!proc.mPkgList.contains(pkg)) {
                                                                                                // Don't kill process that is not associated with this task.
                                                                                                continue;
                                                                                            }
                                                                            
                                                                                            if (!proc.shouldKillProcessForRemovedTask(task)) {
                                                                                                // Don't kill process(es) that has an activity in a different task that is also
                                                                                                // in recents, or has an activity not stopped.
                                                                                                return;
                                                                                            }
                                                                            
                                                                                            if (proc.hasForegroundServices()) {
                                                                                                // Don't kill process(es) with foreground service.
                                                                                                return;
                                                                                            }
                                                                            
                                                                                            // Add process to kill list.
                                                                                            procsToKill.add(proc);
                                                                                        }
                                                                                    }
                                                                            
                                                                                    // Kill the running processes. Post on handle since we don't want to hold the service lock
                                                                                    // while calling into AM.
                                                                                    // 杀掉所有满足条件的进程（一个task可能关联多个进程）
                                                                                    final Message m = PooledLambda.obtainMessage(
                                                                                            ActivityManagerInternal::killProcessesForRemovedTask, mService.mAmInternal,
                                                                                            procsToKill){
                                                                                      //============== frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java      
                                                                                        for (int i = 0; i < procsToKill.size(); i++) {
                                                                                            final WindowProcessController wpc =
                                                                                                    (WindowProcessController) procsToKill.get(i);
                                                                                            final ProcessRecord pr = (ProcessRecord) wpc.mOwner;
                                                                                            if (pr.mState.getSetSchedGroup() == ProcessList.SCHED_GROUP_BACKGROUND
                                                                                                    && pr.mReceivers.numberOfCurReceivers() == 0) {
                                                                                                pr.killLocked("remove task", ApplicationExitInfo.REASON_USER_REQUESTED,
                                                                                                        ApplicationExitInfo.SUBREASON_REMOVE_TASK, true);
                                                                                            } else {
                                                                                                // We delay killing processes that are not in the background or running a
                                                                                                // receiver.
                                                                                                pr.setWaitingToKill("remove task");
                                                                                            }
                                                                                        }
                                                                                    }
                                                                                    mService.mH.sendMessage(m);
                                                                                }
                                                                                
                                                                                ...
                                                                            }
                                                                            return true;
                                                                        }
                                                                        return false;
                                                                    }
                                                                }
                                                                task.removedFromRecents();
                                                            }
                                                        }
                                                        mTaskNotificationController.notifyTaskListUpdated();
                                                    }
                                                }
                                            }
                                        }
                                    }
                                },
                                REMOVE_TASK_WAIT_FOR_APP_STOP_MS);
                            removeTasksViewsAndClearAllButton();
                            startHome();
                        });
                    }
                    mPendingAnimation = null;
                });
                    
            }
        }
    }

````


## setContentView流程

````

//=============== frameworks/base/core/java/android/app/ActivityThread.java
  public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    ...    
    final Activity a = performLaunchActivity(r, customIntent){
        ...
        activity = mInstrumentation.newActivity(appContext.getClassLoader(), component.getClassName(), r.intent);
        activity.attach(){
          //=============== frameworks/base/core/java/android/app/Activity.java
            ...
            // 创建PhoneWindow
            mWindow = new PhoneWindow(this, window, activityConfigCallback);
            mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0){
                // 创建WindowManagerImpl
                mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this){
                    return new WindowManagerImpl(mContext, parentWindow, mWindowContextToken);
                }
            }
            ...
        }
        ...
        mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState){
            activity.performCreate(icicle, persistentState){
                // onCreate回调
                onCreate(icicle){
                  //=============== frameworks/base/core/java/android/app/Activity.java
                    setContentView(View view) {
                        getWindow().setContentView(view){ // 前面创建的PhoneWindow
                          //================ frameworks/base/core/java/com/android/internal/policy/PhoneWindow.java
                            setContentView(view, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT)){
                                ...
                                // 创建DecorView
                                installDecor(){
                                    ...
                                    mDecor = generateDecor(-1){... return new DecorView(context, featureId, this, getAttributes()); }
                                    ...
                                    mContentParent = generateLayout(mDecor){
                                        ...
                                        // 根据主题样式（如有无toolbar），为DecorView匹配合适的layout文件
                                        if (xxx){
                                            ...
                                        }else if (xxx){
                                            ...
                                        } else {
                                            // 一般会匹配这个layout: frameworks/base/core/res/res/layout/screen_simple.xml
                                            /* 
                                            <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
                                                android:layout_width="match_parent"
                                                android:layout_height="match_parent"
                                                android:fitsSystemWindows="true"
                                                android:orientation="vertical">
                                                <ViewStub android:id="@+id/action_mode_bar_stub" // toolbar位置
                                                          .../>
                                                <FrameLayout
                                                     android:id="@android:id/content" // 用户设置的view填入的位置
                                                     ... />
                                            </LinearLayout>
                                            */
                                            layoutResource = R.layout.screen_simple;
                                        }
                                        
                                        mDecor.onResourcesLoaded(mLayoutInflater, layoutResource){
                                            ...
                                            mDecorCaptionView = createDecorCaptionView(inflater);
                                            final View root = inflater.inflate(layoutResource, null);
                                            ...
                                            mContentRoot = (ViewGroup) root;
                                            ...
                                        }
                                
                                        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT); // 这个即为上面xml中的id为content的FrameLayout
                                        ...
                                        return contentParent;
                                    }
                                }
                                ...
                                // 将用户设置的view添加进decorview的content部分
                                mContentParent.addView(view, params);
                            }
                        }    
                    }// setContentView完，此时DecorView还未可见

                }
            }
        }
    }



````

## setContentView后view的显示流程（setContentView后DecorView还未可见）

````
//=============== frameworks/base/core/java/android/app/ActivityThread.java
    public void handleResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
        boolean isForward, boolean shouldSendCompatFakeFocus, String reason) {
        performResumeActivity(r, finalStateRequest, reason){
            r.activity.performResume(r.startsNotResumed, reason){
                mInstrumentation.callActivityOnResume(this){
                    // onResume回调，此时还无法获取窗口宽高
                    activity.onResume();
                }
            }
        }
    
        final Activity a = r.activity;
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            // 在完成测绘前先设置 DecorView 不可见，等测绘结束再设置为可见
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            ...
            
            // 通过 WindowManager（WindowManagerImpl） 将 DecorView添加到窗口
            // 期间会完成测绘，而后即可展示
            wm.addView(decor, l){
              //============== frameworks/base/core/java/android/view/WindowManagerImpl.java
                applyTokens(params);
                mGlobal.addView(view, params, mContext.getDisplayNoVerify(), mParentWindow,
                        mContext.getUserId()){
                  //========== frameworks/base/core/java/android/view/WindowManagerGlobal.java      
                    ViewRootImpl root;  
                    ...
                    // 创建 ViewRootImpl。负责测绘view,负责和WMS通信（通过windowsession）,负责处理UI输入事件（触屏、按键等事件通过ViewRootHandler）
                    // 可以被理解为“View树的管理者”——它有一个mView成员变量，它指向的对象和上文中Window和Activity的mDecor指向的对象是同一个对象。
                    root = new ViewRootImpl(view.getContext(), display){
                      //=============== frameworks/base/core/java/android/view/ViewRootImpl.java
                        this(context, display, WindowManagerGlobal.getWindowSession(){
                          //==================== frameworks/base/core/java/android/view/WindowManagerGlobal.java
                            // 创建WindowSession
                            InputMethodManager.ensureDefaultInstanceForDefaultDisplayIfNecessary();
                            IWindowManager windowManager = getWindowManagerService(){
                                // WMS在客户端的proxy
                                sWindowManagerService = IWindowManager.Stub.asInterface(ServiceManager.getService("window"));
                                return sWindowManagerService;
                            }
                            sWindowSession = windowManager.openSession(
                                new IWindowSessionCallback.Stub() {
                                    @Override
                                    public void onAnimatorScaleChanged(float scale) {
                                        ValueAnimator.setDurationScale(scale);
                                    }
                                }){
                                  //=============== frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
                                    return new Session(this, callback);
                            }
                            return sWindowSession;
                        }, new WindowLayout());
                    }
                    ...
                    // 把 Window 对应的 View 设置给创建的 ViewRootImpl，通过 ViewRootImpl 来更新界面并添加到 WIndow中。
                    // 完成view的测绘
                    root.setView(view, wparams, panelParentView, userId){
                      //============== frameworks/base/core/java/android/view/ViewRootImpl.java
                        mView = view; // activity的DecorView
                        ...
                        requestLayout(){
                            scheduleTraversals(){
                                mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable{
                                    @Override
                                    public void run() {
                                        doTraversal(){
                                            performTraversals(){
                                                ...
                                                // Ask host how big it wants to be
                                                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec){
                                                    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
                                                }
                                                
                                                performLayout(lp, mWidth, mHeight){
                                                    mView.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
                                                }
                                                
                                                if (!isViewVisible) {
                                                } else if (cancelAndRedraw) {
                                                } else {
                                                    performDraw(){
                                                        boolean canUseAsync = draw(fullRedrawNeeded, usingAsyncReport && mSyncBuffer){
                                                            ...
                                                            if (isHardwareEnabled()) {
                                                                ...
                                                            }else{
                                                                drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                                                                    scalingRequired, dirty, surfaceInsets);
                                                            }
                                                        }
                                                    }
                                                }
                                            }
                                        }
                                    }
                                }, null);
                            }
                        }
                        
                        ...
                        // 将 Window 添加到屏幕上，mWindowSession 实现了 IWindowSession接口，是 Session 的 Binder 对象。
                        // addToDisplay 是一次 AIDL 的过程，通知 WindowManagerServer 添加 Window
                        // mWindowSession即为前面“windowManager.openSession”创建的
                        res = mWindowSession.addToDisplayAsUser(mWindow, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), userId,
                            mInsetsController.getRequestedVisibilities(), inputChannel, mTempInsets,
                            mTempControls, attachedFrame, compatScale){
                          //=============== SystemServer进程WMS
                          //================ frameworks/base/services/core/java/com/android/server/wm/Session.java  
                            return mService.addWindow(this, window, attrs, viewVisibility, displayId, userId,
                                    requestedVisibilities, outInputChannel, outInsetsState, outActiveControls,
                                    outAttachedFrame, outSizeCompatScale){
                              //============= frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java       
                                // 主要就是创建了一个和Window一一对应的WindowState对象，并将WindowState插入到父容器WindowToken的子容器集合中，
                                // 而WindowToken又保存在DisplayContent的键值对集合中。 DisplayContent中包含WindowToken列表，WindowToken包含WindowState列表
                                // WindowState对应一个具体的窗口如activity窗口，dialog窗口，WindowToken负责管理具有相关性的一组WindowState，如同一个App下的所有窗口（Activity,Dialog,Toast,InputMehotd,等等）
                                ...
                                final DisplayContent displayContent = getDisplayContentOrCreate(displayId, attrs.token);
                                ...
                                if (type >= FIRST_SUB_WINDOW && type <= LAST_SUB_WINDOW) {
                                    // 这是一个子窗口，它肯定依附于一个父窗口
                                    parentWindow = windowForClientLocked(null, attrs.token, false);
                                }
                                ...
                                // 创建WindowToken
                                token = new WindowToken.Builder(this, binder, type)
                                .setDisplayContent(displayContent)
                                .setOwnerCanManageAppTokens(session.mCanAddInternalSystemWindow)
                                .setRoundedCornerOverlay(isRoundedCornerOverlay)
                                .setFromClientToken(true)
                                .setOptions(options)
                                .build();
                                ...
                                // 创建WindowState。概念上对应App侧的Window
                                final WindowState win = new WindowState(this, session, client, token, parentWindow,
                                    appOp[0], attrs, viewVisibility, session.mUid, userId, session.mCanAddInternalSystemWindow){
                                    void attach() {
                                        mSession.windowAddedLocked(){
                                            // 创建SurfaceSession用于和SurfaceFlinger通信
                                            mSurfaceSession = new SurfaceSession(){
                                                mNativeClient = nativeCreate(){
                                                  //==============frameworks/base/core/jni/android_view_SurfaceSession.cpp
                                                    // 实现细节都在native层
                                                    SurfaceComposerClient* client = new SurfaceComposerClient();
                                                    // SurfaceComposerClient构造函数是个空实现。但后续通过incStrong增加一次强引用计数。在第一次增加强引用后会执行onFirstRef()方法
                                                    client->incStrong((void*)nativeCreate)；触发=> onFirstRef() {
                                                        sp<ISurfaceComposer> sf(ComposerService::getComposerService()); //获取到SurfaceFlinger的Binder代理类对象
                                                        if (sf != nullptr && mStatus == NO_INIT) {
                                                            sp<ISurfaceComposerClient> conn;
                                                            //通过Binder调用createConnection与SurfaceFlinger建立连接，创建一个Client
                                                            conn = sf->createConnection(){
                                                              //================ frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp
                                                                sp<ISurfaceComposerClient> SurfaceFlinger::createConnection() {
                                                                    // 内部创建的Client对象是ISurfaceComposerClient的Binder代理实现类，其内部操作还是转交给SurfaceFlinger调用。
                                                                    const sp<Client> client = new Client(this){
                                                                        // 创建surface
                                                                        status_t Client::createSurface(const String8& name, uint32_t /* w */, uint32_t /* h */,
                                                                                                       PixelFormat /* format */, uint32_t flags,
                                                                                                       const sp<IBinder>& parentHandle, LayerMetadata metadata,
                                                                                                       sp<IBinder>* outHandle, sp<IGraphicBufferProducer>* /* gbp */,
                                                                                                       int32_t* outLayerId, uint32_t* outTransformHint) {
                                                                            // We rely on createLayer to check permissions.
                                                                            LayerCreationArgs args(mFlinger.get(), this, name.c_str(), flags, std::move(metadata));
                                                                            // 最终还是通过SurfaceFlinger创建
                                                                            return mFlinger->createLayer(args, outHandle, parentHandle, outLayerId, nullptr,
                                                                                                         outTransformHint);
                                                                        }
                                                                    }
                                                                    return client->initCheck() == NO_ERROR ? client : nullptr;
                                                                }
                                                            }
                                                        }
                                                    }
                                                    
                                                    return reinterpret_cast<jlong>(client);
                                                }
                                            }
                                        }
                                    }
                                }
                                ...
                                // 根据mPolicy调整window的attr属性，mPolicy的实现类是PhoneManagerPolicy
                                final DisplayPolicy displayPolicy = displayContent.getDisplayPolicy();
                                displayPolicy.adjustWindowParamsLw(win, win.mAttrs);
                                ...
                                // 执行WindowState的openInputChannel，这里主要是打通和Input系统的通道，用于接收IMS的输入事件请求。 
                                win.openInputChannel(outInputChannel);
                                ...
                                // 将客户端app层的Window对象和WindowState关联上，这样WMS就可以通过Window找到WMS中的WindowState对象
                                mWindowMap.put(client.asBinder(), win);
                                ...
                                // 将WindowState加入到WindowToken的子容器集合中。 
                                win.mToken.addWindow(win);
                                ...
                                // 设置进入的过渡动画
                                final WindowStateAnimator winAnimator = win.mWinAnimator;
                                winAnimator.mEnterAnimationPending = true;
                                winAnimator.mEnteringAnimation = true;
                                ...
                                if (focusChanged) {
                                    displayContent.getInputMonitor().setInputFocusLw(displayContent.mCurrentFocus,
                                            false /*updateInputWindows*/);
                                }
                                displayContent.getInputMonitor().updateInputWindowsLw(false /*force*/);

                            }
                        }
                        ...
                        // 设置当前 View 的 Parent
                        view.assignParent(this);
                    }
                }
            }
            ...
            
            // 测绘已完成，恢复可见性
            r.activity.makeVisible(){
                ...
                mDecor.setVisibility(View.VISIBLE);
            }
        }
    }
````

## SurfaceView
每个Activity一般有一个Window（弹框会有自己的window）,每个Window有一个surface,surface对应到SurfaceFlinger侧是一个layer, SF负责把各个layer合成起来最终形成完整的图像进行渲染。
UI的渲染默认要求在主线程，主要为了避免多线程问题，如果加锁来允许多线程，效率太低，所以规定只能在UI线程渲染。大部分情况下没问题，但有些对渲染效率要求非常高的场景
比如视频流，会导致UI线程负担过重，而UI线程还有其他工作比如响应用户事件。所以需要提供在UI线程以外渲染的机制。SurfaceView应运而生，SurfaceView既是一个普通的View
在其宿主窗口的视图树中，可以在onDraw中通过canvas绘制内容到宿主window的surface上，同时它又有自己的独立的Surface,该Surface可以在单独的线程绘制。这点可以通过setzorderontop(true)来验证——
设置onTop后视频图像会覆盖onDraw中绘制的内容，因为视频图像是绘制在SurfaceView独有的Surface上，而onDraw是绘制在宿主窗口的Surface上，
onTop就把独有的surface盖在主窗口的surface上了，所以就会覆盖其他UI显示了。默认SF的surface是在宿主窗口的Surface之下的（为了不被遮盖会将宿主窗口的surface“掏个洞”（设置透明）），因为一般要在视频内容上方添加操作按钮（这些操作按钮最终也是绘制到主窗口的surface上的）。


## PackageManagerService app安装流程

### PackageManagerService启动流程
PackageManagerService启动过程中会扫描并解析当前系统中存在的应用并记录下解析的信息供后续使用（比如Launcher需要，startActivity也需要查找）

````
//============ frameworks/base/services/java/com/android/server/SystemServer.java
    // 启动system server时会启动packagemanagerservice,期间会加载已安装app（扫描特定目录下的apk文件，如/data/app是普通应用的安装目录(base.apk)。（/data/data下是app的数据））
    // 的信息到PackageManagerService（解析AndroidManifest.xml），
    // 这个过程只相当于在PackageManagerService注册应用相关信息。如果我们想要在Android桌面上看到这些应用程序，还需要有一个Home应用程序，
    // 负责从PackageManagerService服务中把这些安装好的应用程序取出来，并以友好的方式在桌面上展现出来，例如以快捷图标的形式。
    // 在Android系统中，负责把系统中已经安装的应用程序在桌面中展现出来的Home应用程序就是Launcher了。
    private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
        ...
        Pair<PackageManagerService, IPackageManager> pmsPair = PackageManagerService.main(
            mSystemContext, installer, domainVerificationService,
            mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore){
              //================ frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
                ...
                // 创建PackageManagerService实例
                PackageManagerService m = new PackageManagerService(injector, onlyCore, factoryTest,
                    PackagePartitions.FINGERPRINT, Build.IS_ENG, Build.IS_USERDEBUG,
                    Build.VERSION.SDK_INT, Build.VERSION.INCREMENTAL){
                    ...
                    mAppInstallDir = new File(Environment.getDataDirectory(), "app");    // 普通app安装目录
                    ...
                    // 解析/data/system/package.xml（已安装app的概要信息）
                    mFirstBoot = !mSettings.readLPw(computer,
                                mInjector.getUserManagerInternal().getUsers(
                                /* excludePartial= */ true,
                                /* excludeDying= */ false,
                                /* excludePreCreated= */ false));
                    ...
                    // 加载系统app
                    mOverlayConfig = mInitAppsHelper.initSystemApps(packageParser, packageSettings, userIds,startTime);
                    // 加载普通app
                    mInitAppsHelper.initNonSystemApps(packageParser, userIds, startTime){
                      //============= frameworks/base/services/core/java/com/android/server/pm/InitAppsHelper.java
                        scanDirTracedLI(mPm.getAppInstallDir(), /* frameworkSplits= */ null, 0,
                            mScanFlags | SCAN_REQUIRE_KNOWN,
                            packageParser, mExecutorService){
                            mInstallPackageHelper.installPackagesFromDir(scanDir, frameworkSplits, parseFlags,
                                    scanFlags, packageParser, executorService){
                              //============= frameworks/base/services/core/java/com/android/server/pm/InstallPackageHelper.java
                                ...
                                parallelPackageParser.submit(file, parseFlags){
                                  //=========== frameworks/base/services/core/java/com/android/server/pm/ParallelPackageParser.java
                                    ...
                                    parsePackage(scanFile, parseFlags){
                                        return mPackageParser.parsePackage(scanFile, parseFlags, true, mFrameworkSplits){
                                          //================ frameworks/base/services/core/java/com/android/server/pm/parsing/PackageParser2.java
                                            ParseResult<ParsingPackage> result = parsingUtils.parsePackage(input, packageFile, flags,
                                                frameworkSplits){
                                              //=============== frameworks/base/services/core/java/com/android/server/pm/pkg/parsing/ParsingPackageUtils.java
                                                if (((flags & PARSE_FRAMEWORK_RES_SPLITS) != 0)
                                                        && frameworkSplits.size() > 0
                                                        && packageFile.getAbsolutePath().endsWith("/framework-res.apk")) {
                                                    return parseClusterPackage(input, packageFile, frameworkSplits, flags);
                                                } else if (packageFile.isDirectory()) {
                                                    return parseClusterPackage(input, packageFile, /* frameworkSplits= */null, flags);
                                                } else {
                                                    return parseMonolithicPackage(input, packageFile, flags){
                                                        // 每个安装在"/data/app"目录下的应用都有一个base.apk。其实就是原始apk,导出该apk也可直接安装。
                                                        final ParseResult<ParsingPackage> result = parseBaseApk(input,
                                                            apkFile,
                                                            apkFile.getCanonicalPath(),
                                                            assetLoader, flags){
                                                            
                                                            // 清单文件（AnddroidManifest.xml）解析器。
                                                            /// 安装过程的主要工作是解析清单文件。
                                                            XmlResourceParser parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);
                                                            
                                                            ParseResult<ParsingPackage> result = parseBaseApk(input, apkPath, codePath, res, parser, flags){
                                                                ...
                                                                ApkLiteParseUtils.parsePackageSplitNames(input, parser);
                                                                ...
                                                                final ParseResult<ParsingPackage> result = parseBaseApkTags(input, pkg, manifestArray, res, parser, flags){
                                                                    ...
                                                                    result = parseBaseApplication(input, pkg, res, parser, flags){
                                                                        //解析四大组件
                                                                        switch (tagName) {
                                                                            case "activity":
                                                                            ...
                                                                            case "receiver":
                                                                            ...
                                                                            case "service":
                                                                            ...
                                                                    }
                                                                    ...
                                                                    result = parseBaseApkTag(tagName, input, pkg, res, parser, flags){
                                                                        switch (tag) {
                                                                            case "key-sets":
                                                                                ...
                                                                            case "feature": 
                                                                                ...
                                                                            case "permission":
                                                                                ...
                                                                            case "uses-permission":
                                                                                ...
                                                                        }    
                                                                    }
                                                                }
                                                            }
                                                        }
                                                    }
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }
                    
                    // 保存已解析的app结果
                    mSettings.writeLPr(mLiveComputer);
                    
                }
                ...
                
                IPackageManagerImpl iPackageManager = m.new IPackageManagerImpl(); // 创建Binder通信对象（stub）
                ServiceManager.addService("package", iPackageManager); // 注册到ServiceManager
                final PackageManagerNative pmn = new PackageManagerNative(m);
                ServiceManager.addService("package_native", pmn);
                LocalManagerRegistry.addManager(PackageManagerLocal.class, m.new PackageManagerLocalImpl());
                return Pair.create(m, iPackageManager);
            }
        ...
    }

````

### 文件管理器中点击APK文件安装APP

````

    // 发起安装请求
    {
        ...
        File file = new File(getExternalFilesDir(null), "xxx.apk");
        
        Intent intent = new Intent(Intent.ACTION_VIEW);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        Uri apkUri =FileProvider.getUriForFile(this, "com.xxx.xxx.fileprovider", file);
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        intent.setDataAndType(apkUri, "application/vnd.android.package-archive");
        startActivity(intent);
    }
    
    // 启动安装器经历的过程如下
    com.google.android.packageinstaller/com.android.packageinstaller.InstallStaging: +349ms // 对应从content URI 读取 apk 文件
    com.google.android.packageinstaller/com.android.packageinstaller.PackageInstallerActivity: +1s516ms // 对应解析apk 提示未知来源安装的设置，以及展示APK icon和名称并提示安装确认。
    com.android.settings/.Settings$ManageAppExternalSourcesActivity: +385ms  // 设置页面-允许未知来源安装
    com.google.android.packageinstaller/com.android.packageinstaller.InstallInstalling: +296ms // 安装中
    com.google.android.packageinstaller/com.android.packageinstaller.InstallSuccess: +375ms // 安装完成
    
    

````


## Paragraphs
 I really like using Markdown.

**I think** I'll use it to <br>
format ***ALL*** of my documents *from now on*. 

> Dorothy followed her through many of the beautiful rooms in her castle.
>
> The Witch bade her clean the pots and kettles and sweep the floor and keep the fire fed with 


## Lists

1. First item
    - First item
    - Second item
1. Second item
1. Third item
1. Fourth item 


## Code

To denote a word or phrase as code, enclose it in backticks (`).

-----------

At the command prompt, type `nano`.
