### package.json

```html
{
  "name": "demo",
  "version": "1.0",
  "dependencies": {
    "socket.io-client": "^2.3.0"
  }
}
```





### Demo.js 

```javascript
import React, { Component } from 'react';
import io from 'socket.io-client';

export default class Demo extends Component {
  constructor(props) {
    super(props);

    this.state = {
      socket: null,
      connected: false,
    };
  }

  componentDidMount() {
    this.connect();

    window.addEventListener('beforeunload', this.beforeUnload);
  }

  componentWillUnmount() {
    window.removeEventListener('beforeunload', this.beforeUnload);

    this.disconnect();
  }

  beforeUnload = e => {
    this.disconnect();
    // e.preventDefault();
    // e.returnValue = '';
  };

  build = () => {
    let token = 'a0b1c2d3e4f5g6h7i8j9';
    let username = 'admin';
    let socket = io(`?username=${username}&token=${token}`);
    // 建议用 Nginx跳转/socket.io/ 取代 固定IP端口的地址

    socket.on('connect', () => {
      this.setState({ connected: true });
    });
    socket.on('disconnect', () => {
      this.setState({ connected: false });
    });
    socket.on('event', data => {
      console.log(data);
    });
    socket.on('broadcast', data => {
      // receive message from server
      data = JSON.parse(data);
    });
    this.setState({ socket: socket });
  };

  connect = () => {
    try {
      let { socket, connected } = this.state;
      if (null === socket) {
        this.build();
        // 注意：异步
        socket = this.state.socket;
      }
      
      if (!connected) {
        socket.connect();
      } else {
        socket.disconnect();
        socket.connect();
      }
    } catch (error) {
      console.log(error);
    }
  };

  disconnect = () => {
    try {
      let { socket, connected } = this.state;
      if (null != socket && connected) {
        socket.disconnect();
      }
    } catch (error) {
      console.log(error);
    }
  };

  send = () => {
    try {
      let { socket, connected } = this.state;
      if (null != socket && connected) {
        // send message to server
        socket.emit('topic', '新消息来啦~');
      }
    } catch (error) {
      console.log(error);
    }
  };

  render() {
    return <></>;
  }
}
```





### pom.xml

```html
  <dependency>
    <groupId>com.corundumstudio.socketio</groupId>
    <artifactId>netty-socketio</artifactId>
    <version>1.7.18</version>
  </dependency>
```





### application.yml

```html
socket:
  hostname: "0.0.0.0"
  port: 9010
  upgrade-timeout: 10000
  ping-interval: 25000
  ping-timeout: 60000
```





### SocketConfig.java

```java
import com.corundumstudio.socketio.Configuration;
import com.corundumstudio.socketio.SocketIOServer;
import com.corundumstudio.socketio.annotation.SpringAnnotationScanner;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;

@org.springframework.context.annotation.Configuration
public class SocketConfig {

    @Value("${socket.hostname}")
    private String hostname;

    @Value("${socket.port}")
    private Integer port;

    @Value("${socket.upgrade-timeout}")
    private Integer upgradeTimeout;

    @Value("${socket.ping-interval}")
    private Integer pingInterval;

    @Value("${socket.ping-timeout}")
    private Integer pingTimeout;

    @Bean
    public SocketIOServer socketIOServer() {
        Configuration config = new Configuration();
        // 设置主机名，默认是0.0.0.0
        config.setHostname(hostname);
        // 设置监听端口
        config.setPort(port);
        // http协议升级ws 的超时时间（毫秒），默认10000
        config.setUpgradeTimeout(upgradeTimeout);
        // 客户端向服务端发送心跳包的时间间隔，默认25000
        config.setPingInterval(pingInterval);
        // 客户端向服务端发送消息的超时时间，默认60000
        config.setPingTimeout(pingTimeout);
        // JWT的Token校验
        config.setAuthorizationListener(data -> {
            String token = data.getSingleUrlParam("token");
            if (null != token) {
                return true;
            }
            return false;
        });
        return new SocketIOServer(config);
    }

    @Bean
    public SpringAnnotationScanner springAnnotationScanner(SocketIOServer socketServer) {
        return new SpringAnnotationScanner(socketServer);
    }
}
```





### SocketHandler.java

```java
import com.corundumstudio.socketio.AckRequest;
import com.corundumstudio.socketio.SocketIOClient;
import com.corundumstudio.socketio.SocketIOServer;
import com.corundumstudio.socketio.annotation.OnConnect;
import com.corundumstudio.socketio.annotation.OnDisconnect;
import com.corundumstudio.socketio.annotation.OnEvent;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class SocketHandler {

    private final SocketIOServer server;

    @Autowired
    private SocketHandler(SocketIOServer server) {
        this.server = server;
    }

    @OnConnect
    public void onConnect(SocketIOClient client) {
        System.out.println("Socket connect ..");
        if (client != null) {
            String username = client.getHandshakeData().getSingleUrlParam("username");
            // 单用户可能多终端登陆
            client.joinRoom(username);

            String sessionId = client.getSessionId().toString();
            System.out.println("username=" + username + ", sessionId=" + sessionId);
        } else {
            System.out.println("client is empty");
        }
    }

    @OnDisconnect
    public void onDisconnect(SocketIOClient client) {
        System.out.println("Socket disconnect ..");
        System.out.println("sessionId=" + client.getSessionId().toString());
        client.disconnect();
    }

    /**
     * 前端向后台发消息
     *
     * @param client
     * @param request
     * @param message
     */
    @OnEvent(value = "topic")
    public void onEvent(SocketIOClient client, AckRequest request, String message) {
        System.out.println("Socket receive message ..");
        System.out.println("message=" + message);
        if (request.isAckRequested()) {
            request.sendAckData(message);
        }
    }

    /**
     * 后台向前端推消息
     *
     * @param username
     * @param message
     */
    public void broadcast(String username, String message) {
        System.out.println("Socket post message ..");
        System.out.println("username=" + username);
        server.getRoomOperations(username).sendEvent("broadcast", message);
    }
}
```





### SocketRunner.java 

```java
import com.corundumstudio.socketio.SocketIOServer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(1)
public class SocketRunner implements CommandLineRunner {

    private final SocketIOServer server;

    @Autowired
    public SocketRunner(SocketIOServer server) {
        this.server = server;
    }

    @Override
    public void run(String... args) {
        System.out.println("Socket start ..");
        server.start();
    }
}
```





### DemoController.java

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.concurrent.TimeUnit;

@RestController
@RequestMapping("/app/demo")
public class DemoController {

    @Autowired
    private SocketHandler socketHandler;

    synchronized public void broadcast() {
        try {
            socketHandler.broadcast("admin", "新消息啊哈哈~");
            TimeUnit.MILLISECONDS.sleep(500);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```





### 参考链接

[《SpringBoot2.0集成WebSocket》](https://blog.csdn.net/moshowgame/article/details/80275084)

[《spring-websocket-template》](https://github.com/lahsivjar/spring-websocket-template)

[《SpringBoot WebSocket四种实现方式》](https://www.cnblogs.com/kiwifly/p/11729304.html)