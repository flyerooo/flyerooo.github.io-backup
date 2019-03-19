+++
date = "2018-10-30T00:00:00+08:00"
tag = ""
title = "Python 使用socket模拟http请求，从阻塞到协程"
undefined = ""

+++
阻塞式

    import socket
    from urllib.parse import urlparse
    
    
    def get_url(url):
        url = urlparse(url)
        host = url.netloc
        path = url.path
        if path == "":
            path = "/"
    
        client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        client.connect((host, 80))
        # 模拟http协议
        client.send("GET {} HTTP/1.1\r\nHost:{}\r\nConnection:close\r\n\r\n".format(path, host).encode('utf8'))
        data = b''
        while True:
            d = client.recv(1024)
            if d:
                data += d
            else:
                break
        data = data.decode('utf8')
        html_data = data.split("\r\n\r\n")[1]  # 去掉请求头
        print(html_data)
        client.close()
    
    if __name__=="__main__":
        get_url("http://www.baidu.com")
    复制代码

非阻塞 因为要询问连接是否建立好，需要while循环不停的检查状态，多余消耗了CPU

    import socket
    from urllib.parse import urlparse
    
    
    def get_url(url):
        url = urlparse(url)
        host = url.netloc
        path = url.path
        if path == '':
            path = '/'
    
        client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        client.setblocking(False)  # 设置为非阻塞
    
        try:
            client.connect((host, 80))
        except BlockingIOError as e:
            pass
    
        while True:
            try:
                client.send(
                    'GET {path} HTTP/1.1\r\nHost:{host}\r\nConnection:close\r\n\r\n'.format(path=path, host=host).encode(
                        'utf8'))
                break
            except OSError as e:
                pass
    
        data = b''
        while True:
            try:
                d = client.recv(1024)
            except BlockingIOError as e:
                continue
    
            if d:
                data += d
            else:
                break
    
        data = data.decode('utf8')
        html_data = data.split('\r\n\r\n')[1]
        print(html_data)
        client.close()
    
    
    if __name__ == '__main__':
        get_url('http://www.baidu.com')
    
    复制代码

select(poll/epoll) + 回调 + 事件循环 看起来比较复杂，为什么要改成这样呢，因为只会处理那些准备好的socket，不会等待网络I/O，使用单线程模式，省去了线程间切换的开销。实现了单线程并发，并发性高 但这种回调的写法实在是太蛋疼

    import socket
    from urllib.parse import urlparse
    # 是select更易用的一个封装，会根据平台 win/linux 去自动选择select/epull
    from selectors import DefaultSelector, EVENT_READ, EVENT_WRITE
    
    selector = DefaultSelector()
    
    urls = ['http://www.baidu.com']
    stop = False
    class Fetch:
        def connected(self, key):
            selector.unregister(key.fd) # 注销监控的事件
            self.client.send('GET {path} HTTP/1.1\r\nHost:{host}\r\nConnection:close\r\n\r\n'.format(path=self.path, host=self.host).encode(
                        'utf8'))
            selector.register(self.client.fileno(), EVENT_READ,self.readable)
    
        def readable(self, key):
            d = self.client.recv(1024)
            if d:
                self.data += d
            else:
                selector.unregister(key.fd)
    
            data = self.data.decode('utf8')
            html_data = data.split('\r\n\r\n')[1]
            print(html_data)
            self.client.close()
            urls.remove(self.spider_url)
            if not urls:
                global stop
                stop = True
    
    
        def get_url(self, url):
            self.spider_url = url
            url = urlparse(url)
            self.host = url.netloc
            self.path = url.path
            self.data = b''
            if self.path == '':
                self.path = '/'
    
            self.client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            self.client.setblocking(False)
    
            try:
                self.client.connect((self.host, 80))
            except BlockingIOError as e:
                pass
    
            # 注册
            selector.register(self.client.fileno(), EVENT_WRITE, self.connected)
    
    def loop():
        # 事件循环，不停的请求socket的状态并调用对应的回调函数
        # 1. select 本身是不支持register模式
        # 2. socket撞田变化以后的回调式是由程序员完成的
        while not stop:
            ready = selector.select()
            for key, mask in ready:
                call_back = key.data
                call_back(key)
    
    
    
    if __name__ == '__main__':
        fetcher = Fetch()
        fetcher.get_url('http://www.baidu.com')
        loop()
    复制代码

可以说： 同步模式并发性不高， 回调模式编码复杂度高， 多线程需要线程间同步，影响并发性能。