# 行为树
  * 基于该框架思想，将一款游戏产品的完整流程，分解成不同或重复的行为，通过用户或逻辑驱动某个行为根据数据跳转到另一个行为，最终完成整个流程。

----

## 定义
行为树（Behavior Tree），是指由行为节点组成的树状结构。  

在行为树中，节点是有层次（Hierarchical）的，子节点由其父节点来控制。每个节点的执行都有一个结果（成功Success，失败Failure或运行Running），该节点的执行结果都由其父节点来管理，从而决定接下来的行为，父节点的类型决定了不同的控制类型。节点不需要维护向其他节点的转换，节点的模块性（Modularity）被大大增强了。实际上，在行为树里，由于节点不再有转换，它们不再是状态（State），而是行为（Behavior）。

## 设计
### BevTreeConfig
行为树配置，一般通过读取json文件然后转成对应类。

### Blackboard
一棵行为树共享数据的对象，方便树内节点进行数据共享和更新。

### Tick
通过该类对象可以获取 BehaviorTree、Blackboard等对象引用。在每次执行数的时候，都会作为参数传入。

### BaseNode、BaseNodeWorker
行为树节点基类。  
每个节点执行tick()时，会返回一个状态值。  
行为树的原理，就是利用树分叉、每个节点执行后返回的状态值的不同，从而产生不同的行为分支。  
总共有4种状态，如下定义：  
```go
const (
  SUCCESS Status = 1
  FAILURE Status = 2
  RUNNING Status = 3
  ERROR Status = 4
)
```
RUNNING状态该节点 本次执行尚未完成行为，其余返回值均代表该节点行为执行完毕。  

### 节点类型
  * 序列（Sequence）节点：顺序执行所有子节点返回成功，如果某个子节点失败返回失败。
  * 循环（Loop）节点：循环执行子节点到指定次数后返回成功，如果循环次数为-1，则无限循环。
  * 条件（Condition）节点：根据条件的比较结果，返回成功或失败。
  * 动作（Action）节点：根据动作结果返回成功，失败，或运行。
  * 等待（Wait）节点：当指定的时间过去后返回成功
