## 开头

## 修改流程实例

流程引擎提供流程实例修改API ，包括 `RuntimeService.createProcessInstanceModification(..)`或 `RuntimeService.createModification(...)`

这些API允许通过使用流式的构造器在一次调用中指定多个 修改指令

而且它还能做到：

- 在一个活动之前就开始执行
- 在离开活动的序列流上开始执行
- 取消一个正在运行的活动实例
- 取消一个给定活动的所有运行实例
- 设置变量

该api功能非常强大；

* **我们可以借助该api实现诸如任意跳转、驳回起始节点、退回上一个等各种中国式流程需求**
* 修复流程实例，其中一些步骤必须重复执行或跳过
* 将流程实例从流程定义的一个版本迁移到另一个版本
* 测试阶段：可以跳过或重复做一些活动，以便对个别流程段进行独立测试。

### 功能特性

#### 接口列表

- `startBeforeActivity(String activityId)`、`startBeforeActivity(String activityId, String ancestorActivityInstanceId)`

  通过`startBeforeActivity`在一个活动之前启动。该指令支持`asyncBefore`标志，意味着如果活动是`asyncBefore`，将创建一个job。一般来说，该指令从指定的活动开始执行流程模型，直到遇到一个等待状态

- `startAfterActivity(String activityId)`、`startAfterActivity(String activityId, String ancestorActivityInstanceId)`

  通过 `startAfterActivity` 在一个活动之后运行，意味着将从活动的下一节点开始执行。该指令不考虑活动设置的 `asyncAfter` 标志。如果有一个以上的下一节点或根本没有下一节点，则该指令将失败

- `startTransition(String transitionId)`、`startTransition(String transition, String ancestorActivityInstanceId)`

  通过 `startTransition` 启动一个过渡，就意味着在一个给定的序列流上开始执行。当有多个流出的序列流时，这可以与 `startAfterActivity` 一起使用

- `cancelActivityInstance(String activityInstanceId)`

  可以通过`cancelActivityInstance`取消一个特定的活动实例。既可以是一个活动实例，如用户任务的实例，也可以是层次结构中更高范围的实例，如子流程的实例

- `cancelTransitionInstance(String transitionInstanceId)`

  过渡实例表示即将以异步延续的形式进入/离开一个活动的执行流。一个已经创建但尚未执行的异步延续Job被表示为一个过渡实例。这些实例可以通过`cancelTransitionInstance`来取消

- `cancelAllForActivity(String activityId)`

  为了方便起见，也可以通过指令 `cancelAllForActivity` 来取消一个特定活动的所有活动和过渡实例

#### 修改变量

每一个实例化活动 (i.e., `startBeforeActivity`, `startAfterActivity`, or `startTransition`), 都可以提交流程变量。 我们提供了下面这些API：

- `setVariable(String name, Object value)`
- `setVariables(Map<String, Object> variables)`
- `setVariableLocal(String name, Object value)`
- `setVariablesLocal(Map<String, Object> variables)`

变量会在 创建了实例化的必要范围 之后，在指定活动执行之前。 这意味着，在流程引擎历史记录中，这些变量不会出现，因为它们是在执行指定活动的 “startBefore” 和 “startAfter” 指令时设置的。局部变量设置在即将执行的活动上被设置，例如，进入活动等

#### 活动实例树

流程实例修改API是基于 *活动实例* 的。一个流程实例的活动实例树可以通过以下方法来查询

```java
ActivityInstance activityInstance = runtimeService.getActivityInstance(instId);
System.out.println(activityInstance);
```

ActivityInstance是一个递归的数据结构，由上述方法调用返回的流程实例代表流程实例。ActivityInstance对象的ID可用于取消特定实例 或者 在实例化中指定父级。

接口 “ActivityInstance” 有 “getChildActivityInstances” 和 “getChildTransitionInstances” 方法，可以在活动实例树中向下查询。例如，假设活动 *Assess Credit Worthiness* 和 *Register Application* 是活动的。那么活动实例树看起来如下。

```
ProcessInstance
  Evaluate Loan Application
    Assess Credit Worthiness
    Register Application Request
```

在代码中， *Assess* 和 *Register* 可以通过对活动实例进行如下查询获取到:

```java
ProcessInstance processInstance = ...;
ActivityInstance activityInstance = runtimeService.getActivityInstance(processInstance.getId());
ActivityInstance subProcessInstance = activityInstance.getChildActivityInstances()[0];
ActivityInstance[] leafActivityInstances = subProcessInstance.getChildActivityInstances();
// leafActivityInstances has two elements; one for each activity
```

