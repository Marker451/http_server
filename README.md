# http_server
A simple http server for studty


这是一个基于Tinyhttp写的 简单http服务器
////////////////////////
整体结构：只处理静态网页、网页数据存储在mysql中、使用memcached做缓存、服务器主体设计成半同步/半异步模式
          使用epoll 和boost的扩展库threadpool.
	  设计成两级页面，一级页面是存储在htdocs中的 html页面，二级静态页面存储在数据库中

//////////////////
主要接口功能：
	accept_request：处理线程
	
	memcache_inti ：初始化memcached 默认有两个缓存服务结点，监听在端口11211 11212
	
	query_memcache: 从缓存中查询数据
	
	query_database: 从数据库中查询数据
	
	header_and_cat: 发送响应头及网页数据
	
	startup       : 启动服务器程序，初始化监听套接字，绑定，监听
	
	AcceptConn    ：当epoll_wait 返回事件为监听套接字可读时，接收新连接，并加入监听队列
	
	socket_send,socket_recv:  非阻塞接收发送函数

/////////////////////////
主要工作流程
	

	主程序启动 调用startup 设置好监听端口

	初始化epoll，将listen套接字 加入监听队列，监听EPOLLIN事件，并使用ET模式

	epoll_wait返回时，检查是否为listen套接字上的可读事件，如果是，则循环调用（处理多个连接同时到来）
	调用AcceptConn 接受连接，并将新连接加入监听队列。

	如果epoll_wait 返回的非listen上的事件，则为已建立的连接可读，此时设置线程accept_request来处理,
	并将线程请求加入threadpool的请求队列

accept_request中的处理逻辑

	读取socket上的信息，分析请求类型，及请求的资源名称。
	
	若请求主页，则读取htdocs文件夹中的html文件，并封装返回
	
	若请求的资源为二级页面，则先从memcached中查询，如果命中，将页面信息存储在page_info中。
	
	如果没有命中，则去mysql中查询，在数据库中查询到页面信息之后存到page_info中，并把信息同步set到
	memcached中，最后用header_and_cat 发送响应头及页面信息

////////////////
环境及测试

	运行环境 CentOs  libevent  memcached   boost的扩展库pthreadpool mysql

	使用了 http_load 和webbench 进行测试，在我的机器上（各种服务都搭在本地）本地连接测试大概能达到

	 3000 fetch/sec 左右。把客户端测试放到其他机子上，局域网内连接测试。由于网络，不同机子硬件配置

	等原因，没能得到有参考意义的数据（高并发测的时候把同学的老爷机跑死了。。。。。）
	
///////////////
其他	
	在高并发下仍有不稳定的bug

	当前版本，默认网页数据已经存储在数据库中，其实我想搞的是一个天气预报的服务

	（我假想我的网站火到爆，每秒上万的用户来查看天气，好	吧。。。）

	再跑一个线程，用网络爬虫定时从网络上爬取各城市天气	信息，然后处理存入数据库。

	后边要把这部分整理加入，并且对高并发继续优化。
	