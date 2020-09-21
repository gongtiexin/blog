---
title: gRPC-Web 初窥
date: 2019-08-16 16:42

categories:
  - 技术

tags:
  - gRPC
---

## 什么是 [gRPC][1]

在`gRPC`中，客户端应用程序可以直接调用不同计算机上的服务器应用程序上的方法，可以更轻松地创建分布式应用程序和服务。`gRPC`基于定义服务的思想，指定可以使用其参数和返回类型远程调用的方法。在服务器端，服务器实现此接口并运行`gRPC`服务器来处理客户端调用。在客户端，它提供与服务器相同的方法

## 什么是 [Protocol Buffers][2]

`Protocol Buffers`是一种与语言无关，平台无关的可扩展机制，用于序列化结构化数据

## Web 上的实践

### 准备工作

1. 需要安装[protobuf][3]

mac 可用用 homebrew 安装，我装的版本为 3.7.1

```
brew install protobuf
```

2. 安装[protoc-gen-grpc-web][4]

```
$ sudo mv ~/Downloads/protoc-gen-grpc-web-1.0.6-darwin-x86_64 /usr/local/bin/protoc-gen-grpc-web
$ chmod +x /usr/local/bin/protoc-gen-grpc-web
```

2. 安装`grpc-web`和`google-protobuf`

```
yarn add grpc-web
yarn add google-protobuf
```

### 开始实践

1. 定义 proto 文件

```
// proto 版本
syntax = "proto3";

// 包名
package grpc;

// 定义请求参数结构
message RequestDTO {
    string id = 1;
}

// 定义返回结构
message ResponseBO {
}

// 定义请求方法
service DemoService {
    // 获取用户普通话主页
    rpc GetInfo (RequestDTO) returns (ResponseBO) {
    }
}
```

proto 文件可以找后端的同事要。这里有个坑，import 路径不能像服务端的包名一样，我就只有放在一个目录里面，直接引入文件名`import 'xxx.proto'`

2. 编译 js 和 client 文件

```
# js
protoc -I=$DIR *.proto --js_out=import_style=commonjs:$OUT_DIR
# grpc-web
protoc -I=$OUT_DIR *.proto --grpc-web_out=import_style=commonjs,mode=grpcweb:$OUT_DIR
```

`$DIR`为你 proto 文件所在路径，`$OUT_DIR`为输出路径，我用的当前文件`.` `import_style`采用的前端比较常用的`commonjs` `mode`支持文本和二进制，我用的`grpcweb`，也可以用`grpctext`

3. 编写 js 代码

```
// 引入路径根据你生成文件的路径做相应的调整
const { DemoServiceClient } = require('./proto/grpc/service_grpc_web_pb');
const { RequestDTO } = require('./proto/grpc/demo.js');

const client = new DemoServiceClient('https://local.hongkazhijia.com:3000');

const main = () => {
  const requestDTO = new RequestDTO();
  requestDTO.setId('test');

  const getInfo = client.getInfo(requestDTO, {}, (err, response) => {
    if (err) {
      console.log(err);
    } else {
      console.log(response);
    }
  });
  getInfo.on('status', status => console.log('status', status));
  getInfo.on('data', data => console.log('data', data));
  getInfo.on('end', end => console.log('end', end));
  getInfo.on('error', error => console.log('error', error));
};

export { main };
```

4. 浏览器跨域问题

开发时调用的服务端的接口和本地地址肯定会有跨域问题平时开发的时候用的 webpack-devServer 做反向代理，因为`gRPC`基于`http2`，所以在配置文件里面设置`http2 = true`(注意`http2`只支持`node < 10`)

DemoServiceClient 的地址指向本地，然后添加一个 proxy

```
// grcp.DemoService 为你的package名 + service名称，不知道的话先直接请求一次直接在console里面看就行了
      '/grcp.DemoService': {
        target: 'http://xxx.xx.xx.xx:xxxx',
        changeOrigin: true,
      },
```

至此整个流程就跑通啦，你就能在控制台看见后端返回的结果了

剩下的就是组织代码，怎么更好的集成在自己的项目中了

[1]: https://grpc.io/docs/
[2]: https://developers.google.com/protocol-buffers/docs/reference/javascript-generated
[3]: https://github.com/protocolbuffers/protobuf/releases/
[4]: https://github.com/grpc/grpc-web/releases
