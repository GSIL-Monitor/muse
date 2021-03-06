调度系统用户使用说明
======


##调度系统介绍
* 为什么需要调度系统?

			大部分复杂的后台系统都有定时执行的任务，比如计算KPI、dump数据库、更新基础数据等。如果每个系统
		自身实现调度功能，浪费资源且维护困难，实无必要。所以需要一个统一的调度系统，可以满足用户的大部分
		调度需求，对外只暴露接口，内部实现对用户透明，将调度作为基础服务向外提供。

* 关于Jarvis
	
			Jarvis是钢铁侠的智能管家，负责计算处理各种事务。我们希望调度系统能为所有系统提供稳定的任务调度服务，
		用户不需要关心实现细节，能专注于实现自己的业务。Jarvis就像一个管家一样为各系统提供服务。
* 技术架构
	* Akka、RESTful、
* 操作方式
	* web界面
			
				为方便大部分用户操作，我们提供对用户友好的web界面让用户操作，实现基本的
			任务管理，执行结果查看等功能。
	* RESTful API 
	
				对于有高级开发需求的用户，我们提供RESTful接口让第三方服务接入。可以在调度任务配置的过程中实现
			无人工干预的操作，方便有自动化调度配置需求的系统接入。

##概念
* 任务 ： 需要调度的主体，可以配置基本信息，比如调度内容，调度时间，参数，优先级，任务依赖，报警等。
* 执行计划 ： 任务在当天的执行计划
* 执行流水 ： 任务的执行详情
* 原地重跑 ： 重跑某个任务(原参数)
* 手动重跑 ： 重跑某个任务，可设置数据日期，依赖等
* 应用 ： 任务来源的应用，默认都为jarvis-web，通过REST接入的第三方应用显示对应应用名
* worker ： 真正的任务执行机器
* WorkerGroup ： 任务机器分组，比如WorkerGroup为HIVE的worker都是执行HIVE任务的
* 任务类型 ： 可支持Hive、Shell、Java、MapReduce、Dummy
* 业务标签 ： 任务的业务说明，可方便后期维护，比入KPI
* 优先级 ： 任务的优先级，可设置1(最低)-10(最高)，当前暂无限制
* 提交人 ： 任务的创建者
* 执行用户 ： 执行任务的用户，默认为创建者，重跑的时候为重跑用户
* 调度时间 ： 根据任务的配置，计算出的调度时间
* 开始执行时间 ： 真正执行的时间，集群繁忙的时候可能会比计划调度时间延迟

## 工具箱
* 右上角工具箱最左边的切换可以将任务信息转化成卡片形式，对于显示量大的任务可以更直观。
* 右上角工具箱中间的选项可以选择当前列表显示哪些列，能显示默认隐藏的列，也能隐藏默认显示的列。
* 右上角工具箱最右边的导出选项可以将当前的数据导出到不同格式的文件，支持JSON、XML、CSV、TXT、SQL、World、Excel。

## 分页
* 底栏可以选择每页显示多少条记录，当前最多支持1000条
