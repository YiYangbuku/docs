## Token自鉴权的方案设计与验证

### 引子

最近在项目上或者售前中接触了一些比较特殊的权限管理场景，它们有的需要**支持上亿量级的认证鉴权**， 有的**鉴权网络请求延迟特别高**（数据中心在其它洲），于是我在JWT（json web token）的基础上设计了一个**自鉴权**的方案，并进行了[demo](https://github.com/YiYangbuku/self-authorization-jwt)验证。

首先需要声明的是解决上述问题的途径肯定不是只有重新设计方案这一条，诸如缓存、数据异地备份、横向扩展等都有可能解决问题。本文中的方案实际上是牺牲**空间、复杂性**来换取**时间**。

该方案是基于业务服务与认证服务分离的场景下。

### 什么是自鉴权

可以通过类比解释**自认证**来解释**自鉴权**。

在认证结构中，外部请求一般会在网关处进行拦截，未认证的请求会被重定向到认证服务，认证一般有两种方式：

* 基于session的方式：

  ![Session auth](/Users/yiyang/Downloads/Session auth.png)

  1. 第一次请求（未登录）：网关重定向到认证服务，用户登录后，认证服务将会缓存这次session（会话）的信息（包括用户的个人信息，角色权限等），并将sessionId保存到用户的浏览器cookie中。
  2. 后续请求（已登录）：网关从请求中读取sessionId，**然后与认证中心通信**获取用户信息。

* 基于JWT的方式：

  ![Token auth](/Users/yiyang/Downloads/Token auth.png)

  1. 第一次请求（未登录）：网关重定向到认证服务，用户登录后，认证服务将用户必要的身份信息用自己的秘钥进行签名（防伪造）并生成换一个token（包含用户信息、签名算法、有效期与签名等），并将token保存的用户的浏览器cookie中。
  2. 后续请求（已登录）：网关拦截请求后，用公钥（提前配置到网关中）对token进行验证，**不需要去认证中心通信**确认token是否有效。

在上述的两种认证方式中，最大的不同在于登录后，网关是否还需要再与认证服务器通信获取用户的登录信息，它的好处在于：

* 认证中心不需要缓存会话信息，这些信息都在token中
* 网关和认证中心在后续请求中少了一次网络请求，比较适用于高并发，或者网关与认证中心延迟较大的场景

所谓**自认证**指的就是在JWT这种方式中，token中包含了必要的信息外还对其进行了签名，来保证token的不可为伪造性。

相比于**认证**，**鉴权**由于包含的信息比较多，无法直接将所有鉴权信息放入到token中去，**所以即便用户已经登录，在后续的访问中仍然需要与认证中心进行通信确认权限信息**，如下图：

![Token auth with permission](/Users/yiyang/Downloads/Token auth with permission.png)

在本文中，会设计一套的**自鉴权**的方案，与**自认证**类似，目的是在第一次登录后使token包含足够的信息，后续请求不在需要与认证中心通信，设计思路是：

1. **将所有鉴权信息保存到token中**
2. **将token的长度控制在可以接受的范围内**

### 自鉴权设计步骤

分为几步，先是将鉴权的信息直接放到token中，然后缩小token的长度。

以API鉴权举例，场景4000个API（/api2/business/product/id{n}），一个用户拥有其中1/4的API的权限(/api/business/product/id{n/4})

#### 1. 直接将拥有权限的API放到token中

返回的token大小为：41.5KB

![image-20200428165634876](/Users/yiyang/Library/Application Support/typora-user-images/image-20200428165634876.png)

对token解码后包含的数据为：
![image-20200428155208676](/Users/yiyang/Library/Application Support/typora-user-images/image-20200428155208676.png)

token的长度与url的平均长度成正比，一般正常的系统中url的长度可能更长，于是token可能会更大。

token的长度也与用户有权限访问的API数量成正比，admin用户的token长度会进一步膨胀。

虽然JWT并没有限制token的长度，但是token会在每次请求中通过网络传输，太长的token会对网络请求速度造成影响。而一个正常的token长度一般都在1KB左右。

#### 2. 设计字典

用户对每个API权限是否具有权限其实是一个布尔值，true或者false，1或者0。

而计算机中的数据其实都是二进制进行存储的，本质上就是一串**101010**，那我们就可以用一个二进制串来代表用户的API权限，1代表有权限，0代表没有权限，只需要让token的生成与验证方保存同一个API字典，用来维护不同位置对应的是哪个API。

| API                        | 用户权限    |
| :------------------------- | :---------- |
| /api2/business/product/id0 | 1（有权限） |
| /api2/business/product/id1 | 1（有权限） |
| /api2/business/product/id2 | 0（无权限） |
| /api2/business/product/id3 | 0（无权限） |
| /api2/business/product/id4 | 1（有权限） |

这时候token会变成如下这个样子：

![image-20200429095659282](/Users/yiyang/Library/Application Support/typora-user-images/image-20200429095659282.png)

decode后，API的权限部分其实就是一串10101010（由于需要序列化，二进制串已被BASE64）：

![image-20200428161713046](/Users/yiyang/Library/Application Support/typora-user-images/image-20200428161713046.png)

可以看到token的长度仅仅只有1.19KB，通过token的长度与url的长度无关（每个API都只占一个二进制位的长度）

token的长度与用户具有多少权限无关，只与字典长度有关，当字典为4000时token为1.19KB；字典为8000时，长度为2.06KB，已经能够满足大部分非API密集型场景。

**从10101串与内存中的字典匹配计算出用户API权限的时间在1ms左右**

#### 3. 字典同步

由于API的数量并不是一成不变的，用户的权限也是有可能实时修改的，这部分的修改一般发生在认证中心。

而网关对token的验证也是基于同一个字典，于是字典必须进行同步。

这部分没有在demo中实现，主要的思路如下：

* 字典通过同步（比如统一缓存）或者异步（比如消息队列异步通知）进行同步。
* 字典进行版本管理：签发的token中添加其基于字典的版本
* 字典的修改可以兼容已颁发token
  1. 添加API——通过append的方式添加到字典最后，不影响已有API在字典中的顺序
  2. 删除API——假删除，在字典中添加一列“是否删除”，删除只改变标志位，于是不影响API字典的顺序
  3. 更新API——假删除旧API并append新API。
* 当字典中的被删除的API达到一定比例后，进行清理（将假删除变为真删除），清理操作不能兼容已经颁发token，需要重新申请。

### 与scope相比

在一些认证机制中，用户可以申请属于某个具体scope（例如read-write、public-protected-private、prod-test）下的token，以获取某些受限资源的访问权限。如果这些scope是直接配置在客户端的resource或者method上的话，scope鉴权同样也不需要进行通信。这种scope机制可以提供一种更粗粒度的鉴权，和本文的方案相比：

* scope适用于可以进行预划分的资源，如读接口/写接口、公开信息/私有信息等，一般划分的维度不会太多。
* scope需要对资源所属的scope进行预定义。
* 客户端配置的scope无法实时修改。

### 总结

该方案其实是牺牲了空间（保存字典），复杂性（字典同步）来获取更高效的时间（少了一次与认证中心的网络请求）。

Demo: [点我](https://github.com/YiYangbuku/self-authorization-jwt)