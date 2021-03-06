# 调度系统作业调度逻辑

这里暂时不打算把全部概念或细节列全，只涉及为了澄清调度逻辑所需要的内容，后续需要完善。

## 调度逻辑相关的需求和各种细节：

### 概念

* 调度时间
  * 计划调度时间
  * 实际调度时间（任务依赖满足时的时间？）
    * 作业可以设置Expire超时时间？实际一个有Crontime的调度实例（最大的场景就是Ironman提交的一次性任务）超过超时调度时间，直接失败返回。
  * 一个实际任务运行Attempt需要更多的时间参数？ 实际任务派发给worker的时间？worker上派发给Executor的时间？ executor派发给后端集群的时间？ （executor这两个可以是一个？））。

* 数据时间 ： 实际作业需要操作的数据可能有特定的时间范围（目前主要是hive离线任务，其它作业可能也会有时间需求）

* 超时时间：Job需要允许设置Expire时间，用于支持各种返回

* 任务状态 ： 一个作业的运行实例的状态
  * Waiting: 有依赖未满足，可能只在任务展示时有，数据库中不存在。
  * Ready: 依赖满足，Master处等待派发中
  * Scheduled：Worker已接受
  * running：Worker已成功提交后端执行引擎
  * successed
  * failed
  * canceled (也可以用failed＋reason来处理，不过，感觉单独拉出来业务含义更明确)
  * 对失败和成功，需要用一个Reason字段标识任务的失败或成功的原因，这样可以区分真正的失败成功和人为标记的成功失败(一些特殊的override计划plan的操作可能会手动标示成功失败状态)

* Job状态
  * 需要Pause状态以区别Disable状态？ 

### 调度依赖类型的Scope

任务依赖支持：

  * 纯时间调度
  * 只有单个父亲的纯依赖任务（即自身可以没有Crontime）
  * 时间加Offset范围依赖调度（比如现在的这种依赖于当天父任务实例的任务，或者offset范围内特定的依赖策略）
    * 要求父任务必须有crontime，即单亲纯依赖任务向下只能是一路单亲依赖下去。

Offset范围依赖策略，支持：

  * 全部 ALL
  * 任意 ANY
  * Last-N ? 不支持
  
调度频率次数的依赖，不直接支持（比如Arun2次，B Run一次这种），考虑通过Offset＋All这种来覆盖一部分


### 调度配置变更的Scope

* 如果作业的调度关系变更,调度系统需要保证依赖未满足的任务能依赖于新的调度逻辑关系
* 对于依赖已经满足的任务，调度系统可以检查并添加未执行的新任务，但不负责修改依赖满足，已进入Task调度阶段的任务（因为各种条件判断，是否是被修改的调度关系对应的Task，周期变更，是否延迟任务，是否历史重run任务等等，类似的各种逻辑场景太琐碎复杂），这些任务由变更作业调度关系的业务方或应用负责修改或删除。
  * 以上考虑Tradeoff点，基于任务依赖条件满足，但是时间还不满足，或者集群资源还不满足，这时候想要变更调度依赖关系这种组合情况出现概率很低，来简化设计。将来可能的补救手段：
    * 对一些常见变更，可以在前端比如Jarvis-web中实现一些修改逻辑
      * 比如如果能绝对判断和已在Task调度中等待运行的任务是一样的任务，提示用户是否删除。
    * 是否真的还有常见的这种操作场景没有考虑到。如果是一大类固定的逻辑，可以考虑在后续添加Fix Task逻辑的手段。
