# 网络协议

## 游戏基本流程

**9路棋盘**

1. 客户端连接服务端
2. 客户端向服务端发送 `READY_OP` 表示申请对局
3. 服务端向客户端回复 `READY_OP` 或 `REJECT_OP` 表示同意或拒绝对局（`REJECT_OP` 相当于重置到第二步开始前，要彻底断开连接，再不同意，应该使用 `LEAVE_OP`）
4. 双方交换 `READY_OP` 后，执黑方发送第一步棋 `MOVE_OP`，比赛开始
5. 双方轮流发送 `MOVE_OP`
6. 当一方认输、超时或违规，比赛结束
7. 胜方向败方发送 `GG_OP` 确认（替代本来到自己应该要发的 `MOVE_OP`）
8. 败方回复 `GG_OP` 确认（没有规定败方否认时的操作，所以此时你们应该用 `聊天功能` 来协调解决）
9. 任意一方发送 `LEAVE_OP` 后，可以断开连接（在这之前保持连接）或 **任意一方**发送 `READY_OP` 请求再来一局，相当于做了第二步

> 更具体的协议请仔细阅读以下内容，仔细按照协议通信。如果发现协议有漏洞（少考虑等），可以在 Issues 中反应，或微信和助教探讨。

## 网络协议

一个完整的请求包含一个操作符 `OP` 以及至多两个数据段 `data1` 和 `data2`。操作符指定了当前请求需要执行的操作，数据段则在此基础上给出了一些额外信息。

下面给出本次作业中可能用到的所有请求及其处理行为，"空"代表接收方可以不处理（但也要能接受为空，不能假设这个一定有值），发送方也可以置这个数据段为空。

### `READY_OP`

即是准备，也是申请。**请求对局方**要向对方发送自己希望执的颜色，对方同样回应 `READY_OP` 表示对局成立。

1. 建立连接成功时，客户端是请求对局方，也就是服务端在收到客户端任何通信前不与客户端进行交流。
2. 对局结束后，或暂时拒绝后，想再来一局（重发）的是请求对局方。另外，**如果已经收到对方想再来一局了，程序应该阻止本方重复发送请求，而引导先接受或拒绝**。你可以不处理因信息传输导致的理想状态下的同时发送，你可以假设不存在这种情况。

- `data1`: 己方用户名
- `data2`: 己方执棋颜色（回应方可以不发）

**用户名**：只能包含英文字母大小写、数字和下划线 `_`，不得包含空格、换行符、制表符等分隔符。

**颜色编码**：黑则 "`b`"，白则 "`w`"

### `REJECT_OP`

拒绝对局申请，向对方发送 `REJECT_OP`。之后可以发送 `LEAVE_OP` 离开，也可以交流信息后重新发 `READY_OP`，但不能直接离开。

- `data1`: 己方用户名
- `data2`: 拒绝理由（空）

### `MOVE_OP`

决定如何行棋后，向对方发送 `MOVE_OP`

- `data1`：落子点坐标
- `data2`：己方落子时间戳，己方从这个时间戳开始计时对方的超时。

坐标编码：用 `A1`，`I9` 这种表示，坐标轴方向同 [nogo.md](../../guidance/nogo/nogo.md) 中所示，双方不旋转棋盘，也就是视角是相同的。

时间戳：统一用 `QDateTime::currentMSecsSinceEpoch` 返回的结果。

> qint64 QDateTime::currentMSecsSinceEpoch()
> 
> Returns the number of milliseconds since 1970-01-01T00:00:00 Universal Coordinated Time. This number is like the POSIX time_t variable, but expressed in milliseconds instead.

时间戳意味着，假设超时时间是 3000 毫秒，例如我是黑方先手，我在发送 `MOVE_OP` 时获取当前时间是 10000（data2 的信息即设 10000），我的计时器就以此为头开始计时，如果 13000 这一刻我没有收到消息就给对方发超时。然后看白方，白方收到 `MOVE_OP`，data2 是 10000，但其实它收到的那一刻已经是 10500 了，但它仍然要在 13000 时保证能发到对方手中，所以你的计时器要变一下。原则上来说，你最好设倒计时剩余时间为 3000 - 2*(10500-10000)，也就是加上发回去对方收到的预计耗时。

### `GIVEUP_OP`

认输，替代 `MOVE_UP`，向对方发送 `GIVEUP_OP`

- `data1`：己方用户名（空）
- `data2`：问候语（空）

> 无处落子时及时认输才是君子之道

### `GG_OP`

`GG_OP` 并不是一个 `OP`，而是一组指定获胜原因的 `OP`。胜者先发，败者回复相同操作表示确认。

- `TIMEOUT_END_OP`: 一方超时
- `SUICIDE_END_OP`: 一方落子非法：即自杀，重复落子或**吃了对方**
- `GIVEUP_END_OP`: 一方认输

> 如果对方在对局中突然发送 `LEAVE_OP` 则不需要向对方回复任何信息，房间可以直接关闭。

他们的数据段含义是

- `data1`：己方用户名（空）
- `data2`：问候语（空）

### `LEAVE_OP`

任何时候，发送 `LEAVE_OP` 说明即将立刻断开连接。

- `data1`：己方用户名（空）
- `data2`：离开的原因（空）

> 任何时候，断开前都先发 `LEAVE_OP`，以区分意外断开。接受方可以不用回复，直接断开（不用回复，但一定要断开）。
>
> 这里的“断开”需要解释一下，对于我们的网络库，你可以理解成这里只能客户端主动断开服务端（也就是接口里的 bye），然后客户端甩手就走，不管服务端愿不愿意。反过来服务端要主动和客户端断开只能约定一个协议，服务端给客户端发一个特定消息后（比如这里的 LEAVE_OP），客户端再同上主动离开（bye），然后服务端收到这个这个信号后才会离开（对应槽函数 leave）。具体的 TCP 协议怎么处理这些问题远远超过了本课要求，所以你不需要深究。
>
> 所以你实现这个操作的全过程大概会像这样：
> 
> - 客户端主动离开时先发 `LEAVE_OP` 再 `bye`，服务端收到 `bye` 后会自动触发槽函数 `leave` 断开。
> - 服务端主动离开先发 `LEAVE_OP`，客户端收到后可选地回复 `LEAVE_OP` 然后必须执行 `bye`，服务端收到 `bye` 后自动触发槽函数 `leave` 断开。
> - 为了简单，**你可以假设**服务端一定会收到客户端回复的 `bye`，但如果真的没收到，你可以在一开始打算主动断开时就从逻辑上断开连接，也就是以后收到这个 client 发的信号都直接 return 不处理。或者你可以自己研究网络协议和 Qt 的原始网络库设计更合理的方法来处理，但这远远超过了本课程的要求范围。

### `CHAT_OP`

只要保持连接，就可以发送 `CHAT_OP` 聊天或交换信息。

- `data1`：信息内容
- `data2`: 空

> 上面所提到的特殊编码（颜色，坐标，用户名）均不能莫名其妙包含空格，制表符，换行等无关字符，保证编码严格统一。
