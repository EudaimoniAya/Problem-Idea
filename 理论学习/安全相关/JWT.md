### 一、基本概念

JWT(JSON Web Token)，是一种让JSON对象安全传输的开放标准，通常用作身份验证和授权。它有三部分组成：头部(Header)、载荷(Payload)和签名(Signature)，格式一般为：`xxxxxx.yyyyyy.zzzzzz`。
* 头部说明了这是JWT和签名使用的算法
* 载荷携带了如用户ID、过期时间等实际数据；
* 签名是防伪标志，用`secret_key`生成，验证令牌是否被篡改。令牌存在两种：用于访问API的短期令牌`ACCESS_TOKEN`和用于更新短期令牌的长期令牌`REFRESH_TOKEN`。（**Q：** 为什么会需要两个Token？ **A：**`ACCESS_TOKEN`如果泄露，那么它就能够访问API，但是有效期短且无法获取新Token；`REFRESH_TOKEN`如果泄露，虽然可以获取新的短期令牌，但需要知道`secret_key`才能签名，而且通常它会有更严格的保护措施，还有使用次数限制和时间限制。）
在HTTP请求报文报文中，请求头的`Authorization`字段可以选择`Bearer {token}`来实现认证，Bearer是HTTP授权方案的一种类型，它的工作逻辑就是只要持有Token就可以访问对应的资源。FastAPI框架通过`HTTPBearer`提供了Bearer的功能，它可以自动从HTTP请求报文的请求头中获取到`Authorization`的token，避免了以往的手动获取、手动解析。

### 二、Token生成
初步优化后的`AINameProject`中的生成逻辑：
```python
# 令牌最好都使用字符串
class TokenTypeEnum(Enum):
    ACCESS_TOKEN = "access_token"    # 短期令牌，用于访问API
    REFRESH_TOKEN = "refresh_token"  # 长期令牌，用于更新ACCESS_TOKEN
    
class AuthHandler(metaclass=SingletonMeta):  
	# security会自启动，检查请求头中是否有Authorization字段，其格式是否是
	# Authorization: Bearer {token}，并自动提取token，如果格式错误返回401错误 
	security = HTTPBearer()  
	secret = settings.jwt_secret_key
	
	def __init__(self):  
    self.secret = settings.jwt_secret_key  
    self.algorithm = "HS256"
	
	def _encode_token(self, user_id: int, token_type: TokenTypeEnum):
	    # 1. 获取当前时间（使用UTC标准时间）
	    now = datetime.utcnow()
	    
	    # 2. 根据Token类型设置不同的有效期
	    if token_type == TokenTypeEnum.ACCESS_TOKEN:
	        expires_delta = settings.jwt_access_token_expires  # 15天
	    else:
	        expires_delta = settings.jwt_refresh_token_expires  # 30天
	    
	    # 3. 计算过期时间
	    expire = now + expires_delta
	    
	    # 4. 构建荷载（payload） - 这是Token的核心数据
	    payload = {
	        "iss": str(user_id),           # 签发者Issuer（谁发的令牌）
	        "sub": token_type.value,       # 主题Subject（Token类型）
	        "iat": int(now.timestamp()),   # 签发时间（issued at）
	        "exp": int(expire.timestamp()),# 过期时间（expiration）
	    }
	    
	    # 5. 用HS256算法签名，生成完整的Token
	    return jwt.encode(payload, self.secret, self.algorithm)
```
其中`iss`、`sub`、`iat`、`exp`、`type`这些信息都是载荷payload中存在的字段，有些字段可以不填，如`type`。另外`sub`最好是字符串类型，否则可能出错。最后生成Token的在于方法`jwt.encode()`它会根据配置中设置的密钥`self.secret`和指定的加密算法`self.algorithm`对数据payload进行编码。

### 三、Token解码与验证
Token可能不合法、可能过期、而且需要识别类型，所以服务器同样也需要将自己编码的Token信息用对应方法解码。
```python
def decode_access_token(self, token: str):  
    """解码并验证 Access Token"""    
    try:  
	    # 1. 解码并验证签名
        payload = jwt.decode(  
            token,  
            self.secret,  
            algorithms=[self.algorithm],
            # 前三个同编码，option的require要求载荷必须包含的字段  
            options={"require": ["exp", "iat", "iss", "sub"]}  
        )  
  
        # 验证 Token 类型是不是access_token（不是refresh_token或其他）  
        if payload.get("sub") != TokenTypeEnum.ACCESS_TOKEN.value:  
            raise HTTPException(  
                status_code=HTTP_403_FORBIDDEN,  
                detail="Invalid token type: not an access token"  
            )  
  
        # 返回用户ID，逻辑仅是规定id大于0
        try:  
            user_id = int(payload.get("iss", 0))  
            if user_id <= 0:  
                raise HTTPException(  
                    status_code=HTTP_403_FORBIDDEN,  
                    detail="Invalid user ID"  
                )  
            return user_id  
        except (ValueError, TypeError):  
            raise HTTPException(  
                status_code=HTTP_403_FORBIDDEN,  
                detail="Invalid user ID format"  
            )  
  
    except jwt.ExpiredSignatureError:  
        raise HTTPException(  
            status_code=HTTP_403_FORBIDDEN,  
            detail="Access token has expired"  
        )  
    except jwt.InvalidTokenError as e:  
        raise HTTPException(  
            status_code=HTTP_403_FORBIDDEN,  
            detail=f"Invalid access token: {str(e)}"  
        )
```

### 四、依赖注入
```python
def auth_access_dependency(
	self, 
	auth: HTTPAuthorizationCredentials = Depends(security)
):  
    """Access Token 验证依赖"""  
    return self.decode_access_token(auth.credentials)
```
其中`HTTPAuthorizationCredentials`是Pydantic模型，它由`HTTPBearer`安全返回，而`HTTPBearer`则用于从请求头中获取Bearer Token。对于一个HTTP请求头的鉴权部分：
```HTTP
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```
`HTTPBearer`处理请求后（FastAPI框架自动从请求头中获取，取不到则报错），会返回一个对象，类型就是`HTTPAuthorizationCredential`：
```python
auth = HTTPAuthorizationCredentials(
    scheme="Bearer",           # 授权方案类型
    credentials="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."  # 令牌
)
```
所以`auth.credentials`就是请求体中的Token。通过依赖注入，让视图函数只需声明鉴权功能就能让框架自动执行验证相关逻辑，实现验证逻辑与业务逻辑分离。

