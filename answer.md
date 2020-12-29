# PseudoEncrypt算法(java)
```
public class PseudoEncrypt {
    public int pseudoEncrypt(int value) {
        int l1 = (value >> 16) & 65535;
        int r1 = value & 65535;
        for (int i = 0; i < 3; i++) {
            int l2 = r1;
            int r2 = l1 ^ (int)((((1366 * r1 + 150889) % 714025) / 714025.0) * 32767);
            l1 = l2;
            r1 = r2;
        }
        return ((r1 << 16) + l1);
    }
}
```

# 注册登录功能协议(proto)
```
syntax = "proto3";

package mobi.joycastle.question;

service SignInService {
  rpc signUpSmsVerify (signUpSmsVerifyReq) returns (signUpSmsVerifyRes); // 获取注册短信验证码
  rpc signUp (SignUpReq) returns (SignUpRes); // 注册
  rpc signIn (signInReq) returns (signInRes); // 登录
}

message commonResHeader {
  int32 statusCode = 1;
  string message = 2;
}

message signUpSmsVerifyReq {
  string phoneNumber = 1;
}

message signUpSmsVerifyRes {
  commonResHeader header = 1;
}

message SignUpReq {
  string phoneNumber = 1;
  string password = 2;
  string verificationCode = 3;
}

message SignUpRes {
  commonResHeader header = 1;
  string username = 2;
}

message signInReq {
  string username = 1;
  string password = 2;
}

message signInRes {
  commonResHeader header = 1;
}
```

# 注册与登录功能设计方案
1. 使用redis的INCR指令，为每一个新注册用户获取一个递增的序列号，然后使用PseudoEncrypt算法，将这个序列号变换成无规律的整数
2. 使用mysql来存储username，phoneNumber，password(密码将在前端转换成密文密码)等个人基本信息
3. 使用redis的map来存储<phoneNumber, verificationCode>，并设置过期时间
4. 请求将首先经过反向代理服务器，再转发到SingInService服务集群。可设置多个反向代理服务器保证多可用
5. 服务在接收获取验证码的请求后，会随机生成一个验证码，然后将<phoneNumber, verificationCode>写入redis，将需要发送的短信的相关信息写入消息队列，然后给client发送响应
6. 服务在接收到注册请求后，会判断redis中是否有<phoneNumber, verificationCode>，并删除对应条目。如果有，则从redis中拿取序列号，并使用PseudoEncrypt加密，然后将用户信息写入mysql并返回正确。如果没有则返回错误
7. 因为服务本身没有状态，所以可以进行弹性部署。同时mysql与redis也可以采用合适的高可用方案，如主从等。

# 三段用户名
设三个词库分别为A, B, C，用户的序列号为n，生成用户名的伪代码如下：
```
A = [...]
B = [...]
C = [...]
max_n = A.length * B.length * C.length - 1
# 可将序列号n映射为[0, max_n]之间的无规律的数，再映射为三个数字，从而得到无规律且不重复的三元组，然后根据该三元组获取用户名
def generate_username(n):
    if (n < 0 || n > max_n):
        raise Error('n不可小于0，不可大于max_n')
    n = bounded_pseudo_encrypt(n, max_n) # 变换n，使其被映射到[0, max_n]区间
    a = n % len(A)
    n /= len(A)
    b = n % B.length
    n /= len(B)
    return f"{A[a]} {B[b]} from {C[n]}"
``` 

# 圈地策略
骑士应该跑一个半圆的弧线，理由如下：  
设曲线a为骑士跑的曲线，将a沿起点与终点的连线做对称反转，获得曲线a'。  
则a与a'构成了一个封闭图形，将其命名为b。  
b的长度为a的长度的两倍，面积也为骑士所获的两倍。当b取最大值时，骑士所获面积也最大。  
因b的长度固定为a长度的两倍，由等周定理可得，b为圆形时，其面积最大。  
而a为b曲线的一半，所以a是半圆形。  