* 对于理论上本次调度实例已经执行过的任务，由于调度周期，依赖等变更造成的可能的再次触发（不管是变更时，还是系统状态恢复时），原则上能判断出来是同一个周期实例任务的，都不再重复调度。 
  * 如果该实例还没完成调度，则应该用新的调度计划调度，但是如果已经调度完成，进入task调度阶段了，则如上描述，暂不考虑删除旧任务工作。
  * 举例：一个纯时间任务是每天3点运行，然后8点的时候改成每天5点或10点运行，同一个周期间隔内（天）已经有一个实例运行过了，对改到5点的任务，8点不应该触发，改到10点的任务，10点时间到了，也不触发任务的运行。上述两者，如果3点的任务都没有来得及运行过，则应该触发。
  * 时间＋offset依赖的任务类似
  * 有Crontime和依赖的任务，判断时不考虑依赖关系变更的问题。如果只是因为依赖关系变更，比如加了一个新的父任务依赖，新的父任务完成，依赖关系触发后，检查发现当前条件满足了，但是同一个crontime间隔范围内已经有一个跑过的任务，理论上不再跑，（可能需要输出log纪录，纪录这种情况）的确需要再跑，由人工触发。
  * 大于天的调度间隔的例子，因为没有任务列表，很可能用户没法判断是否将来任务是否会执行，这时候，可能需要增加提示，提醒用户如果需要等周期再运行，需要添加一个一次性任务（可以的话，用户确认后，自动添加）
  * 没有crontime的单亲纯依赖任务，变更依赖关系，变更完如果变更后新的父任务再次满足，则触发，不检查新的父任务的历史情况来判断是否要补充触发，所以业务流程不变。
    * 但是系统崩溃以后怎么办，系统状态重建时，纯依赖任务也是要检查上一次的状态的，来补充运行的，这时如何判断这些新的父任务，哪些是变更之前完成的，哪些是之后？
      * 简单的重用方案，可以考虑，凡是变更纯依赖任务的依赖关系的，需要设定lastchecktime，明确截断历史。这样不需要做过多的特殊逻辑判断。


###  数据结构

* 每次Task的执行（包括原地Retry执行），需要有一条单独的执行记录，但是各个Retry逻辑上要能关联成一个执行节点。

* Schedule表里面，可能需要一条类似，lastCheckTime这样的纪录，防止在schedule新建以后，或者系统崩溃之类的场合下，恢复作业，判断历史时，没法判断历史上有Schedule本身被拖延了很多次没有执行呢，还是schedule新建，或者变更，根本就不需要对应的历史，用于截断历史任务的检查时间范围。

## 如何配置各种调度作业？

### 老系统作业适配

* 基准check，dump类任务 ： crontime

* 长周期无依赖任务
  * 真的不需要依赖？ -> crontime
  * 其实需要，但是以前人为的认为时间可以保证？ -> corntime ＋ offset依赖，加强管理。

* 按日执行一次的作业
  * 没有依赖的，指定天为基准的CronTime
  * 有前置依赖的
    * 单亲任务：指定依赖即可。
    * 多亲任务：指定Crontime为零点，指定offset依赖为0-24小时或current day，依赖策略为ANY或ALL（根据情况，前置任务也是一天为周期的，貌似都可以，现在有不是这种情况的特例么？）
      * 如果确实需要调整crontime，比如设置为8点，来错开高峰时间的话，offset的指定current day这样的范围描述比较灵活。

* 小时作业
  * 没有依赖的，指定特定周期的CronTime
  * 有前置依赖的
    * 单亲任务：可以只指定依赖？
    * 多亲任务：指定周期 Cron time，指定正确的offset依赖范围。


## 如何执行各种调度作业？

调度组件：

* DAG Job调度模块：管理作业依赖关系，包括提交依赖满足的任务，但时间尚未满足的给TimeScheduler
* DAG Task调度模块，用来定制化的管理一次性的，需要针对任务实例进行依赖管理的任务。 
* Task Manager，单纯的Task调度，管理已经完成依赖和时间关系的任务，仅仅取决于集群资源的任务。一旦满足随时可以run的任务，Task调度不负责监控任务依赖关系。


