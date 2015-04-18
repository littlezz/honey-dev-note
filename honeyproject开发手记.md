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

4-18
---------

今天重写了热门， 性能提升了750%  

###能让数据库做的事情就不要让python做
数据库优化是一个说几天几夜都说不完的东西， 今天着手于数据库的查询优化。  
能让数据库做的事情， 尽量让数据库来做， 比如按照日期排序啊之类啊。  django提供了`F`语法和一些聚合语法来加强这些操作。  

如果在python这个层面来做， 很多东西就会比较慢。  

另外就是对于排序这些， 如果使用了列表，里面存放了model 的实例， 可能真的要考虑性能了， 主要在数据的传递上面。对于长度比较大的列表，返回就不能直接返回了， 应该使用迭代器。  

###数据库查询优化
提供了`select_related`, `prefetch_related`,`Prefetch`。    
我记得之前火狐的一个开发者说用这个速度提升了2000倍。 当然他那个比较浮夸了， 但是总的来说， 速度提升，sql查询次数下降这些都是很明显的。  

####`select_related`  
主要提前载入一对一的键和外键。和主查询同时进行。   

####`prefetch_related`
和`select_related`的目的是一样的， 但是实现方法不一样。 支持多对多和多对一的关系。 原理是另外进行一次sql查询， 独立于主查询， 在主查询生效之后发生。然后再python层面进行合并。  
他的宗旨和queryset不一样， 他会一次读取所有数据， 然后存入内存中， queryset则是尽量减少一次性载入。  
之后， 主查询里面要查询相关的关系的时候， 就会直接从存缓中寻找， 不再接触数据库。


这个优化对性能的提升是非常明显的， 说说副作用。  

很明显， 一次性读取所有有关的数据会加大内存的开销， 所以使用的时候要小心， 需要想清楚， 提取的这些数据是不是被大量共享的。
而且他不能对不同的queryset优化  

文档给出了样例

```
>>> pizzas = Pizza.objects.prefetch_related('toppings')
>>> [list(pizza.toppings.filter(spicy=True)) for pizza in pizzas]
```

只对`pizza.toppings.all()`有效， `pizza.toppings.filter`会被认为是另一个query。

另外不能和`iterator()`一起用， 因为他们是相反的优化方式。

####`Prefetch`
django1.7之后引入的， 大幅强化`prefetch_related`。  

由于`prefetch_related`是另一个sql查询，所以可选的`query`参数是对这个sql查询进行自定义。 可以使用filter之类的操作。  

另外因为是在python层面进行合并的， 所以`to_attr`自定义了存入的属性名字， 不然就是默认覆盖相关的属性名字。  

ref:  
[queryset ref](https://docs.djangoproject.com/en/1.8/ref/models/querysets/)  
[Prefetch](https://docs.djangoproject.com/en/1.8/ref/models/querysets/#prefetch-objects)

不想变成姿势普及贴， 只写了自己的思考和一些细节， 具体看查文档。


###关于redis  
redis 值键对存储结构， 虽然在内存中读写， 但是不代表就应该完全依赖他， 数据库的读取是有相应的优势的， redis的优势应该在于他的储存方式和对存入的数据结构没有要求。  
而django的orm强化了数据库的操作， 对于复杂的查询有着天然的优势。
对于大量的数据读取， 我觉得还是要回到数据库读取。  

当然还是要看情况。  
最后还是拼经验， 还有具体的性能测试。  

django文档上关于优化的第一条， 性能测试先行。  

总结起来就是没做测试之前乖乖闭嘴。  

**No do no bb**

___
Have fun!  

2015年04月18日20:36:19
