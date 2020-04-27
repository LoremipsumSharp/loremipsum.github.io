---
title: 使用ValueTask减少假异步代码引起的GC
---
## async/await代码存在的问题
async/await是.NET4.5引入的一个语法糖，如下所示，下面的代码首先会以同步的方法判断一个文件是否存在，如果存在，那么就会已异步的方式去读取文件的内容：

```
public async Task<string> ReadFileAsync(string filename)
{
    if (!File.Exists(filename))
        return string.Empty;
    return await File.ReadAllTextAsync(filename);
}
```
通过async/await开发人员可以编写更加简洁的异步代码，通过IL SPY我们可以观察到，编译器实际上将async/await转为位了一个StateMachine,如下所示：

```
[AsyncStateMachine(typeof(Program.<ReadFileAsync>d__14))]
public Task<string> ReadFileAsync(string filename)
{
    Program.<ReadFileAsync>d__14 <ReadFileAsync>d__;
    <ReadFileAsync>d__.filename = filename;
    <ReadFileAsync>d__.<>t__builder = AsyncTaskMethodBuilder<string>.
    Create();
    <ReadFileAsync>d__.<>1__state = -1;
    AsyncTaskMethodBuilder<string> <>t__builder = <ReadFileAsync>d__.<>t__
    builder;
    <>t__builder.Start<Program.<ReadFileAsync>d__14>(ref <ReadFileAsync>d__);
    return <ReadFileAsync>d__.<>t__builder.Task;
}
[CompilerGenerated]
[StructLayout(LayoutKind.Auto)]
private struct <ReadFileAsync>d__14 : IAsyncStateMachine
{
    void IAsyncStateMachine.MoveNext()
    {
        int num = this.<>1__state;
        string result;
        try
        {
            TaskAwaiter<string> awaiter;
            if (num != 0)
            {
                if (!File.Exists(this.filename))
                {
                    result = string.Empty;
                    goto IL_A4;
                }
                awaiter = File.ReadAllTextAsync(this.filename,
                default(CancellationToken)).GetAwaiter();
                if (!awaiter.get_IsCompleted())
                {
                    this.<>1__state = 0;
                    this.<>u__1 = awaiter;
                    this.<>t__builder.AwaitUnsafeOnCompleted<TaskAwaiter
                    <string>, Program.<ReadFileAsync>d__14>(ref awaiter, ref
                    this);
                    return;
                }
        }
        else
        {
            awaiter = this.<>u__1;
            this.<>u__1 = default(TaskAwaiter<string>);
            this.<>1__state = -1;
        }   
        result = awaiter.GetResult();
    }
    catch (Exception exception)
    {
        this.<>1__state = -2;
        this.<>t__builder.SetException(exception);
        return;
    }

    IL_A4:
    this.<>1__state = -2;
    this.<>t__builder.SetResult(result);
    }
}
```

注意到上面的代码，当文件不存在的时候会执行goto语句，也就是this.<>t__builder.SetResult(result)，t_builder是AsyncTaskMethodBuilder<T>的一个实例，
我们可以看下SetResult的[实现](https://github.com/dotnet/runtime/blob/110282c71b3f7e1f91ea339953f4a0eba362a62c/src/libraries/System.Private.CoreLib/src/System/Runtime/CompilerServices/AsyncTaskMethodBuilderT.cs)

```
public void SetResult(TResult result)
{
    // Get the currently stored task, which will be non-null if get_Task has already been accessed.
    // If there isn't one, get a task and store it.
    if (m_task is null)
    {
        m_task = GetTaskForResult(result);
        Debug.Assert(m_task != null, $"{nameof(GetTaskForResult)} should never return null");
    }
    else
    {
        // Slow path: complete the existing task.
        SetExistingTaskResult(m_task, result);
    }
}
```
由于此时这个时候并没有异步代码，所以m_task is null 成立,查看GetTaskForResult(result)：

```
[MethodImpl(MethodImplOptions.AggressiveInlining)] 
// method looks long, but for a given TResult it results in a relatively small amount of asm
internal static Task<TResult> GetTaskForResult(TResult result)
{
    if (null != (object?)default(TResult)) // help the JIT avoid the value type branches for ref types
    {
        .....
    }
    else if (result == null) // optimized away for value types
    {
        return s_defaultResultTask;
    }
    return new Task<TResult>(result);
    }
}
```
由于string是引用类型且result不为null,所以必然会导致一个新的Task对象被创建，那么也就是说，即便不存在异步调用，也会创建一个新的Task对象，如果存在大量的假异步调用，势必会照成大量的一代GC，从而影响程序的性能。

## 问题的解决
为了解决上述问题，.NETC CORE 2.1引入了一个新的概念，ValueTask：

```
public struct ValueTask<TResult>
{

    IValueTaskSource<TResult>
    internal readonly object _obj;
    internal readonly TResult _result;
}
```

使用ValueTask改写ReadFile代码：

```
public async ValueTask<string> ReadFileAsync(string filename)
{
    if (!File.Exists(filename))
        return string.Empty;
    return await File.ReadAllTextAsync(filename);
}
```
当调用ReadFileAsync时，可以使用ValueTask.IsCompleted来判断这个调用是不是假异步调用，如下所示：

```
var valueTask = ReadFileAsync2();
if(valueTask.IsCompleted)
{
    return valueTask.Result;
}
else
{
    return await valueTask.AsTask();
}
```
如果假异步调用，IsCompleted为true，这个时候返回valueTask.Result,不会产生Task对象的分配而ValueTask本身也是一个值类型，也不会产生allocation。反之，如果是真异步调用则按照原来的逻辑去执行。


## 结论
当应用程序对性能要求比较苛刻的时候并且存在大量假异步调用的情况下，可以考虑使用ValueTask来提高性能。