类型|正常逻辑|系统状态崩溃恢复重建逻辑|父任务依赖推导|重复触发逻辑|
---|---|---|---|---|
无依赖任务|crontime|检查上一次成功执行时间＋expire策略，确定还需要启动哪些|无依赖，但需要上次任务的计划调度时间|同周期已涵盖任务，不再触发|
单亲纯依赖|父任务完成就触发|检查上一次成功所依赖父任务，以及是否有新的父任务完成|需要查询上次任务的触发任务|不存在正常触发但不该执行的|
Offset依赖|父任务触发时检测offset范围内父任务完成列表，依赖满足后提交crontime检查时间？|检测上一次成功执行时间，确定后续还有哪些条件满足的任务应该开始执行|根据计划调度时间和offset计算|计算被触发实例，以及是否已有同周期实例被执行|

* 如何检查除了最近一次执行记录之后的任务计划，之前的时间段还有哪些任务可能需要处理的如何检查？
  * 所有失败作业（达到失败次数），自身不再主动重跑
  * 历史上如果存在A0成功，A1没有记录，A2成功这种空洞的作业执行记录（理论上可能出现。。。），那么不再主动检测A1的条件是否已经满足。暂时不予处理。
  * 用lastCheckTime来截断检查历史（某些流程或者各种异常情况可能需要设置这个字段）

* 配置了Expire策略的任务，需要在Expire时间到达时，提前取消并失败。
  * 哪些任务需要配置Expire策略？ 还是有Crontime的都可以 

* 如何保存和加快父任务完成情况检查，但又不需要引入持久化，或着预plan？
  * 考虑检查过的父任务和最近完成的父任务加入Cache，用FIFO或LRU淘汰


## 异常恢复

前面很多内容涉及到了具体不同任务的异常恢复的逻辑和细节，总体上流程上，server启动时，遍历所有等待资源和运行中任务记录表，重建Task调度表，然后遍历Job，按Job的崩溃重建逻辑，进一步恢复所有应该调度但是任务表中尚未记录的任务，这之后，再开始调度以及对外提供服务。


## 如何展示各种调度作业相关信息？

理论上，所有的作业的历史执行情况和未执行的列表都可以计算得到，但是任务的安排走的是实时变更触发机制，所以内存中并没有存储将来的一个具体的任务的列表，而且一个纯依赖任务，具体被触发的时刻是不确定的，严格来说，如果指定一个时间段，精确的未执行任务的列表是无法获得的。

但是实际上会需要这样的一个列表进行全局状态的监控，估计平台作业的整体进度，或者查看执行情况（特别是当前hive开发平台这种有明确plan的）。而另一方面，针对某一个具体的任务，又需要保证能提供手段分析出精确的执行情况。 这两方面的要求各有各的难点。

### 按时间段展示的全局列表维度的信息

对全局监控型的需求，考虑按天生成一个估计的作业列表：

* 这张表作为展示依据，不作为调度依据
* 这张表用作整体进度估计和展示，用作快速评估集群状态，不作为也不可能作为具体每一个任务的精确调度状态展示，精确调度状态的展示由任务维度的信息来支持。 
* 如果这张表以后能做到精确完美的实时变更同步，不是不能考虑后续可以考虑作为调度依据，毕竟实时变更的准确同步是我们的要求，有没有plan只是实现方案。
  
可能的实现逻辑：

* 每天计算一次理论上的周期作业的列表，这个作业列表里面，可以估计执行次数(让短周期任务的统计更精确)，但是不涉及任务的具体依赖关系。
* 如果有任务cron表更新，更新这张表。
* 任务执行成功，标注这张表。增加次数，不标注其它具体信息
* 任务显示时可以按理论执行次数区分执行进度。
* 一次性执行的任务：
  * 因为可能没有没有独立的Job信息，但是和周期任务的执行情况又需要加以区分，考虑放在在一张单独的任务的表里，标注等流程和周期任务分开，以避免互相干扰，比如进度上是计算前天的数据，不能标注成功一次，让用户以为今天完成了。（如果有方案能做到互相不影响，可以考虑合并）
* 具体任务的精确的执行情况，比如5次执行了三次，是不是真的有问题，是不是中间调度计划有变更导致？这个每个任务自己再提供精确的执行流水和理论执行次数，由用户自己判断，系统不做判断。
  * 举例： 比如一个作业之前12小时run一次，13点的时候，改成6小时run一次，这样理论计算出来一天有4次，但是今天实际只会执行3次，这个系统不去自动根据变更时间之类的去计算真正该执行的次数，因为有各种case不能保证算的是准的。有没有执行完毕，由用户自己判断。


