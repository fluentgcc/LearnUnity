# Unity job system

创建多线程执行的代码，以便重复利用CPU多核心。

## 1. job 示例

### job 定义 计算各个元素的平方的job

```csharp
using Unity.Burst;
using Unity.Collections;
using Unity.Jobs;

[BurstCompile]
public struct SquaresJobs : IJob
{
    public NativeArray<int> Nums;

    public void Execute()
    {
        for (int i = 0; i < Nums.Length; i++)
        {
            Nums[i] *= Nums[i];
        }
    }
}

```

说明：

- 继承 IJob，需要实现 Execute() 接口。
- 使用 NativeArray , 而不是使用一般C#的managed array。NativeArray 是Unity api 中的一个unmanaged native array, 线程安全，且jobs能与主线程共享访问，不需拷贝。
job应该避免使用托管对象，因为托管对象不是线程安全以及GC问题。 使用 Burst 编译器时，无法使用托管对象。

### job使用

```csharp
//... somewhere in main thread code

// instance the job
var job = new SquaresJob { Nums = myArray}

// schedule the job
JobHandle handle = job.Schedule();

// main thread sync the job.
handle.Complete();

```

- 主线程创建数据，传入job。
- Schedule() 将job instance 放入global job queue，当线程空闲会选取作业并在单个线程上执行一次Execute().
- Complete()用于job和主线程的同步。只有job执行完，complete才会返回。Complete()会从job queue移除记录，因此所有Schedule的job必须要Complete(),否则会内存泄漏。
- 只有主线程才能对job进行 Schedule 和 Complete。
- complete之前，主线程不能访问job使用NativeArray。
- 尽可能晚调用 Complete.

### job 依赖

多个job操作同一个对象，如何确定执行顺序。

```csharp
// main thread:

// 多个个job
var job1 = new SquaresJob {Nums = myArray};
var job2 = new SquaresJob {Nums = myArray};

JobHandle handle1 = job1.Schedule();

// job2依赖job
JobHandle handle2 = job2.Schedule(handle);


handle1.Complete();    //可以不执行。
handle2.Complete();


//如果多个依赖， 这里A依赖BCD
//JobHandle.CombineDependencies(handleB, haandleC, handleD)
//JobHandle handleA = jobA.Schedule(combined);

```

- 最末尾叶子job调用Complete()时，会递归调用所有依赖job的Complete, 所以没有必要所有的job都调用Complete()。
- 多次调用Complete()也没事。
- 循环依赖，Schedule时提供依赖，Schedule之后无法改变dependency，所以无法形成循环依赖。

### 并行job

对于 element-independent 的操作，可以并行对 element执行。 或者一些固定迭代次数的操作。

```csharp
[BurstCompile]
public struct SquaresJobs2 : IJobParallelFor
{
    public NativeArray<int> Nums;

    public void Execute(int index)
    {
        for (int i = 0; i < Nums.Length; i++)
        {
            Nums[i] *= Nums[i];
        }
    }
}


//... main thread:

// instantiate the job
var job = new SquaresJobs2 { Nums = myArray};

// schedule and complete the job
JobHandle handle = job.Schedule(myArray.Length, 100);
handle.Complete();

```

- 分成多个 batch 并行执行。示例中Length // 100 个batch.
- batchsize选择？不大不小，否则影响性能。

:::image type="content" source="pictures/1_jobs_1.png" alt-text="使用8核":::

:::image type="content" source="pictures/1_jobs_2.png" alt-text="使用1核":::

- 多核版占用所有CPU核心，在执行期间CPU干不了其他事情，消耗更多。单核用了相同时间做了核相同事情，如何解释？
- 这个函数实际计算量小，真正的bottleneck是 memory access，job分给多核也只能等待内存读写，跟单核一样，反而多占用了cpu时间。
- 区分任务是计算密集型还是io密集型。计算密集型多线程能够提速。