也可以直接查询一个特定活动的所有活动实例：

```java
ProcessInstance processInstance = ...;
ActivityInstance activityInstance = runtimeService.getActivityInstance(processInstance.getId());
ActivityInstance assessCreditWorthinessInstances = activityInstance.getActivityInstances("assessCreditWorthiness")[0];
```

与活动实例相比，*过渡实例* 不代表正在执行的活动，而是代表即将进入或即将离开的活动。当异步延续的job存在但尚未被执行时就是这种情况。对于一个活动实例，可以用`getChildTransitionInstances` 方法查询子过渡实例，过渡实例的API与活动实例的API类似

#### 嵌套实例化

实例化子流程中的活动时，会将子流程也实例化。

除了实例化这些父作用域外，引擎还确保在这些作用域中注册事件的订阅和job。例如，考虑如下流程：

![image-20231109140346687](image-20231109140346687.png) 

启动活动 *Assess Credit Worthiness* 也为消息边界事件 *Cancelation Notice Received* 注册一个事件订阅，这样就有可能以这种方式取消子流程。

#### 父级选择

支持传入实例id，那么实例id的孩子节点将会重新实例化。

有了给定的父级活动实例 ID，在父级活动和要启动的活动之间的所有作用域将被实例化，不管它们是否已经被实例化

现在

```
ProcessInstance
  Evaluate Loan Application
    Assess Credit Worthiness
```

使用父级选择

```java
ProcessInstance processInstance = ...;
ActivityInstance activityInstanceTree = runtimeService.getActivityInstance(processInstance.getId());
runtimeService.createProcessInstanceModification(activityInstanceTree.getId())
  .startBeforeActivity("assessCreditWorthiness", processInstance.getId())
  .execute();
```

参数`ancestorActivityInstanceId`取当前活动的活动实例的id，该实例属于要启动的活动的 *父级* 活动。如果一个活动包含要启动的活动（无论是直接的，还是间接的，中间是否有其他活动），它就是一个有效的父级

产生的活动实例树如下所示

```
ProcessInstance
  Evaluate Loan Application
    Assess Credit Worthiness
  Evaluate Loan Application
    Assess Credit Worthiness
```

#### 取消传递

取消一个实例，如果父级下的所有的实例被取消了，那么这个父级也会被取消

注意，如果取消了所有的活动实例（此时不会把流程实例取消），再start一个活动，最终会在这个流程实例下创建start活动实例

#### 执行顺序

衔接上一个顺序，如果子流程内的活动先取消，再启动内部的活动，会重新创建子流程；

如果时先启动内部活动，在取消内部活动，则不会重新创建子流程

修改指令总是按照提交的顺序执行。因此，以不同的顺序执行相同的指令会产生不同的效果。考虑一下下面的活动实例树。

```
ProcessInstance
  Evaluate Loan Application
    Assess Credit Worthiness
```

假设你的任务是取消 *Assess Credit Worthiness* 实例并开始 *Register Application* 的活动。这两条指令有两种顺序。要么先执行取消再实例化，要么先实例化再取消。在前一种情况下，代码看起来如下：

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .cancelAllForActivity("assesCreditWorthiness")
  .startBeforeActivity("registerApplication")
  .execute();
```

由于取消传播，子流程实例在执行取消指令时被取消，只是在执行实例化指令时被重新实例化。这意味着，在修改被执行后，会产生一个不同的 Evaluate Loan Application 子流程的实例。任何与前一个实例相关的实体都已经被移除，例如变量或事件订阅。

与此相反，考虑先执行实例化的情况：

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .startBeforeActivity("registerApplication")
  .cancelAllForActivity("assesCreditWorthiness")
  .execute();
```

由于在实例化过程中的默认父级选择，以及在这种情况下取消不会传播到子流程实例，子流程实例在修改前后是一样的。像变量和事件订阅这样的相关实体会被保留下来。

#### 中断活动

中断活动可以同时创建

如果准备启动的活动存在中断或取消行为，流程实例的修改也会触发这些行为。尤其是，启动一个中断的边界事件或中断的事件子流程将取消／中断其他活动。考虑一下下面的流程；

![image-20231109144257137](image-20231109144257137.png) 

