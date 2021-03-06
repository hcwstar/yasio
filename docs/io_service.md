# io_service
## new
io_service io_service.new(count)  
io_service io_service.new(hostent)  
io_service io_service.new(hostents)  
重载版本参数说明:  
count: 信道数量  
  
hostent: 信道地址，格式:  
```lua
{host=ip138.com, port=80}
```
  
hostents: 信道地址列表， 格式：  
```lua
{
    {host="ip138.com", port=80},
    {host="baidu.com", port=80}
}
```

## start_service
void start_service(cb)  
功能：启动网络服务  
参数说明:  
**cb**: 网络事件回调函数, 示例:
```lua
function(event)
  local kind = event:kind()
end
```
调用示例:
```lua
local service = io_service.new({host="ip138.com", port=80})
service:start_service(function(event)
        local kind = event:kind()
        if(kind == yasio.YEK_CONNECT_RESPONSE) then
            if(event:status() == 0) then
                print('[%d] connect succeed.', event:cindex())
            else
                print('[%d] connect failed!', event:cindex())
            end
        end
    end)
```
注意: start_service只是启动服务，不会发起连接，只有调用open打开信道时才会发起连接

## stop_service
void stop_service()  
功能：停止网络服务

## set_option
set_option(opt,...)
功能：设置选项
参数说明(超时时间单位均为秒):  
### 说明: 所有选项均可在start_service之前设置
opt: 选项枚举，取值有:
  + yasio.YOPT_S_CONNECT_TIMEOUT: TCP连接超时时间(10s)
  + yasio.YOPT_S_DNS_CACHE_TIMEOUT: dns缓存时间(600s)
  + yasio.YOPT_S_DNS_QUERIES_TIMEOUT: dns域名解析超时时间(10s), 仅当c-ares启用时有效
  + yasio.YOPT_S_TCP_KEEPALIVE: 设置tcp协议keepalive参数，例如```tcpclient:set_option(yasio.YOPT_S_TCP_KEEPALIVE, 60, 10, 3)```, 60是发送底层协议心跳的空闲等待时间，10是发送心跳间隔, 3是未收到对方回应心跳重试次数
  + yasio.YOPT_S_PRINT_FN: 设置网络日志自定义打印函数，注意函数必须是线程安全的
  + yasio.YOPT_S_EVENT_CB: 设置网络事件回调
  + yasio.YOPT_C_LOCAL_PORT: 设置信道本地端口, 例如```tcpclient:set_option(yasio.YOPT_C_LOCAL_PORT, 0, 20191)```, 0是信道索引, 20191是端口
  + yasio.YOPT_C_REMOTE_HOST: 设置信道远程域名/ip, 参数顺序为: 信道索引, 远程域名/ip
  + yasio.YOPT_C_REMOTE_PORT: 设置信道远程端口, 参数顺序为: 信道索引, 端口
  + yasio.YOPT_C_REMOTE_ENDPOINT: 设置信道远程域名/ip和端口, 参数顺序为: 信道索引, 远程域名/ip, 端口
  + yasio.YOPT_C_LFBFD_PARAMS: 设置Length Field Based Frame Decode参数, 例如:
```lua
-- 对于有包长度字段的协议，对于tcp自定义二进制协议，强烈建议设计包长度字段，并设置此选项，业务无须关心粘包问题
tcpclient:set_option(yasio.YOPT_C_LFBFD_PARAMS, 
    0, -- channelIndex, 信道索引
    65535, -- maxFrameLength, 最大包长度
    0,  -- lenghtFieldOffset, 长度字段偏移，相对于包起始字节
    4, -- lengthFieldLength, 长度字段大小，支持1字节，2字节，3字节，4字节
    0 -- lengthAdjustment：如果长度字段字节大小包含包头，则为0， 否则，这里=包头大小
)
-- 对于没有包长度字段设计的协议，例如http， 设置包长度字段为-1， 那么底层服务收到多少字节就会传回给上层多少字节
httpclient:set_option(yasio.YOPT_C_LFBFD_PARAMS, 0, 65535, -1, 0, 0)
```

## open
void open(cindex,cmask)  
功能：打开信道  
参数说明:  
cindex: 信道索引，start_service传入的列表  
ckind: 信道类型，只是如下值  
  + yasio.YCK_TCP_CLIENT: 将信道打开为tcp客户端，将会发起远端服务器的tcp连接
  + yasio.YCK_TCP_SERVER: 将信道打开为tcp服务端, 启动本地tcp服务器
  + yasio.YCK_UDP_CLIENT: 将信道打开为udp客户端, 和和远端udp建立绑定, 以便进行udp通讯
  + yasio.YCK_UDP_SERVER: 将信道打开为udp服务端, windows平台不支持此类型
  + yasio.YCK_KCP_CLIENT: 将信道打开为kcp客户端, 和和远端kcp建立绑定, 以便进行基于udp的kcp通讯, 仅当C++编译宏YASIO_HAVE_KCP开启时才能用
  + yasio.YCK_KCP_SERVER: 将信道打开为kcp服务端, 仅当C++编译宏YASIO_HAVE_KCP开启时才能用, windows平台不支持此类型

## close
void close(cindex)  
void close(transport)  
功能：关闭信道或会话, 断开tcp连接或关闭udp映射
 
## write
void write(transport, data)  
功能：向指定会话发送数据  
参数说明:  
transport: 传输会话句柄  
data: 要发送的数据, 对于Lua可以是obstream或者string类型； 对于js可以是string,ArrayBuffer,Uint8Array,Int8Array

## dispatch
void dispatch(max_events)  
功能: 分派网络事件  
参数说明:  
max_events: 每次调用从事件队列分派最大事件数，通常固定为64,128,256就足够了  
  
备注: 对于游戏引擎，通常使用方法是在渲染线程每帧调用此函数来实现网络事件分派回调发生在渲染线程，以便直接根据业务逻辑更新游戏界面

## Design Patterns
+ 单利模式: 在C++中使用框架预定义宏yasio_shared_service()即可， 这种方式很方便给C#导出接口调用
+ 非单利模式: 就像普通对象一样实例化后即可使用
