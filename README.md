# nginx-windows

#### 环境依赖：

1. MSYS
2. vs2010（2010以上版本无法编译windows2003运行的执行文件）
3. Perl （我使用的是 [ActivePerl](http://www.activestate.com/activeperl) ）
4. [Mercurial](https://www.mercurial-scm.org/)  （源码管理器，获取nginx完整源码）
5. [PCRE](http://www.pcre.org/), [zlib](http://zlib.net/) [OpenSSL](http://www.openssl.org/) 库源码



#### 1、Perl升级

新版的openssl依赖比较新的Perl，mingw自带的perl版本太老了，编译不通过。需要先对perl升级。

- 下载mingw版本perl https://sourceforge.net/projects/perl-mingw/
- 解压后的perl 放到mingw目录下（如 C:\MinGW\msys\1.0）
- 修改etc/profile 添加perl路径  ```export PATH=".:/usr/local/bin:/perl/bin:/mingw/bin:/bin:$PATH"```
- 运行msys.bat 启动mingw环境。运行perl -V，perl已经升级为5.24.符合openssl编译的最低要求

#### 2、获取编译参数

从nginx官网下载官方编译的最新稳定版。目前最新版为1.18.0

运行 nginx -V，获取官方编译参数

```
nginx version: nginx/1.18.0
built by cl 16.00.40219.01 for 80x86
built with OpenSSL 1.1.1f  31 Mar 2020
TLS SNI support enabled
configure arguments: --with-cc=cl --builddir=objs.msvc8 --with-debug --prefix= --conf-path=conf/nginx.conf --pid-path=logs/nginx.pid --http-log-path=logs/access.log --error-log-path=logs/error.log --sbin-path=nginx.exe --http-client-body-temp-path=temp/client_body_temp --http-proxy-temp-path=temp/proxy_temp --http-fastcgi-temp-path=temp/fastcgi_temp --http-scgi-temp-path=temp/scgi_temp --http-uwsgi-temp-path=temp/uwsgi_temp --with-cc-opt=-DFD_SETSIZE=1024 --with-pcre=objs.msvc8/lib/pcre-8.44 --with-zlib=objs.msvc8/lib/zlib-1.2.11 --with-http_v2_module --with-http_realip_module --with-http_addition_module --with-http_sub_module --with-http_dav_module --with-http_stub_status_module --with-http_flv_module --with-http_mp4_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_auth_request_module --with-http_random_index_module --with-http_secure_link_module --with-http_slice_module --with-mail --with-stream --with-openssl=objs.msvc8/lib/openssl-1.1.1f --with-openssl-opt='no-asm no-tests -D_WIN32_WINNT=0x0501' --with-http_ssl_module --with-mail_ssl_module --with-stream_ssl_module
```

可知，官方用的编译器 “cl 16.00.40219.01 for 80x86”  属于vc2010 sp1版本。依据这份参数，也可以获取依赖的第三方库版本，建议保持与官方相同的版本。

#### 3、获取源码

运行msys.bat,启动mingw环境。使用hg获取nginx完整源码。注意：网站download页面的源码压缩包，仅包含unix环境源码，不含win32源码。

```shell
hg clone  -b stable-1.18 http://hg.nginx.org/nginx
```

```cd nginx```

```  curl -O https://www.openssl.org/source/openssl-1.1.1f.tar.gz ```

``` curl -O https://ftp.pcre.org/pub/pcre/pcre-8.44.tar.gz ```

``` curl -O https://zlib.net/zlib-1.2.11.tar.gz ```



```
mkdir objs
mkdir objs/lib
cd objs/lib
tar -xzf ../../pcre-8.44.tar.gz
tar -xzf ../../zlib-1.2.11.tar.gz
tar -xzf ../../openssl-1.1.1f.tar.gz
cd ../..
```

#### 4、增加ngx_http_proxy_connect_module模块

clone 源码到nginx目录下

`git clone https://github.com/chobits/ngx_http_proxy_connect_module.git`

根据文档对源码打补丁

`patch -p1 < ngx_http_proxy_connect_module/patch/proxy_connect_rewrite_1018.patch`

#### 5、开始编译,生成Makefile

```shell
auto/configure \
--with-cc=cl \
--with-debug \
--prefix= \
--conf-path=conf/nginx.conf \
--pid-path=logs/nginx.pid \
--http-log-path=logs/access.log \
--error-log-path=logs/error.log \
--sbin-path=nginx.exe \
--http-client-body-temp-path=temp/client_body_temp \
--http-proxy-temp-path=temp/proxy_temp \
--http-fastcgi-temp-path=temp/fastcgi_temp \
--http-scgi-temp-path=temp/scgi_temp \
--http-uwsgi-temp-path=temp/uwsgi_temp \
--with-cc-opt=-DFD_SETSIZE=1024 \
--with-http_v2_module \
--with-http_realip_module \
--with-http_addition_module \
--with-http_sub_module \
--with-http_dav_module \
--with-http_stub_status_module \
--with-http_flv_module \
--with-http_mp4_module \
--with-http_gunzip_module \
--with-http_gzip_static_module \
--with-http_auth_request_module \
--with-http_random_index_module \
--with-http_secure_link_module \
--with-http_slice_module \
--with-mail \
--with-stream \
--with-pcre=objs/lib/pcre-8.44 \
--with-zlib=objs/lib/zlib-1.2.11 \
--with-openssl=objs/lib/openssl-1.1.1f \
--with-openssl-opt='no-asm no-tests -D_WIN32_WINNT=0x0501' \
--with-http_ssl_module \
--with-mail_ssl_module \
--with-stream_ssl_module \
--add-module=ngx_http_proxy_connect_module
```

#### 6、使用vc编译器编译

在命令行中运行vs2010目录下的 `vcvars32.bat` 准备环境变量。

```shell
nmake
```

如果打过补丁的源码，在编译过程中会报错：

==src/http/ngx_http_request.c(1128) : error C2275: “ngx_int_t”: 将此类型用作表达式非法==

这是因为c编译器要求类型声明必须在函数或作用域最前面的源码。打开ngx_http_request.c文件第1128行，移动声明位置后继续编译。

还会报错：

==ngx_http_proxy_connect_module/ngx_http_proxy_connect_module.c(1524) : error C2275: “ngx_str_t”: 将此类型用作表达式非法==

再次修改下位置，继续编译。

编译完成后会有错误提示

=='sed' 不是内部或外部命令，也不是可运行的程序==
==或批处理文件。==
==NMAKE : fatal error U1077: “sed”: 返回代码“0x1”==
==Stop.==

sed是处理文本的，不需理会，nginx已经编译成功