假设活动 *Assess Credit Worthiness* 目前是活动的。事件子流程可以用以下代码启动。

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .startBeforeActivity("cancelEvaluation")
  .execute();
```

由于 *Cancel Evaluation* 子流程的启动事件是中断的，它将取消 *Assess Credit Worthiness* 的运行实例。当事件子流程的启动事件通过以下方式启动时，也会发生同样的情况：

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .startBeforeActivity("eventSubProcessStartEvent")
  .execute();
```

然而，当位于中断事件子流程中的活动被直接启动时，中断不会被执行。思考一下下面的代码：

```java
ProcessInstance processInstance = ...;
runtimeService.createProcessInstanceModification(processInstance.getId())
  .startBeforeActivity("notifyAccountant")
  .execute();
```

由此产生的活动实例树将是：

```
ProcessInstance
  Evaluate Loan Application
    Assess Credit Worthiness
    Cancel Evaluation
      Notify Accountant
```

#### 多实例

多实例的子实例可以新增启动

也可以启动父级多实例活动

对于指定变量名，设置list方式的多实例节点，直接跳转子实例可能会有报错，分析如下：

* 串行多实例

  （直接回到子执行会报错，父执行需要设置审批人变量， 但是可以直接设置myAssigneeVarName、nrOfCompletedInstances、myAssigneeList_1fylv2e等local变量，他们的作用域一样，注意第一次modify的时候不会执行myAssigneeVarName的设置，即behiver的执行，需要手动设置，后面complete的时候会从List中获取）

  报错的原因为，原生代码就不支持：

  ```
  /**
  
   \* Cannot create more than inner instance in a sequential MI construct
  
   */
  
  protected boolean supportsConcurrentChildInstantiation(ScopeImpl flowScope) {
  
    CoreActivityBehavior<?> behavior = flowScope.getActivityBehavior();
  
    return behavior == null || !(behavior instanceof SequentialMultiInstanceActivityBehavior);
  
  }
  ```

  代码记录如下：

  //如果没有活动实例，直接进入子活动会成功，但是这里虽然执行了mutli执行监听器(生成了list审批人集合也没用,myAssigneeVarName没有设置进去)，但是并没有走mutli相关的beahiver，所以不会生成muti正常需要的本地变量，需要手动提交本地变量myAssigneeVarName

  //**如果有子活动，直接进入子活动会报错，因为不符合串行多实例的语义**

  **正常情况串行多实例，会新生成一条串行多实例执行，还有一个整体的父执行。新生成的执行会关联act_hi_actinst中multiInstanceBody和actid的exc_id自动，这两个是一样的**

  **当执行了startBeforeActivity（multi activity）后：**

  * 针对前面执行会给它生成一个父执行，更新parent_id为新生成的执行，新生成的执行的parent_id为最顶级的了
  * 当前操作也会生成一对父子执行
  * 当前操作的act_hi_actinst中multiInstanceBody和actid关联操作的子exc_id，之前的act_hi_actinst关联的exe_id不变，因为之前的执行的parent_id已经修改为新生成的执行了

  **总的来说，就是给之前的执行找了个父亲，本次操作会生成父子执行，和并行多实例类似多了一层，但是act_hi_actinst中关联的都是子执行**

  //注意这里传递的local变量的作用域为父执行，而一些myAssigneeVarName、loopCounter、myAssigneeList_1fylv2e、nrOfActiveInstances、nrOfInstances等变量的作用域为子执行

  /**

   \* └── test_back_up:2:e297b27d-e1a1-11ed-87ee-005056be26d2=>08359c9e-e3f4-11ed-90a3-049226e08e36

   \*     ├── Activity_1fylv2e#multiInstanceBody=>2c43e44d-e3f6-11ed-9f49-049226e08e36

   \*     │   └── Activity_1fylv2e=>Activity_1fylv2e:2c456afd-e3f6-11ed-9f49-049226e08e36

   \*     └── Activity_1fylv2e#multiInstanceBody=>084b95bc-e3f4-11ed-90a3-049226e08e36

   \*         └── Activity_1fylv2e=>Activity_1fylv2e:084be3ec-e3f4-11ed-90a3-049226e08e36

   */

* 并行多实例

  **如果执行为空，和串行多实例一样，不会执行beahiver中的设置**myAssigneeVarName、loopCounter等操作，会直接使用传递的local变量myAssigneeVarName，并且动态生成nrOfInstances的值。

  并行多实例的完成状态由条件或默认代码决定，不由总数nrOfInstances决定，默认情况只要所有执行结束就会完成。

  camunda关于modifyInstacne的bug。新启动子实例会增加nrOfInstances的值，cancels子实例并不不会减少nrOfInstances值，这会造成完成条件依赖此变量会有不可预知的错误。

