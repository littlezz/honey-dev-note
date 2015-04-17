honey开发手记
==========
[littlezz](https://github.com/littlezz)

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


2月份主要是和外包配合， 所以有点不爽吧。
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

另外就是过滤之后`final_username` 可能为空， 这个没有判断.  

<https://github.com/omab/python-social-auth/pull/595>
 
额外学习到的东西就是  

```
Klass.objects.filter(username='').count() > 0
```
这个语句是一定False的。。。

###感想
这么久大家都没有发现， 想必是国外那些有名的公司在用户名的限制上比较严格吧。  

国内的。。。  
比如qq这边， psa用qq的nickname作为username来保存的， 然后有些人的用户名直接就是中二得不行。

比如，`◥▇▇▇╋︻`

我日你妈。  

2015年04月16日20:56:53

4-17
---------
今天着手编写另一个模块， 而且据说原先计划这部分就是大头= =   

###delete删除
django的instance删除之后， 指向他的ForeignKey的记录也会被删除。  
其他的OneToOneField 和 ManyToManyField不会又影响。  
如果这不是你想要的， 可以添加`on_delete`参数。  

比如

```python
class Blog(models.Model):
    pass
    
class Entry(models.Model):
    blog = models.ForeignKey(Blog)
    
```
这样， 当blog实例被删除的时候， 指向他的entry都会被删除。

如果不想这样， 

```
class Entry(models.Model):
    blog = models.ForeignKey(Blog, blank=True, null=True, on_delete=models.SET_NULL)

```
将相应的entry的外键设置为null。

ref:  

- <https://docs.djangoproject.com/en/1.8/topics/db/queries/#deleting-objects>  
- <https://docs.djangoproject.com/en/1.8/ref/models/fields/#django.db.models.ForeignKey.on_delete>  



###关于数据库设计
之前听说数据又冷热之分， 一直不是很理解， 今天终于知道了， django查询数据库的时候会把所有的字段都提取出来， 所以如果总是提取不是经常需要的数据就会很慢。  
另外， 热的数据可以用缓存存起来， 方便及时的修改。  

django提供了`defer`和`only`这些方法来决定获取数据的时候不提取那些数据。 

现在假设我们是这样设计数据库的

```python
class Blog(models.Model):
    title = models.CharField(max_length=50)
    content = models.TextField()
    view_times = models.IntField() 
```  

然后我有一个查询， 提取所有的blog的title

```python
for b in Blog.objects.all():
    print(b.title)
```

我只需要title。 但是每次django从数据库中提取出一个blog的时候， 默认提取所有的字段， 也就是content这个字段也会读取， 假设content里面存了10k个字符， 那这个简单的查询将会变成**无比的慢**。  

####解决方法
一种是用django提供的方法， 但是如果你有很多不必每次都获取的字段， 那数据库应该设计成， 不常用的数据单独在一个table， 然后用OneToOneField连接。  

```python
class Blog(models.Model):
    title = models.CharField(max_length=50)
    view_times = models.IntField()
    
class Article(models.Model):
    blog = models.OneToOneField(Blog)
    content = models.TextField()
    ...
```

另外，如果有字数限制， 应该使用`CharField`， `TextField`的`max_length`只是对渲染出来的表单又限制， 对于实际存入的数据不做检查。 而且`CharField`的速度要比`TextField`快。 


###关于restframework
之前一直在想怎么样让一个model里面自己的字段用一个字典包裹， 今天终于想到解决办法了。

```python
class Blog(models.Model):
    title = models.CharField(max_length=50)
    content = models.TextField()
    view_times = models.IntField() 
    
    def nest(self):
        return {'title': self.title,
                'view_times': self.view_times}

```
在serializer中

```python

class InfoSerializer(serializer.Serializer):
    title = serializer.CharField()
    view_times = serializer.IntField() 


class BlogSerializer(serializer.ModelSerializer):
    info = InfoSerializer(source='nest')
    
    class Meta:
        fields = ('content', 'info')
    
```
但是这个前提是要在model里面写好方法。还是不方便。  
有空再想想

2015年04月17日20:43:40

