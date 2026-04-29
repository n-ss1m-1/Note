基于 Qt 的航空订票系统项目开发报告
1.项目概述
本项目开发了一套桌面端的航空订票系统，用来模拟真实的订票业务流程。系统采用客户端 + 服务端的结构：
客户端（Qt 桌面应用）：主要负责界面展示和用户交互，例如登录/注册、航班查询条件输入、表格展示、选择乘机人、下单与支付操作入口、订单详情与状态展示等。客户端通过网络向服务端发送请求，并根据服务端返回的数据刷新界面。
服务端（Qt/C++ 后端程序）：主要负责业务处理和数据管理，例如用户认证、航班查询、订单创建、支付/取消/改签/退票规则校验、库存余票更新、订单状态流转等，并将数据持久化到 MySQL 数据库。
客户端与服务端之间使用 QTcpSocket 进行通信，统一采用自定义协议（JSON 格式）传输请求与响应。通过这种分层方式，界面逻辑与业务逻辑分离，便于后续扩展与维护。
1.1主要参加成员
组长：李佩轩 24336061
成员：陈业伟 24336017    赵明辉 23340080    AUNGLINHTET 24952080
1.2业绩权重
后两位均分（1 1 2 2）
1.3工作任务的分解和人员分工
李佩轩：总体规划、架构设计、客户端Qt主界面框架、核心数据结构、客户端UI与核心功能开发、进度协调、系统集成、报告整理。
陈业伟：MySQL 表设计与搭建、ODBC连接配置、数据库访问层封装与调试、服务端业务逻辑开发、网络通信集成调试。
赵明辉：客户端UI与部分功能开发。
AUNGLINHTET：服务端UI与航班/订单/用户管理功能开发。
1.4技术难点
自定义 TCP 文本协议的粘包/拆包处理：所有消息以 '\n' 结尾分隔，客户端与服务端均用缓冲区按行解析，降低粘包导致的解析失败风险。
统一模型与协议常量：使用公共模块统一客户端与服务端的通信协议与数据结构，避免前后端字段不一致。
订单相关的事务与一致性：创建订单/取消订单/改签涉及航班余座与订单状态的联动，服务端使用事务控制关键步骤，尽量保证数据一致。


2.业务需求与基本功能
2.1 业务需求
面向“用户在线订票”场景，核心流程包含：
1.用户注册并登录系统；
2.用户维护个人信息与常用乘机人；
3.用户按条件查询航班并选择航班；
4.用户为某乘机人创建订单；
5.用户支付订单或取消订单；
6.用户查看订单列表，可支付未支付订单；
7.用户可对本人订单执行改签，并处理差价的待支付金额。
2.2基本功能清单
账号与个人信息：登录、注册、退出登录、修改密码、修改手机号。
常用乘机人：查询、添加、删除。
航班：查询城市列表；按条件查询航班（出发/到达、日期范围、时间范围、价格范围等条件，允许不填写）。
订单：创建订单、支付订单、查询订单列表（包括本账号和本人信息）、取消订单、改签订单。

