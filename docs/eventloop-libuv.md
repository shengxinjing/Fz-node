# 事件循环libuv入门

想要详细了解 node 中的事件循环是如何运作的，通过看网上的文章我依旧觉得不是很稳。究其原因就是我对libuv的不熟悉，不管是内部原理，还是外部API都是所知甚少。导致在看文章的时候，诸如```uv_loop_open()```等等API甚是陌生。这对于理解事件循环的本质并不是很有帮助，所以我打算花一点时间，对其进行入门。


幸亏当年写过一年多的c/c++程序，如今只是半吊子，还是能够快速入门libuv，熟悉API。


# 搭建libuv开发环境

- [libuv的仓库](https://github.com/libuv/libuv)

仓库里有很详细的各个平台的安装方法：
- [windows](https://github.com/libuv/libuv#windows)
- [*nix](https://github.com/libuv/libuv#unix)
- [mac](https://github.com/libuv/libuv#os-x)

# Hello world

安装完毕以后，引入头文件，最简单的hello world
```js
#include <stdio.h>
#include <stdlib.h>
#include <uv.h>

int main() {
    printf("Hello world.\n");
    uv_loop_t * loop = uv_default_loop();
    uv_run(loop, UV_RUN_DEFAULT);
    
    uv_loop_close(loop);
    return 0;
}
//output:Hello world.
```
以上的几行代码我们熟悉一下：
- ```v_loop_t * loop = uv_default_loop();```:初始化loop，使用默认loop来跑。node中也是使用默认的loop。
- ```uv_run(loop, UV_RUN_DEFAULT);```:跑loop。
- ``` uv_loop_close(loop);```:关闭loop和释放loop分配的内存

到此，我们对```libuv```有了第一认识。

# 尝试读取一下文件

读写文件使用的是```uv_fs_**```这种样式的函数

在libuv中，文件操作同时提供了```同步 synchronous``` 和 ```异步 asynchronous.```的接口，这和我们的node非常像，调用方式更加像。异步版本API的接口使用的是内部的线程池模型去驱动异步。废话不多说，我们来看看一个api

```js
int uv_fs_open(uv_loop_t* loop, uv_fs_t* req, const char* path, int flags, int mode, uv_fs_cb cb)
```
c语言的参数都很长，解释下参数
- ```loop```:loop变量
- ```req```:类型是```uv_fs_t```结构体实例的一个指针，这个参数会在完成io之后，往最后的```cb```中，传入
- ```path```:明显，就是文件的地址了
- ```flags```和```mode```：参数flags与mode和标准的 Unix flags 相同，具体可以查看unix read api 的flags和mode
- ```cb```:这个就是我们的callback函数了，这个函数必须是接受```uv_fs_t* ```为参数的一个函数

创建一个文件，```text.txt```
```js
i m file
```

我们快速使用一下这个api
```c#
#include <stdio.h>
#include <uv.h>

uv_fs_t open_req;

void on_open(uv_fs_t *req) {
    printf("%zd\n",req->result);//输出10
}

int main() {
    const char* path = "/Users/zf/Desktop/Fz-node/libuv-simple/libuv-simple/text.txt";
    uv_fs_open(uv_default_loop(), &open_req,path, O_RDONLY, 0, on_open);
    uv_run(uv_default_loop(), UV_RUN_DEFAULT);
    uv_fs_req_cleanup(&open_req);
    return 0;
}
```
这么一来，我们的思路一目了然，填写path之后，调用```uv_fs_open```，然后跑loop，当打开文件结束之后，我们就会到达```on_open```这个callback中。值得注意的是，在c中，打开文件和读文件属于分开的逻辑，两步回调，也是够蛋疼的，但是为了获得极限的性能，异步进行到底。

我们得到的结果会存储在全局变量```open_req```中，实际上on_open中的```*req```就是指向这个全局变量。接下来我们要进行一下读操作：
```c#
#include <stdio.h>
#include <uv.h>

uv_fs_t open_req;
uv_fs_t _read;

static char buffer[1024];
static uv_buf_t iov;

void on_read(uv_fs_t *req) {
    printf("%s\n",iov.base);
}
void on_open(uv_fs_t *req) {
    printf("%zd\n",req->result);
    iov = uv_buf_init(buffer, sizeof(buffer));
    uv_fs_read(uv_default_loop(), &_read, (int)req->result,
               &iov, 1, -1, on_read);
}
int main() {
    const char* path = "/Users/zf/Desktop/Fz-node/libuv-simple/libuv-simple/text.txt";
    uv_fs_open(uv_default_loop(), &open_req,path, O_RDONLY, 0, on_open);
    uv_run(uv_default_loop(), UV_RUN_DEFAULT);
    uv_fs_req_cleanup(&open_req);
    return 0;
}
```
通过两步callback，我们终于获得文件中的内容，打印出来```i m file```。在```on_open``中做了以下几个事：
- req->result 是用于判断读取成功与否的标志位，分别有三种值：大于0，小于0，以及等于0。大于0成功，小于0失败
- uv_buf_init 将一个全局变量```buffer```初始化成```uv_buf_t```的类型
- uv_fs_read 读取函数，跟open函数很类似，注意多了一个参数：iov，read函数会把读到的数据塞进iov中
- 读取完毕以后，来到```on_read```函数，结果放在```iov.base```中







