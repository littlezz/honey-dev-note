honey开发手记
==========
之前没有记录, 4月中旬才想到要记录的。  

### 简介
honeyproject 使用django框架， 主要基于djangorestframework。  

最初版的开发时间是2月份， 耗时3周。  
期间包括：

- 设计所有API  
- 同时撰写API文档及提供相关例子以及普及相关知识。  
- 编程。  
- 回答前端外包的问题， 通过错误码猜测他们程序的错误， 帮助他们调整bug  
- 和运维配合搭建服务器。  
- 花了一个下午教html前端怎么用git。

开发时使用的库：  
- django==1.7.7
- djangorestframework==3.0.5
- python-social-auth==0.2.2
- django-redis
- celery
- 各种服务的sdk

数据库：   
- mysql  



3月份引入redis 和celery。  
redis和celery当时主要用于热门的周期计算， 以及异步的推送。  


之前的记录到此为止

---

4-16
-----
今天修复了python-social-auth在过滤用户名的时候使用的正则表达式有问题， 以及整个被过滤掉之后没有判断为空的逻辑问题。  

在正则中， `[]` 里面虽然`+`, `.`这些都失效了， 但是`-`却有特殊的含义， 表示取一个范围。例如`re.compile(r'[0-9]+')`这样。
如果要匹配`-`, 应该对其转译或者放在末尾。  

原先python-social-auth 里面的代码是

```
CLEAN_USERNAME_REGEX = re.compile(r'[^\w.@+-_]+', re.UNICODE)
```
应该是

```
CLEAN_USERNAME_REGEX = re.compile(r'[^\w.@+_-]+', re.UNICODE)
```

或者

```
CLEAN_USERNAME_REGEX = re.compile(r'[^\w.@+\-_]+', re.UNICODE)
```

bug 提交在  
<https://github.com/omab/python-social-auth/issues/594>

另外就是过滤之后`final_username` 可能为空， 这个没有判断。 
<https://github.com/omab/python-social-auth/pull/595>
 
额外学习到的东西就是  

```
Klass.objects.filter(username='').count() > 0
```
这个语句是一定False的。。。

###感想
正则不懂是不是特意这样的， 不得而知， 但是大家都没有发现， 想必是国外那些有名的公司在用户名的限制上比较严格吧。  

比如qq这边， psa用qq的nickname作为username的， 然后有些人的用户名直接就是中二得不行。

比如，`◥▇▇▇╋︻`

我日你妈。  

2015年04月16日20:56:53


