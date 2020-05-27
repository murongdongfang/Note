# 同步异步，阻塞非阻塞的联系和区别

**概念**同步A调用B，**B处理直到获得结果**，才返回给A。需要调用者一直等待和确认调用结果是否返回，然后继续往下执行。异步A调用B，**B无需等待结果，B通过状态通知A或回调函数来处理。调用结果返回时，会以消息或回调的方式通知调用者。**阻塞A调用B，A被挂起直到B返回结果给A，才能继续执行。调用结果返回前，当前线程挂起不能够处理其他任务，一直等待调用结果返回。非阻塞A调用B，A不会被挂起，A可以执行其他操作。调用结果返回前，当前线程不挂起，可以处理其他任务。**二、两者区别**同步异步是个操作方式，阻塞非阻塞是线程的一种状态。同步异步指的是被调用者结果返回时通知线程的一种机制，阻塞非阻塞指的是调用结果返回进程前的状态，是挂起还是继续处理其他任务。



非阻塞和异步的概念辨析：非阻塞只是意味着方法调用不阻塞，就是说作为服务员的你不用一直在窗口等，非阻塞的逻辑是"等可以读（写）了告诉你"，但是完成读（写）工作的还是调用者（线程）服务员的你等菜到窗口了还是要你亲自去拿。而异步意味这你可以不用亲自去做读（写）这件事，你的工作让别人（别的线程）来做，你只需要发起调用，别人把工作做完以后，或许再通知你，它的逻辑是“我做完了 告诉/不告诉 你”，他和非阻塞的区别在于一个是"已经做完"另一个是"可以去做"。







# 基于BIO实现网络通信

![](https://gitee.com/little_broken_child_9527/images/raw/master/20200517225910.png)

## 客户端

```java
public class Client {
    private static final String DEFAULT_HOST = "127.0.0.1";
    private static final int DEFAULT_PORT = 9999;
    private static Socket socket = null;
    private static final String QUIT = "quit";
    private static BufferedReader scanner = null;

    public static void main(String[] args) {
        try {
            socket = new Socket(DEFAULT_HOST, DEFAULT_PORT);
             scanner = new BufferedReader(new InputStreamReader(System.in));

            BufferedReader reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
            while(true){
                String msg = scanner.readLine();
                if("quit".equals(msg)){
                    System.out.println("关闭客户端");
                    break;
                }

                writer.write(msg+"\n");
                //必须要刷新，否则Socket会一直陷入阻塞，直到缓冲区存满
                writer.flush();

                String serverMsg = reader.readLine();
                System.out.println("服务器返回消息："+serverMsg);

            }

        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if(socket != null){
                try {
                    socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
            if (scanner != null){
                try {
                    scanner.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }

    }
}

```







## 服务端

```java

public class Server {
    private static final int DEFAULT_PORT = 9999;
    private static Socket socket = null;
    private static ServerSocket serverSocket = null;
    private static final String QUIT = "stop";

    public static void main(String[] args) {
        try {
             serverSocket = new ServerSocket(DEFAULT_PORT);
            while(true){
                //serverSocket.accept()为阻塞性调用，需要不断地轮询看是否有响应
                socket = serverSocket.accept();
                System.out.println("服务器已经连接客户端："+socket.getPort());
                BufferedReader reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));
                BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
                String clientMsg = null;
                while((clientMsg = reader.readLine()) != null){

                    System.out.println("客户端消息："+clientMsg);
                    writer.write(clientMsg+"\n");
                    //必须要刷新，否则Socket会一直陷入阻塞，直到缓冲区存满
                    writer.flush();
                    if(QUIT.equals(clientMsg)){
                        System.out.println("客户端退出,服务器关闭");
                        break;
                    }
                }
            }
        }catch (Exception e){

        }finally {
            if(serverSocket != null){
                try {
                    serverSocket.close();
                    System.out.println("关闭ServerSocket");
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

        }
    }

}

```



# NIO

![](https://gitee.com/little_broken_child_9527/images/raw/master/20200517225911.png)





# IO多路复用

![](https://gitee.com/little_broken_child_9527/images/raw/master/20200517225912.png)



# AIO

![](https://gitee.com/little_broken_child_9527/images/raw/master/20200517225913.png)

