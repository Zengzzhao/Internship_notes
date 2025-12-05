# HMR

## 编译过程

将源代码中的路径进行处理，重写模块路径（第三方依赖路径替换为预构建缓存.vite中的路径，普通路径替换为根目录的绝对路径）

**源代码**

 <img src="./Vite.assets/image-20251205115435311.png" alt="image-20251205115435311" style="zoom:50%;" />

**编译后的代码**

![image-20251205115413043](./Vite.assets/image-20251205115413043.png)

对于vue文件会将样式部分抽离，通过import方式引入

<img src="./Vite.assets/image-20251205123442649.png" alt="image-20251205123442649" style="zoom:50%;" />

## 页面刷新

### 客户端与服务端建立websocket长连接

![image-20251205123815802](./Vite.assets/image-20251205123815802.png)

<img src="./Vite.assets/image-20251205124631629.png" alt="image-20251205124631629" style="zoom:50%;" />

其中数据可以分为以下几类

**刚连接**

<img src="./Vite.assets/image-20251205124744024.png" alt="image-20251205124744024" style="zoom:50%;" />

**连接后更新模块，发生数据**

<img src="./Vite.assets/image-20251205124831427.png" alt="image-20251205124831427" style="zoom:50%;" />

<img src="./Vite.assets/image-20251205124855383.png" alt="image-20251205124855383" style="zoom:50%;" />

**没有变化，发送心跳**

<img src="./Vite.assets/image-20251205124956535.png" alt="image-20251205124956535" style="zoom:50%;" />

**更新了文件**

![image-20251205125244205](./Vite.assets/image-20251205125244205.png)

![image-20251205125259775](./Vite.assets/image-20251205125259775.png)

![image-20251205125328374](./Vite.assets/image-20251205125328374.png)

### 页面端注入clinet.js实现不同的更新逻辑

![image-20251205125659475](./Vite.assets/image-20251205125659475.png)