3.系统设计与实现
3.1计算机体系设计
开发平台：Qt 6（Windows 环境）。
客户端：Qt Widgets（页面 + 对话框），通过 QTcpSocket 与服务端通信。
服务端：Qt Network，解析 JSON 并调用数据库层。
数据库：MySQL，通过 Qt SQL + ODBC 驱动连接（QODBC）。
通信格式：JSON 文本；每条消息以 '\n' 作为结束符，便于按行拆包。
3.2数据库字典
用户表 (user)：id(主键), username, password, phone, real_name, status。
航班表 (flight)：id(主键), flight_no, from_city, to_city, departure_time, arrival_time, price_cents, seat_total, seat_left等。 
订单表 (orders)：order_id(主键), user_id(外键), flight_id(外键), passenger_name(乘机人信息), passenger_id_card, order_status(待支付/已支付/已改签/已退票), total_amount, create_time等。
常用乘机人表 (passenger)：id, user_id(外键), id_card, name。
3.3主要模块编码实现
1) 公共协议与数据模型（Common）
Protocol.h：统一通信字段与 type
顶层字段固定为：type / data / success / message
统一定义所有请求/响应类型字符串，例如：
登录：login / login_response
航班：flight_search / flight_search_response
订单：order_create / order_create_response 等
提供 makeOkResponse / makeFailResponse，服务端构造响应时不用到处拼字段，降低前后端不一致风险。
Models.h：统一结构体与 JSON 互转
定义并统一字段：
FlightInfo / UserInfo / PassengerInfo / OrderInfo
OrderStatus / FlightStatus 枚举
提供 flightFromJson / orderFromJson / passengerFromJson / userFromJson 及对应的 ToJson 函数，用于统一数据结构与 JSON 转换。
2) 客户端：
网络通信模块NetworkManager
1 单例与基础连接管理
使用 NetworkManager::instance() 单例，全客户端共用一个 TCP 连接。
构造函数中连接 socket 信号：
oreadyRead → 读取数据并拆包
oconnected / disconnected / errorOccurred → 通知 UI 或触发强制登出
2 发送：sendJson（统一换行分隔）
sendJson() 将 QJsonObject 转成 compact JSON，并 追加 '\n' 作为消息分隔符。
若未连接：
o已登录：触发 handleUnexpectedDisconnect()，清理登录态并发 forceLogout
o未登录：发 notConnected() 给 UI 做提示
3 接收：缓冲区拆包（processBuffer）
onReadyRead() 将 readAll() 追加到 m_buffer
processBuffer() 循环查找 '\n'，每次截取一行 JSON 解析：
o成功：emit jsonReceived(doc.object())
o失败：打印 parse error（不中断连接）
4 登录态与异常断连处理
维护 m_loggedIn / m_username / m_userInfo
handleUnexpectedDisconnect()：
o防重复处理（m_disconnectHandled）
o区分手动断开与意外断开
o意外断开：清理 session，发 forceLogout(reason)，由各页面关闭弹窗/刷新 UI
5 常用请求封装
login/registerUser/changePassword/logout 直接构造 type + data 并调用 sendJson()
logout() 发送 TYPE_LOGOUT（若已连接），然后本地清理登录态
个人页面模块ProfilePage
1 登录/退出 UI 刷新
updateLoginUI()：根据 NetworkManager::isLoggedIn() 切换按钮文案与状态显示
updateUserInfoUI()：
o未登录：显示 - 并禁用“修改手机号”
o已登录：启用并调用 applyPrivacyMask()
2 隐私脱敏显示（勾选框控制）
maskMiddle(s, left, right, '*')：保留左右固定长度，中间替换为 *
applyPrivacyMask()：
o手机号：3-4-4
o姓名：仅最后一个字可见
o身份证：前4后4可见
三个 checkbox 的 toggled 槽函数仅在登录后刷新显示
3 “弹窗 + 一次性监听回包”模式（避免全局误处理）
登录/注册/改密等弹窗：发送请求前临时 connect jsonReceived
回包到达后：
o只处理目标 type（或 error）
o立即 disconnect 这个连接，避免后续其它消息误触发
这样做的效果：只有在“主动发请求”的那段时间才处理该响应，避免多个页面同时弹错误框的冲突。
4 强制登出时关闭弹窗
bindForceLogoutToDialog(QDialog*)：
o连接 NetworkManager::forceLogout，强制登出时 dlg->reject()
o防止网络断开后用户仍停留在不可操作弹窗里
5 常用乘机人列表（QTableWidget）
initPassengerTable()：三列（隐藏 ID 列），行选择模式，禁止编辑
requestPassengers()：
o先断开旧的 m_passengerConn
o临时 connect jsonReceived：
TYPE_PASSENGER_GET_RESP：解析 passengers 数组，填充表格
TYPE_ERROR：
若 message 含“暂无”，视为正常空列表（不弹严重错误）
其他错误弹框
o处理完成后断开 m_passengerConn
6 添加/删除乘机人
添加：
oAddPassengerDialog 提交后先做输入校验（姓名不能含字母数字、身份证正则、禁止添加本人信息）
o发 TYPE_PASSENGER_ADD
o收到 TYPE_PASSENGER_ADD_RESP 后提示并 requestPassengers() 刷新
删除：
o获取当前选中行的真实姓名/身份证（表格显示可能是脱敏的）
o发 TYPE_PASSENGER_DEL
o收到 TYPE_PASSENGER_DEL_RESP 后提示并刷新
7 本地设置（主题/缩放）
主题下拉框绑定 SettingsManager::setThemeMode
缩放滑块绑定 SettingsManager::setScaleFactor
属于纯客户端 UI 体验功能，不依赖服务端
航班页面模块FlightsPage
1 表格与缓存
使用 QStandardItemModel + QTableView
m_flightCache[flightId] = FlightInfo：用于下单后快速拿到航班信息（给订单详情弹窗用）
2 城市列表：进入页面自动刷新
构造时与 showEvent() 都会调用 requestCityList()
收到 TYPE_CITY_LIST_RESP：
o更新出发/目的地 combo（带“不限”）
o尽量保留用户当前已选项（oldFrom/oldTo）
m_waitingCityListForSearch 用于区分：
o只是刷新城市
o或者“为了查询航班而先拉城市”（城市回包后自动继续发 flight_search）
3 航班查询：所有条件允许不填
点击查询时读取 UI：
o城市“不限”转为空
o日期范围/时间范围勾选后才纳入条件，并做 min/max 合法性校验
sendFlightSearch()：
o不指定的条件不 insert 到 JSON（服务端按存在字段动态拼查询）
odate 条件打包进 data["date"] 子对象（min/max date/time）
4 查询回包处理与空结果处理
TYPE_FLIGHT_SEARCH_RESP：
o先清空旧 model
o若 flights 数组为空：提示“未找到航班”
o否则逐行填充并更新缓存
对查询时返回 TYPE_ERROR 且 message 含“暂无”：视为无结果提示；否则提示查询失败
5 订票流程：先选乘机人再下单
双击表格行或点订票：
1.校验登录、校验选中航班
2.记录 m_pendingBookFlightId
3.发送 TYPE_PASSENGER_GET
4.等 TYPE_PASSENGER_GET_RESP 后弹出 PassengerPickDialog
5.用户确认后调用 sendCreateOrder(flightId, name, idCard)
兼容“无常用乘机人”：
o若 passenger_get 以 TYPE_ERROR 返回且 message 含“暂无常用乘机人”
o仍然允许选择“本用户信息”下单
6 下单后立即支付
收到 TYPE_ORDER_CREATE_RESP：
o解析 order，提示下单成功
o询问是否立即支付
订单页面模块OrdersPage
1 订单数据来源与合并去重
loadOrders() 同时发送两个请求：
oTYPE_ORDER_LIST（本账号下的订单）
oTYPE_ORDER_LIST_MY（与本人实名信息一致的订单）
使用 m_pendingOrderListResp = 2 计数，确保两个响应都回来后再统一展示
mergeOrdersFromArray(arr, fromUserList)：
o每个数组元素是 { order: {...}, flight: {...} }
o以 orderId 为 key 写入 m_orderCache（去重）
o同时标记来源 fromUser / fromOther
2 表格展示与排序
rebuildTableFromCache()：
o将 cache 转 list，按 createdTime 倒序排序
o每行显示：订单来源、状态、航班号、航线、起飞到达、乘机人、座位号、金额
o在第一列 item 里用 ROLE_ORDER_ID 存订单号（不单独占一列）
3 订单状态文本与颜色
statusToText()：Booked=待支付、Paid=已支付、Rescheduled=已改签、Canceled=已退票、Finished=已完成
statusToColor()：待支付橙色、已支付绿色、其余灰色
颜色通过给状态列 item 设置 Qt::ForegroundRole 实现
4 双击进入订单详情并联动刷新
双击行 → 取 ROLE_ORDER_ID → 从 m_orderCache 拿到 OrderInfo + FlightInfo
打开 OrderDetailDialog，并连接：
oorderPaid / orderCanceled 信号 → loadOrders() 刷新订单列表
3) 服务端：
客户端连接请求处理器ClientHandler
1数据接收与拆包（解决粘包）
onReadyRead()：
om_buffer.append(m_socket->readAll()) 将本次收到的数据追加到缓冲区
o调用 processBuffer() 进行拆包
processBuffer()：
o以 '\n' 为消息结束符循环查找：m_buffer.indexOf('\n')
o每次取出一行作为一条完整 JSON（QJsonDocument::fromJson）
o解析成功且为 object：调用 handleJson(doc.object())
o解析失败：输出 qWarning()，不会断开连接（容错）
2请求分发与校验（handleJson）
handleJson(const QJsonObject &obj)：
o读取 type 与 data：
const QString type = obj.value(Protocol::KEY_TYPE).toString();
const QJsonObject data = obj.value(Protocol::KEY_DATA).toObject();
o获取服务端单例：
DBManager& db = DBManager::instance();
OnlineUserManager& userManager = OnlineUserManager::instance();
o大多数业务分支都会先做“是否已登录”的校验（isLoggedIn()），避免未登录伪造请求。
3用户登录/退出/注册/改密/改手机号
登录 TYPE_LOGIN
o从 data 取 username/password
odb.getUserByUsername(username, user, &errMsg) 查询用户
o校验 user.password == password
o成功：
setUserInfo(user) 保存到当前 handler
isLogin=true; emit loginSuccess(); 通知在线用户管理模块登记
回包：TYPE_LOGIN_RESP，并将完整 user JSON 回传
o失败：回 TYPE_ERROR（“账号或密码错误”）
退出 TYPE_LOGOUT
o必须已登录
ouserManager.getUserInfoByHandler(this) 打日志
o清空 handler 内 m_userInfo，并 userManager.removeOnlineUser(this)
o回包 TYPE_LOGOUT_RESP
注册 TYPE_REGISTER
o从 data 取 username/password/phone/realName/idCard
o做基础输入校验（非空、手机号正则、身份证 18 位）
o仅检查“用户名是否存在”：db.getUserByUsername
o成功调用 db.addUser(...) 插入用户
o回包 TYPE_REGISTER_RESP；失败回 TYPE_ERROR
修改密码 TYPE_CHANGE_PWD
o必须已登录
o从 data 取 username/oldPassword/newPassword
o先 db.getUserByUsername 校验旧密码，再 db.updatePasswdByUsername 更新
o更新成功后同步更新：
this->m_userInfo.password=newPwd
userManager.addOnlineUser(this)（覆盖更新在线映射）
o回包 TYPE_CHANGE_PWD_RESP 或 TYPE_ERROR
修改手机号 TYPE_CHANGE_PHONE
o必须已登录
o从 data 取 username/newPhone，做非空与手机号正则校验
odb.updatePhoneByUsername 更新
o更新成功后同步 handler 与在线表（同上）
o回包 TYPE_CHANGE_PHONE_RESP 或 TYPE_ERROR
4常用乘机人：查询/添加/删除
查询 TYPE_PASSENGER_GET
o必须已登录
o从 OnlineUserManager 拿当前 user.id
odb.getPassengers(user.id, passengers, &errMsg)
o成功：passengersToJsonArray 写入 data.passengers，回 TYPE_PASSENGER_GET_RESP
o无数据：回 TYPE_ERROR（“暂无常用乘机人”）
添加 TYPE_PASSENGER_ADD
o必须已登录
odata 取 passenger_name/passenger_id_card（身份证转大写）
odb.addPassenger(user.id, name, id, &errMsg)
o成功回 TYPE_PASSENGER_ADD_RESP，失败回 TYPE_ERROR
删除 TYPE_PASSENGER_DEL
o必须已登录
odb.delPassenger(user.id, name, id, &errMsg)
o成功回 TYPE_PASSENGER_DEL_RESP
5航班：条件查询与城市列表
航班查询 TYPE_FLIGHT_SEARCH
o必须已登录
o将 data 解析到 Common::FlightQueryCondition cond
fromCity/toCity
date 子对象：minDepartDate/maxDepartDate/minDepartTime/maxDepartTime
minPriceCents/maxPriceCents
o做日期区间合法性校验（max < min 返回错误）
odb.searchFlights(cond, flights, &errMsg)
成功回 TYPE_FLIGHT_SEARCH_RESP，data.flights + data.count
无数据回 TYPE_ERROR（“暂无相关航班”）
城市列表 TYPE_CITY_LIST
odb.getCityList(fromCities, toCities, &errMsg)
o成功回 TYPE_CITY_LIST_RESP，分别返回 fromCities/toCities
6订单：创建/支付/查询/改签/取消
创建 TYPE_ORDER_CREATE
o必须已登录
o从 data 取 flightId/passengerName/passengerIdCard，并设置 order.userId=user.id
odb.createOrder(order, &errMsg)
成功回 TYPE_ORDER_CREATE_RESP，data.order 包含创建后的订单（含订单号、座位号、价格等）
支付 TYPE_ORDER_PAY
o必须已登录
o从 data 取 orderId
odb.payForOrder(orderId, &errMsg)
o回 TYPE_ORDER_PAY_RESP
查询（账号维度）TYPE_ORDER_LIST
o必须已登录
odb.getOrdersByUserId(user.id, ordersAndflights, &errMsg)
o成功回 TYPE_ORDER_LIST_RESP，返回 ordersAndflights（每项含 order + flight）
查询（本人信息维度）TYPE_ORDER_LIST_MY
o必须已登录
odb.getOrdersByRealName(user.realName, user.idCard, ...)
o成功回 TYPE_ORDER_LIST_MY_RESP
改签 TYPE_ORDER_RESCHEDULE
o必须已登录
odata.oriOrder 解析为 oriOrder，并从 data 取新航班与乘机人信息组成 newOrder
o校验：oriOrder.userId == newOrder.userId，否则拒绝（“无权限改签他人订单”）
odb.rescheduleOrder(oriOrder, newOrder, priceDif, &errMsg)
o成功回 TYPE_ORDER_RESCHEDULE_RESP，返回 data.order（新订单）与 data.priceDif（差价）
取消 TYPE_ORDER_CANCEL
o必须已登录
o从 data 取 orderId
odb.cancelOrder(orderId, &errMsg)
o成功回 TYPE_ORDER_CANCEL_RESP
7）回包与连接释放
sendJson(const QJsonObject &obj)：
oQJsonDocument(doc).toJson(Compact) 并追加 '\n'，写回 socket（与客户端拆包逻辑一致）
onDisconnected()：
o打印断开日志
oOnlineUserManager::removeOnlineUser(this)
om_socket->deleteLater(); deleteLater(); 释放连接对象与 handler
数据库访问与核心事务逻辑DBManager
1连接管理与驱动兼容
构造函数：QSqlDatabase::addDatabase("QODBC")
connect(host, port, user, passwd, dbName)：
o通过 ODBC 连接字符串连接 MySQL
o在多个驱动名中尝试：
"MySQL ODBC 8.0 Unicode Driver"
"MySQL ODBC 9.5 Unicode Driver"
o连接失败时回传 lastError() 文本到 errMsg
2通用 SQL 封装
Query(sql, params, errMsg)：
o检查连接有效
oprepare + 位置绑定 bindValue(i, params[i])
oexec 失败输出日志并设置 errMsg
o返回 QSqlQuery（即便失败也返回，用 query.isActive() 判断）
update(sql, params, errMsg)：
o复用 Query 执行
o返回 query.numRowsAffected() 作为影响行数（用于判断更新是否成功）
3事务控制
beginTransaction() → db.transaction()
commitTransaction() → db.commit()
rollbackTransaction() → db.rollback()
在订单创建、取消、改签等需要保证一致性的场景中使用事务，避免中途失败导致“订单写入了但座位没扣/或扣了座位但订单没写入”。
4 ResultSet 到模型的转换
userFromQuery()：读取 user 表字段 id/username/password/phone/real_name/id_card
passengerFromQuery()：读取 passenger 表字段 id/user_id/name/id_card（其中 user_id 在代码中以 userId 取值，需与实际字段别名一致）
flightFromQuery(query, prefix)：
o支持 join 查询时使用别名前缀（例如 f_id/f_flight_no/...）
o解析 depart_time/arrive_time 为 QDateTime
ostatus 转为 Common::FlightStatus
orderFromQuery(query, prefix)：
o同样支持 join 前缀（例如 o_id/o_user_id/...）
o解析 pending_payment/status/created_time
5用户相关数据库操作
getUserByUsername(username, user)：select * from user where username=?
getUserById(userId, user)：select * from user where id=?
addUser(...)：insert into user (username,password,phone,real_name,id_card) ...
updatePasswdByUsername(username, newPasswd)
updatePhoneByUsername(username, phone)
6常用乘机人相关操作
addPassenger(user_id, name, id_card)：
o先查重：select * from passenger where user_id=? and name=? and id_card=?
o若存在则返回失败（避免重复添加）
o不存在则插入：insert into passenger (user_id,name,id_card) values(?,?,?)
delPassenger(user_id, name, id_card)：精确条件删除
getPassengers(user_id, passengers)：
oselect * from passenger where user_id=?
o遍历结果集填充列表，空则返回 DBResult::NoData
7航班查询（动态拼接 where 条件）
searchFlights(const FlightQueryCondition& cond, QList<FlightInfo>& flights)：
o基础 SQL：select * from flight
o使用 whereClauses + params 动态拼接条件：
id=?
from_city=?
to_city=?
日期范围（使用 date(depart_time)）
时间范围（使用 time(depart_time)）
价格范围 price_cents>=?、price_cents<=?
强制追加 seat_left>0（只返回有余座航班）
o最后 order by depart_time asc
o返回 Success/NoData/QueryFailed
8城市列表
getCityList(fromCities, toCities)：
oselect distinct from_city from flight
oselect distinct to_city from flight
o两边都空则 NoData
9订单核心逻辑（事务重点）
1）创建订单 createOrder(OrderInfo& order, bool autoManageTransaction)
若 autoManageTransaction=true 则内部自行 begin/commit/rollback，便于单独调用。
核心步骤：
1.扣减余座：update flight set seat_left=seat_left-1 where id=? and seat_left>0
影响行数为 0 表示余座不足或更新失败 → 回滚
2.查询航班信息（用 searchFlights(cond.id=flightId)）
3.分配座位号：seatNum = seatTotal - seatLeft + 1
4.初始化金额：priceCents = pendingPayment = flight.priceCents
5.初始化状态：Booked
6.插入订单：insert into orders (...) values (...)
7.取新订单号：lastInsertId()
8.提交事务（失败则回滚）
2）支付订单 payForOrder(orderId)
直接更新：
ostatus=Paid
opending_payment=0
3）查询订单（带航班信息 join）
getOrdersByUserId(userId, ordersAndflights)
getOrdersByRealName(realName, idCard, ordersAndflights)
两者都使用 orders o inner join flight f，并为字段起别名：
o订单字段 o_...
o航班字段 f_...
通过 orderFromQuery(prefix) / flightFromQuery(prefix) 解析并组装 QPair<OrderInfo, FlightInfo>
4）改签 rescheduleOrder(oriOrder, newOrder, priceDif)
必须事务包裹（内部固定 begin/commit/rollback）
实现策略：
1.计算原订单已支付金额：oriPaidAmount = oriPrice - oriPendingPayment
2.取消原订单：cancelOrder(oriOrder.id, autoManageTransaction=false)
这里传 false 是为了复用同一事务，不在 cancelOrder 内部重复 begin/commit
3.创建新订单：createOrder(newOrder, autoManageTransaction=false)
4.计算差价：priceDif = newOrder.priceCents - oriPaidAmount
5.新订单待支付：newOrder.pendingPayment = max(0, priceDif)
并写回数据库：update orders set pending_payment=? where id=?
6.将新订单状态改为 Rescheduled 并写回数据库
7.commit（失败 rollback）
5）取消订单 cancelOrder(orderId, bool autoManageTransaction)
若 autoManageTransaction=true 则内部自行事务；否则用于被改签逻辑包裹。
核心步骤：
1.更新订单为取消并清零待支付：
update orders set status=Canceled, pending_payment=0 where id=? and status not in(Canceled, Finished)
防止重复取消或取消已完成订单
2.查询该订单关联航班：select flight_id from orders where id=?
3.恢复余座：update flight set seat_left=seat_left+1 where id=?
4.commit

