---
layout: post
title: 跟我一起写web server
category: other
---

# 跟我一起写web server

今天看到这篇博客，写得很不错，自己也梳理一下关于web server的思路吧。

## Web server工作原理

简单来说，web server是一个在物理主机上的服务，它等待客户端发送请求过来，然后处理请求并将结果返回给客户端。服务器和客户端的通信基于Http协议，因此任何使用Http协议的东西都可以和web服务器交互，比如浏览器、爬虫等等。

当客户端发送一个http请求到服务器时，须要指名服务器的地址，这就是URL，URL包括http协议名称、主机名、主机端口号以及路径。如：`http://localhost:8888/hello`。指定了URL，客户端就知道将请求通过什么协议，发送给哪个主机，与哪个端口通信，以及获取哪个页面。

Http协议需要下层提供可靠连接，通常通过TCP协议来保证可靠连接。因此，在发送http请求时，需要先与服务器建立TCP连接，TCP连接通过sockets来建立。

一个Http请求包括了请求方法（GET POST等），请求路径与http协议版本。

下面是web server的一个简单实现：

	import socket

	HOST, PORT = '', 8888

	listen_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	listen_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
	listen_socket.bind((HOST, PORT))
	listen_socket.listen(1)
	print 'Serving HTTP on port %s ...' % PORT
	while True:
    	client_connection, client_address = listen_socket.accept()
    	request = client_connection.recv(1024)
    	print request

    	http_response = """\
			HTTP/1.1 200 OK

			Hello, World!
		"""
    	client_connection.sendall(http_response)
    	client_connection.close()
    	
## WSGI

当你用许多不同的框架，或者不同的web应用时，怎么保证这些App都能与web server兼容呢？也就是说不用修改server代码来适应不同的应用。答案是WSGI。

WSGI(python web server gateway interface)是为python语言定义的web服务器与web应用或框架之间的简单而通用的接口。它使得遵从该接口实现的web server 和 web client可以相互兼容。

WSGI是怎么工作的呢？

1. web框架或应用提供一个"application"可调用对象（WSGI没有规定怎么实现它）
2. 每当收到http请求时，web server都调用"application"可调用对象。该对象的参数有两个，第一个参数是储存WSGI/CGI规定变量的字典，第二个参数是服务器定义的start_response函数，用来设置请求的响应头和server头。
3. web框架或应用生成http状态码和http响应头作为start_response的参数来调用该函数。之后返回一个response body。
4. web服务器将http状态码，response header以及response body组成http response并将其返回给客户端（WSGI没有规定这步，为了完整性说明一下）。

说完可能你还是没怎么理解， 直接看代码比较直接：

