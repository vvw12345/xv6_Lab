# Lab1 基本配置

## sleep

``` c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc,char *argv[]){
    if(argc < 2){
        printf("The number of the parameter is insufficient!\n");
        exit(1);
    }
    int time = atoi(argv[1]); // atoi将字符串里面的字符转换为数字
    sleep(time); 
    exit(0);
}
```

![image-20250102173659829](Lab1.assets/image-20250102173659829.png)



## pingpong

有两个需要注意的点

第一个是先创建管道再fork子进程，如果是先fork子进程的话，那么每一对进程都会创建两个管道；

第二个是在fork之后的if判断里面再创建管道的话，shell上不会实时出现结果，需要回车几下……

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int buf[2] = {0};

int main(int argc,char *argv[]){
    if(argc != 1){
        printf("The number of parameter is wrong!\n");
        exit(1);
    }
    int first[2],second[2]; //两组文件描述符，用于创建两个管道
    if(pipe(first) < 0){
        printf("pingpong:pipe failed!\n");
        exit(1);
    }
    if(pipe(second) < 0){
        printf("pingpong:pipe failed!\n");
        exit(1);
    }
    int pid = fork();
    if(pid != 0){ //父进程
        int parent = getpid(); //获取父进程当前的进程id
        write(first[1],"x",1);
        read(second[0],"x",1);
        printf("%d: received pong\n",parent);
        exit(0);
    }else{ //子进程
        int child = getpid(); //获取子进程当前的进程id
        read(first[0],"x",1);
        printf("%d: received ping\n",child);
        write(second[1],"x",1);
        exit(0);
    }
}
```



## find

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void find(char *path,char *target){
    char buf[512], *p;
    int fd;
    struct dirent de; 
    struct stat st; 
    
    // open系统调用根据path读，返回一个文件描述符fd
    if((fd = open(path, 0)) < 0){
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    //根据当前文件描述符fd看看是否存在，如果能正确赋值的话，按照定义就可以取出type
    if(fstat(fd, &st) < 0){ 
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    // 判断是文件还是目录
    switch(st.type){
    //如果是文件的话就可以直接比对
    case T_FILE:
        if(strcmp(path+strlen(path)-strlen(target), target) == 0){
            printf("%s\n", path);
        }
        break;

    // 如果是目录的话
    case T_DIR:
        // 那么当前目录的长度再加一个文件的长度不能太长了（因为现在是./的形式，不能连一个文件名也放不下了）
        if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
            printf("find: path too long\n");
            break;
        }
        strcpy(buf, path);
        p = buf+strlen(buf);
        *p++ = '/';
        // 当前fd是目录，提取目录下的每一个内容
        while(read(fd, &de, sizeof(de)) == sizeof(de)){
            // 如果这个目录里面什么也没有，可以跳过了 
            if(de.inum == 0)
                continue;
            memmove(p, de.name, DIRSIZ);
            p[DIRSIZ] = 0;
            if(stat(buf, &st) < 0){
				printf("find: cannot stat %s\n", buf);
				continue;
			}
            // 不要进入 `.` 和 `..`
			if(strcmp(buf+strlen(buf)-2, "/.") != 0 && strcmp(buf+strlen(buf)-3, "/..") != 0) {
				find(buf, target); // 递归查找
			}
        }
        break;
    }
    close(fd);
}

int main(int argc, char *argv[]){
    if(argc <= 2){
        printf("The number of parameter is wrong!\n");
        exit(1);
    }
    char target[512]; //要寻找的目标路径
    target[0] = '/';
    strcpy(target+1, argv[2]);
	find(argv[1], target);
    exit(0);
}
```



## xargs

做这个实验对管道有更深入的理解了……

首先buffer中的字符串中的管道符`|`是不需要我们手动管理的

一条指令`echo hello | xargs ehco hi bye`执行的流程应该是

- `shell`依据管道符将指令分为左右两个部分
- 左侧创建一个子进程执行，右侧创建一个子进程执行
- **把左侧的执行结果重定向到右侧进程的输入端中**

因此我们实际要手动处理的输入是xargs指令的相关参数

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"


// Q1:如何获取到前一个命令的标准化输出
// Q2:如何获取到自己的命令行参数(将前一个命令的输出和管道符后面自己的输入结合起来)
// Q3:如何使用exec执行这个命令
int main(int argc, char *argv[]){
    sleep(10);
    // 新建个缓冲区读入数据
    char buf[16];
    read(0,buf,16);

    // xargs执行的命令参数和刚开始的argv[]并不是完全一致
    char *xargv[16];
    int xargc = 0; 
    
    // argv[0]对应的是原始的命令名称
    // 例如: echo hello | xargs echo hi bye 的第一个echo
    for(int i = 1; i < argc; ++i){
        xargv[xargc] = argv[i];
        xargc++;
    }

    char *p = buf; // 新建一个指针对其起始位置
    // 对整个缓冲区进行扫描
    for(int i = 0;i < 16; ++i){
        if(buf[i] == '\n'){
            int pid = fork();
            if(pid > 0){  // 父进程负责扫描输入字符串，继续扫描下去
                p = &buf[i + 1];
                wait(0);
            }else{ // 子进程需要执行前一行的命令
                buf[i] = '\0';
                xargv[xargc] = p;
                xargc++;
                xargv[xargc] = '\0';
                xargc++;
				
                //  xargv[0]是xargs需要执行的指令的名称 永远保持不变
                exec(xargv[0],xargv);
                exit(0);
            }
        }
    }
    wait(0);
    exit(0);
}
```