4.系统测试
以功能测试与异常场景测试为主，覆盖主要业务链路与边界条件。
登录/注册：正确账号密码登录成功；错误密码返回错误；手机号/身份证格式校验。
常用乘机人：添加成功；重复添加提示；删除后列表更新；未登录请求被拒绝。
航班查询：只填部分条件可返回；日期区间不合法时返回错误；无航班返回提示；仅返回有余座航班（seat_left>0）。
订单流程：下单成功后 seat_left 减少；支付后 pending_payment=0、状态变化；取消订单后 seat_left 恢复；已取消/已完成的订单不可再次取消。
改签：改签后生成新订单并返回差价；差价为正时出现待支付金额；差价为负时待支付为 0；尝试改签他人订单返回无权限。
协议与稳定性：多条 JSON 连续发送时能正确拆包；非法 JSON 不导致服务端崩溃


5.项目展望
项目后续继续完善，较优先的方向包括：
1.安全性：密码哈希存储；敏感信息传输与显示进一步规范。
2.并发与一致性：进一步强化余座扣减策略（例如更严格的并发控制与失败回滚检查）。
3.业务完整度：补充订单完成、退款到账等更完整的状态流转；改签避免同航班同时间等规则在前后端统一校验。
4.可用性：航班/订单列表分页与排序、错误提示更统一、日志与配置文件化。
5.可维护性：将服务端按“路由分发/业务服务/数据访问”进一步解耦，减少单文件分支过长的问题。