server代码

	import socket
    import StringIO
    import sys


    class WSGIServer(object):

        address_family = socket.AF_INET
        socket_type = socket.SOCK_STREAM
        request_queue_size = 1

        def __init__(self, server_address):
            # Create a listening socket
            self.listen_socket = listen_socket = socket.socket(
                self.address_family,
                self.socket_type
            )
            # Allow to reuse the same address
            listen_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
            # Bind
            listen_socket.bind(server_address)
            # Activate
            listen_socket.listen(self.request_queue_size)
            # Get server host name and port
            host, port = self.listen_socket.getsockname()[:2]
            self.server_name = socket.getfqdn(host)
            self.server_port = port
            # Return headers set by Web framework/Web application
            self.headers_set = []

        def set_app(self, application):
            self.application = application

        def serve_forever(self):
            listen_socket = self.listen_socket
            while True:
                # New client connection
                self.client_connection, client_address = listen_socket.accept()
                # Handle one request and close the client connection. Then
                # loop over to wait for another client connection
                self.handle_one_request()

        def handle_one_request(self):
            self.request_data = request_data = self.client_connection.recv(1024)
            # Print formatted request data a la 'curl -v'
            print(''.join(
                '< {line}\n'.format(line=line)
                for line in request_data.splitlines()
            ))

            self.parse_request(request_data)

            # Construct environment dictionary using request data
            env = self.get_environ()

            # It's time to call our application callable and get
            # back a result that will become HTTP response body
            result = self.application(env, self.start_response)

            # Construct a response and send it back to the client
            self.finish_response(result)

        def parse_request(self, text):
            request_line = text.splitlines()[0]
            request_line = request_line.rstrip('\r\n')
            # Break down the request line into components
            (self.request_method,  # GET
             self.path,            # /hello
             self.request_version  # HTTP/1.1
             ) = request_line.split()

        def get_environ(self):
            env = {}
            # The following code snippet does not follow PEP8 conventions
            # but it's formatted the way it is for demonstration purposes
            # to emphasize the required variables and their values
            #
            # Required WSGI variables
            env['wsgi.version']      = (1, 0)
            env['wsgi.url_scheme']   = 'http'
            env['wsgi.input']        = StringIO.StringIO(self.request_data)
            env['wsgi.errors']       = sys.stderr
            env['wsgi.multithread']  = False
            env['wsgi.multiprocess'] = False
            env['wsgi.run_once']     = False
            # Required CGI variables
            env['REQUEST_METHOD']    = self.request_method    # GET
            env['PATH_INFO']         = self.path              # /hello
            env['SERVER_NAME']       = self.server_name       # localhost
            env['SERVER_PORT']       = str(self.server_port)  # 8888
            return env

        def start_response(self, status, response_headers, exc_info=None):
            # Add necessary server headers
            server_headers = [
                ('Date', 'Tue, 31 Mar 2015 12:54:48 GMT'),
                ('Server', 'WSGIServer 0.2'),
            ]
            self.headers_set = [status, response_headers + server_headers]
            # To adhere to WSGI specification the start_response must return
            # a 'write' callable. We simplicity's sake we'll ignore that detail
            # for now.
            # return self.finish_response

        def finish_response(self, result):
            try:
                status, response_headers = self.headers_set
                response = 'HTTP/1.1 {status}\r\n'.format(status=status)
                for header in response_headers:
                    response += '{0}: {1}\r\n'.format(*header)
                response += '\r\n'
                for data in result:
                    response += data
                # Print formatted response data a la 'curl -v'
                print(''.join(
                    '> {line}\n'.format(line=line)
                    for line in response.splitlines()
                ))
                self.client_connection.sendall(response)
            finally:
                self.client_connection.close()


    SERVER_ADDRESS = (HOST, PORT) = '', 8888


    def make_server(server_address, application):
        server = WSGIServer(server_address)
        server.set_app(application)
        return server


    if __name__ == '__main__':
        if len(sys.argv) < 2:
            sys.exit('Provide a WSGI application object as module:callable')
        app_path = sys.argv[1]
        module, application = app_path.split(':')
        module = __import__(module)
        application = getattr(module, application)
        httpd = make_server(SERVER_ADDRESS, application)
        print('WSGIServer: Serving HTTP on port {port} ...\n'.format(port=PORT))
        httpd.serve_forever()
        
 web应用代码
 	
 	def app(environ, start_response):
    	"""A barebones WSGI application.

    	This is a starting point for your own Web framework :)
    	"""
    	status = '200 OK'
    	response_headers = [('Content-Type', 'text/plain')]
    	start_response(status, response_headers)
    	return ['Hello world from a simple WSGI application!\n']
    	
    	
## 处理多个请求

到目前为止的server都是单进程的，无法接受一个新的TCP连接，无法同时处理多个http requests。

### 先修知识

在讲如何实现处理多个请求前，先了解一些基础知识：

1. 什么是socket？ A socket is an abstraction of a communication endpoint and 
it allows your program to communicate with another program using file descriptors

	TCP socket pair是一个标识TCP连接的四元组，包括连接两端的ip地址和端口号，如 {10.10.10.2:49152, 12.12.12.3:8888}。
	一个socket pair唯一确定网络中的TCP连接
2. 什么是进程？ 这个就不讲了，应该都了解。
3. 什么是文件描述符（file descriptor）？ A file descriptor is a non-negative integer that the kernel returns to a process when it opens an existing file, creates a new file or when it creates a new socket.

	stdin、stdout、stderr的file descriptor分别为0、1、2
	
	"Unix万物皆文件"，socket也有一个file description，可以通过如下方式查看：
		
		>>> import socket
		>>> sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
		>>> sock.fileno()
		3
4. `listen_socket.listen(REQUEST_QUEUE_SIZE)`函数的参数是什么含义？ The BACKLOG argument determines the size of a queue within the kernel for incoming connection requests.
	
	即该参数指定kernel能够接受的连接个数，连接存放在队列中来等待server socket。所以说这个参数并不是
	server能同时处理的连接数，它的作用是当连接很多时，调用socket.accept可以直接从队列中获取connection，
	而不用等待建立连接，从而可以提高效率，类似于缓存机制。

### 多进程处理requests

