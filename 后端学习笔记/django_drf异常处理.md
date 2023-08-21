## 对于django drf框架中的exception_handler()处理方法解读以及自定义异常处理。

#### django的drf框架中，具备一个exception_handler()的异常处理方法，具体代码如下：
```
def exception_handler(exc, context):
    """
    Returns the response that should be used for any given exception.

    By default we handle the REST framework `APIException`, and also
    Django's built-in `Http404` and `PermissionDenied` exceptions.

    Any unhandled exceptions may return `None`, which will cause a 500 error
    to be raised.
    """
    if isinstance(exc, Http404):
        exc = exceptions.NotFound()
    elif isinstance(exc, PermissionDenied):
        exc = exceptions.PermissionDenied()

    if isinstance(exc, exceptions.APIException):
        headers = {}
        if getattr(exc, 'auth_header', None):
            headers['WWW-Authenticate'] = exc.auth_header
        if getattr(exc, 'wait', None):
            headers['Retry-After'] = '%d' % exc.wait

        if isinstance(exc.detail, (list, dict)):
            data = exc.detail
        else:
            data = {'detail': exc.detail}

        set_rollback()
        return Response(data, status=exc.status_code, headers=headers)

    return None
```
&emsp;&emsp;其中的两个参数exc与context，分别表示本次请求发生的异常信息，与本次请求发送异常的执行上下文[本次请求的request对象，异常发送的时间，行号等...]，
```
isinstance(exc, Http404)
```
&emsp;&emsp;isinstance()方法用来判断exc是否与Http404是同一个实例，实际上这一串的代码是用来判断实例的类型PermissionDenied是表示文件权限被拒绝。
&emsp;&emsp;然后接下来再对exc判断其是否为api方面的报错信息，用一个headers的字典来收集错误属性，getattr()方法来返回exc的对象属性。
```
if getattr(exc, 'auth_header', None):
    headers['WWW-Authenticate'] = exc.auth_header
```
&emsp;&emsp;这里的意思是exc中是否存在，如果存在'auth_header'，那么handers的字典当中就会记录下来
```
if getattr(exc, 'wait', None):
    headers['Retry-After'] = '%d' % exc.wait
```
&emsp;&emsp;'wait'则是表示等待的时间，如果是超时也会对其进行记录，最后用isinstance判断exc.detail是否是列表或是字典，如果不是，则创建一个字典用来存值，最后将保存的date和headers作为参数传给Pesponse()做一个返回。

&emsp;&emsp;set_rollback()是django的transaction模块的函数，用于设置事务的手动提交与回滚状态，因为save()方法并不支持复杂的数据保存。


&emsp;&emsp;但exception_handler()方法只判断了两种错误，如果是数据库方面的错误就会直接跳过，所以我们需要在自定义一个有关数据库的错误，具体代码代码如下：
```
def custom_exception_handler(exc, context):
    # 调用drf框架原生的异常处理方法
    response = exception_handler(exc, context)

    if response is None:
        view = context['view']
        if isinstance(exc, DatabaseError):
            logger.error('[%s] %s' % (view, exc))
            response = Response({'message': '服务器内部错误'}, status=status.HTTP_507_INSUFFICIENT_STORAGE)
            return response
```
&emsp;&emsp;定义一个response来接受exception_handler()的返回值，随后对response做一个判断，如果response不为None，那说明错误已经被判断出来了，如果为None，那么就只有2种情况：要么程序没出错，要么就是出错了而djang0或者restframework不识别，也就如同上面所说的，不是网络和文件类型的错误。那么就可以引入django.db中的DatabaseError，用来判断是否是数据库的错误，如果是的话，直接在日志中记录下错误信息，再传入Response()方法中做一个返回。

&emsp;&emsp;其中的logger则是引入python中的logging，用来对这次的错误做一个日志的记录。
```
import logging
logger = logging.getLogger('django')
```