#### 异步执行

你可以异步地执行单个流程实例的修改 指令与同步修改相同，流式语法如下：

```java
Batch modificationBatch = runtimeService.createProcessInstanceModification(processInstanceId)
        .cancelActivityInstance("exampleActivityId:1")
        .startBeforeActivity("exampleActivityId:2")
        .executeAsync();
```

这将创建一个修改的batch，然后以异步方式执行。 在执行单个流程实例的异步修改时，不支持修改变量。

#### 修改多个流程实例

指定一个流程定义，可以操作多个流程实例的修改。

同步执行的一个例子：

```java
runtimeService.createModification("exampleProcessDefinitionId")
  .cancelAllForActivity("exampleActivityId:1")
  .startBeforeActivity("exampleActivityId:2")
  .processInstanceIds("processInstanceId:1", "processInstanceId:2")
  .execute();
```

异步执行的一个例子：

```java
Batch batch = runtimeService.createModification("exampleProcessDefinitionId")
  .cancelAllForActivity("exampleActivityId:1")
  .startBeforeActivity("exampleActivityId:2")
  .processInstanceIds("processInstanceId:1", "processInstanceId:2", "processInstanceId:100")
  .executeAsync();
```

#### 监听器、输入输出映射

可以再execute指定是否需要跳过监听器或者输入输出。当修改是在一个无法访问所涉及的流程应用部署及其包含的类的系统上时，这可能是有用的。可以通过使用修改构建器的方法 `execute(boolean skipCustomListeners, boolean skipIoMappings)` 跳过监听器和输入输出映射的调用；

### 实际应用

#### 驳回功能

**整体思路：**

* 驳回、撤回

  含义为取消所有的运行任务实例，移动到开始节点，从头再来，此做法不会造成歧义

* 驳回修改

  取消当前任务的运行实例，移动到开始节点，完成任务时，直接跳转回当时取消的节点；

  * 串行多实例

    串行多实例驳回没有必要回到当前的实例（该方案比较复杂，需要记录活动实例id,然后设置之前的多实例的变量loopcounter和list变量来恢复）

    **注意：直接回到子执行会报错，父执行需要设置审批人变量， 但是可以直接设置myAssigneeVarName、nrOfCompletedInstances、myAssigneeList_1fylv2e等local变量，他们的作用域一样，注意第一次modify的时候不会执行myAssigneeVarName的设置，即behiver的执行，需要手动设置，后面complete的时候会从List中获取**

  * 并行多实例驳回回跳需要注意类型：

    * 单人并行

      删除了会取消整个多实例

    * 多人并行

      全部通过才行，恢复过程需要考虑有的人已经通过（需要记录下来，可以查actinstance表查到之前的list，删除已完成的usrtask assginee，然后重新设置这list即可），活动数、总数作用域在父级、loopcounter，比较难实现，还不及全部取消然后全部重新来，可以只添加没完成的，但是这个不适用于自定义条件设置（自定义比例））

    * 多人任意

      删除了会取消整个多实例

**代码示例：**

#### 任意跳转

任意跳转（有限制，只能跳转到同级节点，否则会有不可预测的错误，但是如果都是排他网关的）

​	取消当前活动实例

​	跳转到目标的活动实例

### 注意事项

流程实例修改是一个非常强大的工具，允许随意启动和取消活动。因此，很容易产生正常流程执行所不能达到的情况。假设有以下流程模型 

![image-20231108181622628](image-20231108181622628.png) 

假设活动 *Decline Loan Application* 是活动的。通过修改，活动 *Assess Credit Worthiness* 可以被启动。在该活动完成后，执行被卡在加入的并行网关上，因为没有令牌会从另一个输入流中流入使流程继续下去。这是流程实例不能继续执行的最明显的情况之一，当然还有许多其他情况，取决于具体的流程模型。

流程引擎是 **不能** 检测到修改是否会产生这样的结果。所以这取决于这个 API 的用户，他们要确保所做的修改不会使流程实例处于不希望的状态。然而如果出现这个情况，也可以使用流程实例的修改API也是修复:-)

## Job执行器

## 流程迁移

## 历史数据处理