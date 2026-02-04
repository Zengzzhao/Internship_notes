dto：数据验证层，分析前端传递过来的数据

```bash
# 查看nest所有命令
nest g --help
# 创建一个crud的资源模版
nest g res
# 创建一个controller
nest g co
# 创建一个module
nest g mo
# 创建一个拦截器
nest g itc 拦截器名
# 创建一个全局过滤器
nest g f 全局过滤器名
```



### IOC控制反转

IOC控制反转是一种设计思想，把对象的创建和依赖关系交给容器管理，而不是手动创建和管理。

- 传统方式：你需要什么手动创建`new A()`，`new B()`
- IOC方式：你需要什么，容器给你送过来


### DI依赖注入

IOC控制反转是一种设计思想，而DI依赖注入是控制反转的一种代码实现方式。

### 装饰器

装饰器的本质其实就是一种特殊的函数，他们会在编译时被解析，并生成元数据，这些元数据可以在运行时被访问。

他有什么好处：

主要目的就是在不改变原有代码的情况下，通过装饰器来实现对代码的扩展。

装饰器一共有5种：

- 类装饰器
- 属性装饰器
- 方法装饰器
- 参数装饰器
- 访问器装饰器

```typescript
@injectable()
class GoodsService {
   @Inject()
   private body: any
   constructor(private userService: UserService) {}
   getGoodsInfo(@Body() body: any) {
    console.log(this.userService.getUserInfo())
    return 'goodsInfo'
   }
   
   @Body()
   getBody() {
    return this.body
   }

   @Getter()
   get name() {

   }
}
```

### Reflect.getMetadata('design:paramtypes', clazz)

typeScript 自动注入的元数据

- design:paramtypes 获取构造函数的参数类型
- design:returntype 获取构造函数的返回类型
- design:type 获取构造函数的类型

条件必须满足该类被装饰器装饰了。

元数据

```bash
npm install reflect-metadata
```

```json
   "experimentalDecorators": true, // 启用装饰器语法              
    "emitDecoratorMetadata": true, // 启用装饰器元数据   启用完成之后typeScript会自动注入例如 design:paramtypes 等元数据 就可以使用Reflect.getMetadata('design:paramtypes', clazz)来获取元数据
```

