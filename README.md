邮件找回密码的一些思考
===========================

忘记密码是很多用户经常遇到的问题，通过注册邮件找回密码，目前用的比较多，在设计这个功能的时候需要考虑到如下的流程

### 点击忘记密码的按钮
点击忘记密码以后，跳转到填写注册邮件的页面，需要用户填写注册邮箱
### 根据邮箱地址发送邮件
经过一系列合法性的邮件地址检测以后，后台构造url来发送重置密码的链接

### 最核心的内容，如何构造重置密码的url
这里是最核心的东西，我们的需求是这样子的:

- 我们需要发送一些重要的信息，这里是重置密码的url到我们不信任的环境中(这里是用户输入的邮件地址)。但是如果保证数据的安全和完整性？这里引入签名的概念，我们可以把要发送的数据，通过我们自己的签名加密，然后发送给别人。当别人把我们发送给他的数据返还发送给我们的时候，我们通过检查签名，可以保证我们发送的数据没有被篡改。
- 这是因为虽然数据的接收者可以修改我们发送的内容，但是他们不能修改数据内容的签名除非他知道我们签名用的私钥。因此，保护好你的签名私钥，你的数据将会很安全。同时，我们也强烈的建议你的私钥应该足够复杂以避免碰撞攻击。
- 我们可以使用HMAC和SHA1算法来用于签名，这两种加密的方式都是非对称性的。并且我们可以参考类似于JWT的方式，来存放一些我们需要的数据在里面。

如果要自己写一个这样方法，只要理解了原理以后其实也不难。但是万能的python早就有了一个这样的库，并且支持非常多的可选项来满足你的多样性需求。那就是 [itsdangerous](https://github.com/pallets/itsdangerous)

### url的过期时间
url设置过期时间是很有必要的，我们可以在我们的数据中加上一个过期时间，然后解析数据的时候根据当前时间来判断该url的时效性

### 检查url的合法性和过期性
后台检测请求的url包含的token，通过检测签名是否一致，是否过期来判断该url的有效性，如果有效，则抛出一个重置密码的页面，如果无效，则返回一个无效的提示。

### 重置用户密码


#一些示例


### 忘记密码的链接
在登录页面添加一个忘记密码的链接
![忘记密码](https://github.com/page1990/email_reset_password/blob/master/imgs/forget_password_link.png)

### 输入注册邮箱地址
![输入邮箱地址](https://ws1.sinaimg.cn/large/005B3DIrgy1fke9p2nhwej309u06mgll.jpg)

邮件地址有错误
![输入邮箱报错](https://ws1.sinaimg.cn/large/005B3DIrgy1fke9qjprhyj30a70913ym.jpg)

邮件地址OK
![发送邮件成功](https://ws1.sinaimg.cn/large/005B3DIrgy1fke9rvq48ij30a508mt8t.jpg)

### 后台构造重置url
```
from itsdangerous import TimedJSONWebSignatureSerializer as Serializer

def make_serialier_url(id, expiration=1200):
    """根据用户id生成一个
    包含该id的url token
    """
    s = Serializer(SECRET_KEY, expiration)
    return s.dumps({'id': id}).decode('utf-8')

token = make_serialier_url(u.id)
url = 'http://your_domain_here/forget_password/?token=%s' % (token)

#然后调用发送邮件的函数发送改url
send_mail(url, to_user)
```

### 去注册邮箱查看重置密码url
![重置密码链接](https://ws1.sinaimg.cn/large/005B3DIrgy1fke9u0g1zrj30if04cweh.jpg)

### 后台检测用户点击url的合法性
用户通过邮件里面的重置链接，进入到重置密码页面之前，后台需要检测这个url的合法性
```
if token:
    try:
        s = TimedJSONWebSignatureSerializer(SECRET_KEY)
        s.loads(token)
        return render_to_response('reset_password.html')
    except Exception as e:
        return HttpResponse('该链接已经失效')
```


链接失效的例子
![链接失效](https://ws1.sinaimg.cn/large/005B3DIrgy1fkeaez4b7uj30ld040747.jpg)


### 重置密码
![重置密码](https://ws1.sinaimg.cn/large/005B3DIrgy1fke9vet5guj309y082dfr.jpg)

用户重置密码的请求，也需要在后台再次检测一次改url的合法性
```
try:
    if token:
        s = TimedJSONWebSignatureSerializer(SECRET_KEY)
        id = s.loads(token).get('id', None)
        if id:
            u = User.objects.get(id=id)
            if new_passwd1 == new_passwd2:
                u.set_password(new_passwd1)
                u.save()
                success = True
            else:
                msg = '两次密码不同'
                success = False
        else:
            msg = '没有找到id'
    else:
        msg = '没有找到token'
        success = False
except User.DoesNotExist:
    msg = '用户不存在'
    success = False
except SignatureExpired:
    msg = '该链接已经失效'
    success = False
except BadSignature:
    msg = '该链接错误'
    success = False
except Exception as e:
    msg = '重置密码失败'
    success = False
return JsonResponse({'data': msg, 'success': success})
```


### 需要完善的地方
攻击者完全可以不断的输入被攻击者的邮件账号通过发送大量的邮件给被攻击者造成一定的影响，同时，大量的发送邮件的请求也会给服务器造成一定的压力。如果是通过自动脚本来大量发送邮件，可能直接造成服务器宕机。

解决的办法是可以加上一些验证码，还有限制每个邮箱地址的发送频率以及限制ip(貌似有代理的情况下也不能解决太多问题)。把发送邮件的功能放到celery做分布式也是不错的选择。


### TIPS
最后，发现一个好用的上传图片到互联网的网站
http://jiantuku.com
支持拖拽上传图片，然后点击可以复制url，方便在其他的地方引用这个图片地址

