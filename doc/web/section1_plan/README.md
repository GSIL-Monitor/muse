执行计划介绍
======

        执行计划由调度器根据任务的调度配置每天计算生成。可以查看当天有哪些任务将要执行及最新的执行状态。可以根据条件查询对应的执行任务。执行计划
        列表是以任务为维度显示(多相同的执行则只显示一个任务)，如果要查看任务的执行详情，需要在操作列点击查看任务的执行记录。

## 查询条件(全部支持多选)
* 任务名称:输入时采用模糊搜索方式提示匹配的任务名，再选择对应的任务
* 任务类型:可以选择具体的任务类型
* 优先级:优先级分为1-10,1为最低，10为最高
* 执行用户:触发执行任务的用户

## 状态
* 等待:任务还在等待其他执行条件满足，比如前置任务未执行
* 准备:任务已经满足触发条件，在队列中等待调度
* 运行:任务正在执行
* 成功:执行成功
* 失败:执行失败
* 终止:任务被强制停止
* 删除:任务被删除

## 执行计划列表
* 默认列出任务ID、任务名称、任务类型、业务组名、任务优先级、提交状态、执行状态等信息。
* 可以在操作列选择查看任务的依赖与任务的执行记录。
* 点击任务名可以查看任务的详细配置信息


