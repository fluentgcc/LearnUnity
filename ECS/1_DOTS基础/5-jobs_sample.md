# Jobs Sample

[git](https://github.com/Unity-Technologies/EntityComponentSystemSamples/tree/master/EntitiesSamples/Assets/Tutorials/Jobs)

## 0. 问题

- Seekers(蓝色cube)，Targets(红色cube),各自在2平面上超随机方向匀速运动。
- 绘制白色调试线，连接每个seeker与其最近的target。

## 1. 不使用Job

## 2. 使用单线程job

用Profile看实际在主线程运行，但是性能优提升。

- 只有一个Update
- 连续内存 (影响更大)
- 打开burst，性能更好。（Jobs->Burst->Enable Complation）

## 3. 使用并行job

- 注意 NeareastTargetPositons 是按index访问，所以没有竞争条件。
- 注意输入的两个array，\[ReadOnly\]，类型安全能保证不被修改。
- 尝试batch size。选择合适的。

## 4. 使用并行job，并改进算法

- 减少循环次数，多加一个SortJob。

结果

- FindNereast时间占比极少了，剩下时间是Go不够高效，可以直接用Graphics底层batch draw, 也可以用ecs架构解决。
