## 项目介绍
这是一个简单的轻量HTTP服务器，能够让我们理解服务器工作的流程与本质。
[我的GitHub地址](https://github.com/BlackGhostLzc/Tinyhttpd.git)。
对代码我做了一点更改，例如再服务器进程中打印一些报文的内容以及做了比较详细的注释，还有更改了`simpleclient.c`文件使得这个程序也可以与`http`进行非`GET`非`POST`的通信。



## 程序运行
编译好项目后，首先运行`http`程序，`http.c`文件中位服务器指定了一个端口，然后可以打开浏览器，输入`localhost:端口号`或者`127.0.0.1:端口号`就可以连接到服务器。
在输入框中输入一个颜色，点击提交，得到下面的效果图。
<img src="https://pic4.zhimg.com/80/v2-c0f3f89359dc034bf5597b83caa42243_1440w.webp" alt="img" style="zoom:50%;" />


## 前置知识

### HTTP请求
当我们在浏览器（客户端）输入一个网址，浏览器就会向服务器发出一个HTTP报文。
这里并不讲HTTP报文的格式，只介绍`Tinyhttpd`用到的的请求方法：
1. `GET`: 请求指定的页面信息，并返回实体主体。
2. `POST`:向指定资源提交数据进行处理请求（例如提交表单或者上传文件）。数据被包含在请求体中。`POST` 请求可能会导致新的资源的建立和/或已有资源的修改。


### CGI脚本
CGI不是一门编程语言。它是网页的表单和你写的程序之间通信的一种协议。
典型的CGI脚本做了如下的事情：
1. 读取用户提交表单的信息。
2. 处理这些信息（也就是实现业务）。
3. 输出，返回html响应（返回处理完的数据）
```
#!/usr/bin/perl -Tw

use strict;
use CGI;

my($cgi) = new CGI;

print $cgi->header;
my($color) = "blue";
$color = $cgi->param('color') if defined $cgi->param('color');

print $cgi->start_html(-title => uc($color),
                       -BGCOLOR => $color);
print $cgi->h1("This is $color");
print $cgi->end_html;
```
上面这个CGI脚本就生成了一个html格式的文件，服务器进程创建的子进程会运行这个脚本，通过管道，服务器进程向子进程传入执行这个脚本需要的一些参数，也就是客户端提交的表单，而子进程也通过管道把脚本执行内容传给服务器进程，再由服务器发送给客户端，这样实现了动态解析。



## 代码讲解
`Tinyhttpd`有下面这几个函数：
```c
void accept_request(int);//处理从套接字上监听到的一个 HTTP 请求
void bad_request(int);//返回给客户端这是个错误请求，400响应码
void cat(int, FILE *);//读取服务器上某个文件写到 socket 套接字
void cannot_execute(int);//处理发生在执行 cgi 程序时出现的错误
void error_die(const char *);//把错误信息写到 perror 
void execute_cgi(int, const char *, const char *, const char *);//运行cgi脚本，这个非常重要，涉及动态解析
int get_line(int, char *, int);//读取一行HTTP报文
void headers(int, const char *);//返回HTTP响应头
void not_found(int);//返回找不到请求文件
void serve_file(int, const char *);//调用 cat 把服务器文件内容返回给浏览器。
int startup(u_short *);//开启http服务，包括绑定端口，监听，开启线程处理链接
void unimplemented(int);//返回给浏览器表明收到的 HTTP 请求所用的 method 不被支持。
```

### accept_request函数
相关代码逻辑在注释中。
```c
void accept_request(void *arg)
{
    setbuf(stdout, NULL);
    int client = (intptr_t)arg;
    char buf[1024];
    size_t numchars;
    char method[255];
    char url[255];
    char path[512];
    size_t i, j;
    struct stat st;
    int cgi = 0;      /* becomes true if server decides this is a CGI
                       * program */
    char *query_string = NULL;

    // 获取请求报文的第一行
    numchars = get_line(client, buf, sizeof(buf));
    i = 0; j = 0;

    printf("First line: %s", buf);

    // 获取请求报文的方法，是GET还是POST方法
    while (!ISspace(buf[i]) && (i < sizeof(method) - 1))
    {
        method[i] = buf[i];
        i++;
    }
    j=i;
    method[i] = '\0';

    // 如果不是这两种方法，那就只能是simpleclient程序
    if (strcasecmp(method, "GET") && strcasecmp(method, "POST"))
    {
        buf[0] = '#';
        // 向simpleclient发送一个字符
        write(client, buf, 1);
        unimplemented(client);
        return;
    }

    // 如果是POST方法，那么需要执行cgi脚本
    if (strcasecmp(method, "POST") == 0)
        cgi = 1;


    i = 0;
    while (ISspace(buf[j]) && (j < numchars))
        j++;
    
    // 获取url
    while (!ISspace(buf[j]) && (i < sizeof(url) - 1) && (j < numchars))
    {
        url[i] = buf[j];
        i++; j++;
    }
    url[i] = '\0';

    // 如果是GET方法，这里的url是  / 
    printf("url: %s\n", url);
	
	// url还需要进行拼接操作，把htdocs目录下的index.html发送给客户端

    if (strcasecmp(method, "GET") == 0)
    {
        // 如果有查询参数
        query_string = url;
        while ((*query_string != '?') && (*query_string != '\0'))
            query_string++;
        if (*query_string == '?')
        {
            cgi = 1;
            *query_string = '\0';
            query_string++;
        }
    }

    sprintf(path, "htdocs%s", url);
    // 如果是GET方法：  path:htdocs/
    // 如果是POST方法： path:htdocs/color.cgi


    if (path[strlen(path) - 1] == '/')
        strcat(path, "index.html");
    
    printf("path: %s\n", path);
    // 如果是GET方法：    path: htdocs/index.html
    // 如果是POST方法：   path: htdocs/color.cgi

    // 把path文件与st关联起来
    if (stat(path, &st) == -1) {
        while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
            numchars = get_line(client, buf, sizeof(buf));
        not_found(client);
    }
    else
    {
        // 如果路径是个目录，那就将主页进行显示
        if ((st.st_mode & S_IFMT) == S_IFDIR)
            strcat(path, "/index.html");
        if ((st.st_mode & S_IXUSR) ||
                (st.st_mode & S_IXGRP) ||
                (st.st_mode & S_IXOTH)    )
            cgi = 1;
        if (!cgi)
            // 为客户端发送静态网页
            serve_file(client, path);
        else
            execute_cgi(client, path, method, query_string);
    }

    close(client);
}
```


### execute_cgi函数
这个函数是进行动态解析的核心函数，
```c
void execute_cgi(int client, const char *path,
        const char *method, const char *query_string)
{
    printf("Get into execute_cgi\n");

    char buf[1024];
    int cgi_output[2];
    int cgi_input[2];
    pid_t pid;
    int status;
    int i;
    char c;
    int numchars = 1;
    int content_length = -1;

    buf[0] = 'A'; buf[1] = '\0';

    // 如果是GET请求，就不断的读取并丢弃头信息
    if (strcasecmp(method, "GET") == 0)
        while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
            numchars = get_line(client, buf, sizeof(buf));
    else if (strcasecmp(method, "POST") == 0) /*POST*/
    {
        // POST请求
        printf("This is a POST method\n"); 
        numchars = get_line(client, buf, sizeof(buf));
        
         /*
        ....
        Content-Length: 9
        ....
        空一行 \n
        内容(color=red)
        */
        
        while ((numchars > 0) && strcmp("\n", buf))
        {
            buf[15] = '\0';
            // 得到消息体的长度是多少
            if (strcasecmp(buf, "Content-Length:") == 0)
                content_length = atoi(&(buf[16]));
            numchars = get_line(client, buf, sizeof(buf));
            printf("%s", buf);
        }
        if (content_length == -1) {
            bad_request(client);
            return;
        }
    }
    else/*HEAD or other*/
    {
    }

	// 初始化管道
    if (pipe(cgi_output) < 0) {
        cannot_execute(client);
        return;
    }
    if (pipe(cgi_input) < 0) {
        cannot_execute(client);
        return;
    }

    if ( (pid = fork()) < 0 ) {
        cannot_execute(client);
        return;
    }
    sprintf(buf, "HTTP/1.0 200 OK\r\n");
    send(client, buf, strlen(buf), 0);

    if (pid == 0)  /* child: CGI script */
    {
        // 子进程来执行 cgi 脚本
        char meth_env[255];
        char query_env[255];
        char length_env[255];

        printf("子进程执行%s程序\n", path);
		
		// 操作文件描述符，使得执行cgi脚本的时候，print打印直接交给管道的写端
        dup2(cgi_output[1], STDOUT);
        dup2(cgi_input[0], STDIN);
        close(cgi_output[0]);
        close(cgi_input[1]);

        sprintf(meth_env, "REQUEST_METHOD=%s", method);
        putenv(meth_env);
        if (strcasecmp(method, "GET") == 0) {
            sprintf(query_env, "QUERY_STRING=%s", query_string);
            putenv(query_env);
        }
        else {   /* POST */
            sprintf(length_env, "CONTENT_LENGTH=%d", content_length);
            putenv(length_env);
        }

        // 执行 cgi 脚本，cgi脚本中的 print 都会通过管道重定向给父进程
        execl(path, NULL);
        exit(0);
    } else {    /* parent */
        close(cgi_output[1]);
        close(cgi_input[0]);
        if (strcasecmp(method, "POST") == 0){
            printf("父进程接受到的消息内容是:\n");
            for (i = 0; i < content_length; i++) {
                recv(client, &c, 1, 0);
                // 父进程把收到的消息内容传递给子进程，也就是 “color=red” 这条信息，由cgi生成新的html文件
                // 再由父进程（服务器）转交给客户端
                write(cgi_input[1], &c, 1);
                printf("%c",c);
            }
            printf("\n");
        }
        
        printf("父进程通过管道接收到的：\n");

        while (read(cgi_output[0], &c, 1) > 0){
            send(client, &c, 1, 0);
            printf("%c", c);
        }
        /*
        我把父进程接收到子进程的字符写进了new.html文件中，可以发现这就是一个网页
        */
        close(cgi_output[0]);
        close(cgi_input[1]);
        waitpid(pid, &status, 0);
    }
}
```
最后，把服务器进程通过管道收到的CGI脚本输出内容再发送给客户端，html文件如下所示，这也正是点击提交后再浏览器上展示的网页。
```html
<!DOCTYPE html
    PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml" lang="en-US" xml:lang="en-US">

<head>
    <title>RED</title>
    <meta http-equiv="Content-Type" content="text/html; charset=iso-8859-1" />
</head>

<body bgcolor="red">
    <h1>This is red</h1>
</body>

</html>
```

## 小结
通过这个项目，清楚了HTTP是如何响应和处理客户端请求的，明白了HTTP服务器的工作机制，也对CGI脚本有了一点了解。