### 按时间段展示的具体任务维度的信息

整体要求是，保证所提供的信息能精确的反映历史和将来的执行预期。


* 已经执行的，按任务实例的维度显示任务拓扑逻辑，便于区分执行链路。
  * 需要选定一个具体的任务实例 
  * 即多次失败原地retry这种，是一个拓扑节点
  * 非原地retry的，是另外的独立拓扑节点
  * offset多个依赖的，多个父亲task实例是不同的拓扑节点

* 未执行的，用Job节点显示拓扑逻辑，这个逻辑有点困难，下面只是可能的一些想法，要好好设计
  * 如果是无时间设定的纯依赖任务，父子任务都显示Job，但是可以展开所取时间段的所有可能任务，执行的，未执行的，可以展开选择一条链路？

类型|历史上的，但是未调度的|已完成|重跑的|未来要调度的|
---|---|---|---|---|
无依赖任务|最近一次完成的任务以前的任务，只根据执行流水展示|执行流水|执行流水多条||
单亲纯依赖|根据执行流水，显示依赖逻辑||||
Cron＋offset依赖|||||


### 和现有业务的兼容分析：

* 以上逻辑能满足现在按天执行一次的hive任务的进度监控（只要重跑任务再统计展示时加以区别对待）
* 现有短周期任务的执行情况能更好的加以支持


## 如何进行非计划临时变更操作？

操作|场景描述|要求|实现方式|界面操作逻辑|后端交互逻辑|
---|---|---|---|---|---|
A 单个任务原地重跑|任务失败，后续任务也没跑，希望重跑该任务|成功后，后续任务要能继续按原调度逻辑运行|重置当前任务为Ready状态？|用户怎么操作？|怎么触发Jarvis Server感知改任务需要再调度？需要直接的Task操作接口的REST API？|
B 指定数据日期,单个（或多个独立）作业立刻运行|比如某个任务产出有问题，只是自己需要再跑一次|不依赖前置作业，不触发后续依赖作业| 可以考虑借用D方案实现|||
C 指定数据日期,单个（或多个独立）作业在指定时间运行|同上，只是不打算立刻运行|同上|同上|||
D 指定数据日期,某个任务开始全链路作业再跑一次|某个任务产出有问题，影响了后续表，需要全部重跑|要求后续依赖作业对应周期的任务需要再被触发|依靠DAG task调度模块，专门处理这类一次性非计划执行链路，这类任务可能需要用特殊的任务信息进行标识，这样便于在完成时进行识别，避免和正常计划类型的task互相干扰调度逻辑，本质上还是在保留Job信息的情况下，区分出周期计划任务和非计划任务，进行执行逻辑的隔离|||
E 指定数据日期,某个任务开始部分链路作业再跑一次|有这种情况？|||||
F 新提交的一次性作业|如Ironman等应用程序提交的作业|可能需要定期清除，或者需要一个添加到Task记录里面以后即刻清除的机制？|和作业重run几乎一致，只是需要完整的Job信息，因此现在的设计，应该需要先落Job信息（主要是Job自身执行内容等需要存储）然后发起一次同D类似的调度，考虑怎么和D的方案简单的兼容实现。|||


有时可能要在任务未完成时，进行单次执行计划的变更：

操作|任务暂停(后续需要恢复)|任务取消（后续任务不执行）|任务跳过（后续任务需要执行）|
---|---|---|---|
已经开始执行的任务|不支持|杀掉，如果需要标失败（防止自动retry？或者流程没法检测失败的？）|杀掉，标假成功？|
依赖满足，生成任务实例，但尚未执行的任务|？|？|？|
依赖未满足的任务(单个)|Job标Pause状态|直接生成Canceled任务记录|生成Success，标记reason为skip，触发DAG调度|
依赖未满足的任务(指定链路)|链路起始job标Pause状态|批量生成Canceled任务记录|没有这个场景组合|
依赖未满足的任务(批量)|批量标Pause状态|批量生成Canceled任务记录|批量生成Success，标记reason为skip，触发DAG调度|