我们知道，使用fork函数可以创建进程，所以多进程处理requests很easy啊，来一个连接就fork一个进程
来处理该连接。见下面部分代码：
	
	 while True:
        client_connection, client_address = listen_socket.accept()
        pid = os.fork()
        if pid == 0:  # child
            listen_socket.close()  # close child copy
            handle_request(client_connection)
            client_connection.close()
            os._exit(0)  # child exits here
        else:  # parent
            client_connection.close()  # close parent copy and loop over

#### 关闭socket和connection

看了代码可能会由两个困惑，为什么子进程要关闭socket？为什么父进程要关闭连接？

1. 当父进程fork子进程时，子进程会复制父进程的file descriptors。
2. 当调用socket.close()和connection.close()时，kernel使用descriptor引用计数来决定是否真的关闭。
3. 由于socket和connection在父子进程中都有file descriptors，因此引用计数为2，调用一次close只减1。

综上所述，调用close()是没问题的，至于为什么要调用，等会讲，现在梳理一下该代码的工作：当获取一个连接时，
父进程fork子进程来处理连接，返回请求，然后关闭连接。而父进程不处理连接，只负责不断获取连接然后fork进程
来处理它们。不同进程同步处理不同请求。（Two events are concurrent if you cannot tell by looking at the program which will happen first.）

好了，那么为什么要关闭socket和连接呢？

1. 如果不关闭socket，termination packet就不会发送给客户端，这时候客户端就会一直在线，不会关闭。
2. 不关闭连接，file descriptors就会耗尽。`ulimit -a`可以查看最大file descriptors数。
	
#### 僵尸进程

什么是僵尸进程？ A zombie is a process that has terminated, but its parent has not waited for it and has not received its termination status yet.

如果一个子进程在父进程之前结束，kernel会把子进程变成僵尸，存储进程信息（包括进程ID，进程终止状态以及进程使用的资源）
以便父进程之后获取，如果之后父进程没有wait子进程，僵尸就一直存在。

僵尸进程无法被kill，当僵尸进程数目达到最大进程数时(`ulimit -a`)，服务器就会报错（Resource temporarily unavailable） 

怎么处理僵尸进程？	一个方法是结合使用signal handler和wait系统调用。具体工作原理为：当子进程结束时，
kernel会发送SIGCHLD信号。父进程可以设置一个signal handler来异步获取SIGCHLD信号，并调用wait来获取
结束状态，因此可以防止僵尸进程产生。核心代码如下：

	def grim_reaper(signum, frame):
    	pid, status = os.wait()
    	print(
        	'Child {pid} terminated with status {status}'
        	'\n'.format(pid=pid, status=status)
    	)
    	
    def serve_forver():
    	listen_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    	listen_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    	listen_socket.bind(SERVER_ADDRESS)
    	listen_socket.listen(REQUEST_QUEUE_SIZE)
    	print('Serving HTTP on port {port} ...'.format(port=PORT))
    	signal.signal(signal.SIGCHLD, grim_reaper)
    	while True:
    		client_connection, client_address = listen_socket.accept()
    		...
    		
即通过signal.signal注册处理SIGCHLD的回调函数，回调函数中调用wait()。

但是，这样还有一个问题，如果在调用listen_socket.accept()时有SIGCHDL信号产生，这时回调函数就会
被调用，回调函数结束后accept系统调用就会报`interrupted system call`。

解决方法很简单，只要restart accept系统调用即可，代码如下：

	while True:
        try:
            client_connection, client_address = listen_socket.accept()
        except IOError as e:
            code, msg = e.args
            # restart 'accept' if it was interrupted
            if code == errno.EINTR:
                continue
            else:
                raise

别高兴太早，还有一个问题，如果同时有很多子进程结束，kernel会给父进程发出一连串的SIGCHLD信号，
而这些信号并不保存在队列中，因此在处理一个信号的过程中会丢失一些信号，就又产生了僵尸进程。

怎么办？使用WHOHANG参数的waitpid系统调用代替wait调用，并且用一个循环来保证所有的信号都被处理了。代码如下：

	def grim_reaper(signum, frame):
    	while True:
        	try:
            	pid, status = os.waitpid(
                	-1,          # Wait for any child process
                 	os.WNOHANG  # Do not block and return EWOULDBLOCK error
            	)
        	except OSError:
            	return

        	if pid == 0:  # no more zombies
            	return

Over.

##参考文档
 
[Let’s Build A Web Server](http://ruslanspivak.com/lsbaws-part3/)
{% include references.md %}
