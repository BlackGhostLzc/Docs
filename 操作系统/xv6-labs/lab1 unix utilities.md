## Prime

这道编程题目是lab1中最难的一个了。我们观察它课程网站上给的算法图示，我们可以知道，第一个进程通过管道不断往子进程写入这些数，然后子进程会特别地接收第一个管道里的数，记为`first_num`，然后打印这个`first_num`，因为这个数一定是素数。然后在它后续接收到的数中，判断这个数是否是`first_num`的整数倍，如果不是，证明它可能是素数，然后它通过管道再向它自己的子进程写入这个数，如此往复。

- 代码如下：
```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

void solve(int *fd){
    int first_number = 0;
    if (read(fd[0], &first_number, sizeof(first_number)) == 4){
        printf("prime %d \n", first_number);
    }
    else{
        exit(1);
    }

    int newfd[2];
    if (pipe(newfd) == -1){
        exit(1);
    }
    pid_t pid = fork();
    if (pid > 0){
        close(newfd[0]);
        // 一边从 fd[0] 端读取，一边向 newfd[1] 写入
        int receive_data;
        while (read(fd[0], &receive_data, sizeof(receive_data)) == 4){
            if (receive_data % first_number){
                write(newfd[1], &receive_data, sizeof(receive_data));
            }
        }
    }
    else{
        solve(newfd);
    }
}

int main(int argc, char *argv[]){
    int n;
    n = atoi(argv[1]);

    int fd[2];
    if (pipe(fd) < 0){
        exit(0);
    }
    pid_t pid = fork();
    if (pid == 0){
        solve(fd);
    }
    else{
        close(fd[0]); // 关闭读端
        for (int i = 2; i <= n; i++){
            write(fd[1], &i, sizeof(i));
        }
    }
}
```



## xargs 

本质是一道阅读理解题和字符串处理题，要求是读取标准输出流（文件描述符为0）的内容作为某一次`xargs`的`argv`参数，需要执行多少次就执行多少次。



## find
仿照`ls.c`可以写出来,需要注意`..`和`.`这两个特殊的文件，放置避免无穷递归。
```c
void find(char *currentp, char *filename){
  char buf[512];
  char *p;
  int fd;
  struct dirent de;
  struct stat st;

  if ((fd = open(currentp, 0)) < 0){
    fprintf(2, "Cannot open %s\n", currentp);
    return;
  }

  if (fstat(fd, &st) < 0){
    fprintf(2, "Cannot stat %s\n", currentp);
    close(fd);
    return;
  }

  if (st.type == T_FILE){
    printf("Find argv[1] must be a DIR\n");
    return;
  }

  if (strlen(currentp) + 1 + DIRSIZ + 1 > sizeof buf){
    printf("find: path too long\n");
    return;
  }

  strcpy(buf, currentp);
  p = buf + strlen(buf);
  *p++ = '/';

  while (read(fd, &de, sizeof(de)) == sizeof(de)){
    if (de.inum == 0)
      continue;
    if (strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0){
      continue;
    }
    memmove(p, de.name, DIRSIZ);
    p[DIRSIZ] = 0;
    if (stat(buf, &st) < 0){
      printf("cannot stat %s\n", buf);
      continue;
    }

    if (st.type == T_FILE){
      if (!strcmp(de.name, filename)){
        printf("%s\n", buf);
      }
    }
    else if (st.type == T_DIR){
      find(buf, filename);
    }
  }
  close(fd);
}

int main(int argc, char *argv[]){
  if (argc != 3){
    printf("find: find error\n");
    exit(1);
  }

  find(argv[1], argv[2]);
  exit(0);
}
```