* 本质上是需要一个手段让将来的调度，或者状态恢复重建时，能识别出这种手工取消的任务。
  * 反向Schema也是一种可能的方式
  * 另一种可能方式是用Schedule的lastCheckTime域，标记到下一次的时间点。但是这样就要求拓展这个字段的业务含义，在DAG在调度的时候还要检查超过这个时间点的任务也不执行。。。
* 无crontime的纯依赖任务估计可以不用标记，一是可能没法标，二是它的状态恢复仅依赖于父任务，不依赖于自身的历史实际未执行任务。


## 其它问题


### 并发逻辑

* 一个作业，相同数据日期的任务是否允许并发？
  * 自动生成的周期任务，不允许，实际上也不应该出现。
  * 人工触发重跑的一次性任务，提示？不限制。
* 一个作业，不同数据日期的任务是否允许并发？
  * 如果是自依赖任务，按下面的逻辑，是不能并发的。
  * 如果没有配置为自依赖任务，应该允许并发
  * 人工触发重跑的一次性任务，如果是自依赖任务，给予提示，要强制跑，也应该允许。

### 自依赖任务逻辑？

* 上个周期的任务没成功完成，下一个周期的任务要block，这个逻辑怎么支持？需要哪些数据和流程逻辑支持？怎么检测？
  * 其它关系满足的话，直接添加一条调度流水，标记为Failed，失败reason填自依赖检查失败
    * 这样做，而不是block着等前面的任务的目的是，这件事情本身有一个直接的明确的失败结果产出，不依赖于前面的任务是否真的会成功，如果前面的任务因为某种原因，永远没有结果，不至于始终状态不明，状态比较可控，如果要重跑也有task可以跑，靠人工恢复。反过来如果采用调度器这里block的方式，而不是直接失败，那么想要忽略block，就很难处理了
    * 如果要做得更好，达到和block同样的效果，在上个周期的任务跑成功了以后，可以检测后面一个周期是否存在没有跑成功，最后状态是自依赖失败的case，自动触发重跑就好。


### 调度时间／数据时间如何判定?


* 目前老系统的应用场景 : 以天为单位执行的任务，需要指定数据日期，传参给脚本。执行日期和数据日期可能不一致。
  * $YTD这样的占位符在调度的时候，用数据日期来替换，如果没有填数据日期，那么就是执行时间减一天。
  * 正常调度作业，普通的没有指定数据日期的，按照计划调度时间计算YTD，计划调度时间在Task调度实例被生成时计算固化下来。
  * 原地重跑任务，计划调度时间不受影响，所以没有关系。
  * 指定数据时间的重跑任务，需要在最后生成的task中存储具体数据时间信息字段
  * 周期调度任务作业不支持指定具体数据时间（指定的问题是，实际这种设定对应的是正确的场景的可能性很小，太容易犯错）
 
依赖任务调度时间参数，没有指定，需要推算的，怎么确定和传递？

类型|计划调度时间|实际调度时间|数据时间|
---|---|---|---|
无依赖任务|crontime|crontime被检测到条件满足的时刻|crontime|
单亲纯依赖|父任务计划调度时间|依赖满足时间|父任务计划调度时间|
Cron＋offset依赖|crontime|依赖满足时间|crontime|



## 其它低优先级遗留问题

* 为了区分ready和失败后retry的ready，是否需要要有一个retry的状态？或者retry次数就够了？
  * 暂不区分。 


## 操作场景case

请根据前面的需求描述和正确处理逻辑，去构建各种正常，异常case。


### 自主调度case发现？

场景可能没法完全枚举，那么，有没有可能用model based test的思维，用基础原子的操作，随机任意组合，生成random覆盖的测试，长期持续的跑？（简单的说就是自动随机模拟用户行为，只要原子操作合理，都是合理的输入操作组合，不应该产生系统状态不一致或不正确输出行为。）

