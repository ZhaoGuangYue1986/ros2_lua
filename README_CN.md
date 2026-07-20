# ros2_lua 学习笔记

## 项目概述

**ros2_lua** 是一个让开发者用 **Lua** 语言编写 ROS2 节点的开源工程（基于 ROS2 Humble + Lua 5.3+）。它将 ROS2 的 `rcl` 核心库封装为 Lua 可调用的接口，并在上层用纯 Lua 封装出类 Python/C++ 风格的 Node、Executor、Action、Lifecycle 等高级抽象。

- 作者: Stanislav Mikhel
- 许可证: Apache License 2.0
- GitHub: https://github.com/mikhel1984/ros2_lua

---

## 一、整体架构：双层设计

核心思想是**分离关注点**——C 语言做高性能绑定，Lua 语言做灵活封装：

```
                用户 Lua 脚本
                     │
        ┌────────────┴────────────┐
        │    纯 Lua 封装层         │  ← 面向用户的类风格 API
        │  (Node, Executor,       │     协程式执行器、Future 异步
        │   ActionClient 等)      │
        └────────────┬────────────┘
                     │ require("rcllua.rclbind")
        ┌────────────┴────────────┐
        │    C 绑定层 (rclbind.so) │  ← Lua C API 桥接
        │   16个C文件，每个对应    │     直接调用 rcl 函数
        │   一个 ROS2 概念         │
        └────────────┬────────────┘
                     │
        ┌────────────┴────────────┐
        │   ROS2 rcl 底层库        │  ← rcl, rcl_action, rcl_lifecycle
        └─────────────────────────┘
```

---

## 二、七大子项目职责

| 子项目 | 角色 | 关键产出 |
|---------|------|----------|
| **rcllua** | 核心引擎 | `rclbind.so`(C绑定) + 9个Lua类文件 |
| **rosidl_generator_lua** | 消息代码生成器 | Python包，将 `.msg/.srv/.action` → C代码 → `.so` |
| **rosidl_luacommon** | 序列类型基础 | 基本类型的 C 序列容器 |
| **rcllua_std_msgs** | 标准消息绑定 | 为 geometry/std/sensor/nav_msgs 等生成 Lua 绑定 |
| **rcllua_cmake** | 构建工具链 | 3个CMake宏：安装脚本/库/可执行文件 |
| **rcllua_examples** | 示例代码 | 高层 + 底层两套示例 |
| **rcllua_unit** | 测试框架 | 轻量 Lua 断言 + CTest 集成 |

### 依赖关系图

```
rosidl_generator_lua  ──→  rosidl_luacommon
    │                           │
    └──────→ rcllua_std_msgs ←──┘
                  │
                  ▼
    rcllua_cmake ──→  rcllua (rclbind.so + Lua脚本)
                           │
                  ┌────────┼────────┐
                  ▼        ▼        ▼
           rcllua_examples  rcllua_unit
```

---

## 三、六大核心设计模式

### 1. 基于原型的类系统

Node 使用 Lua 的元表实现类似类的继承：

```lua
local MyNode = Node {
  name = "my_node",
  init = function(self) ... end,        -- 构造函数
  callback = function(self, msg) ... end -- 回调方法
}
local node = MyNode()  -- 通过 __call 元方法实例化
```

### 2. 协程式执行器（核心亮点）

`Executor` 用 **Lua 协程**实现单线程协作式多任务：

```
spin_once() 启动协程：
  1. 收集所有节点的订阅/定时器/客户端/服务端
  2. 构建 rcl WaitSet 并阻塞等待
  3. 对就绪实体执行 coroutine.yield(fn) → 返回回调给主循环
  4. 主循环执行回调，然后 resume 协程继续等待
```

这避免了多线程的复杂性，同时保持了事件驱动的高效。

### 3. Future 异步模式

`client.lua` 实现了完整的 Promise/Future 模式：
- `call_async()` 返回 Future，支持 `:done()` 检查 + `:add_done_callback()` 注册回调
- 同步等待通过协程 `node:wait(condition, timeout)` 实现

### 4. Action 通信协议

完整实现了 ROS2 Action 的 5 通道协议（SendGoal/CancelGoal/GetResult + Feedback/Status），`ActionServer` 使用 `ServerGoalHandle` 管理目标生命周期。

### 5. 生命周期状态机

`LifecycleNode` 继承 `Node`，通过 `rcl_lifecycle` 实现了标准状态转换：

```
unconfigured → configuring → inactive → activating → active
                                                      ↓
                   finalized ← shuttingdown ← deactivating
```

### 6. 消息生成管道

```
.msg/.srv/.action 文件
    → Python 解析器 (__init__.py)
    → Empy C 模板 (msg.c.em)
    → 编译为 .so 动态库
    → Lua require("std_msgs.msg") 加载
```

---

## 四、使用指南

### 第一步：安装依赖

```bash
# 确保已安装 ROS2 Humble
# 安装 Lua 5.3+
sudo apt install lua5.3 liblua5.3-dev
```

### 第二步：编译安装 ros2_lua

```bash
source /opt/ros/humble/setup.bash
mkdir -p ~/ws_lua/src && cd ~/ws_lua/src
git clone -b humble https://github.com/mikhel1984/ros2_lua.git
cd ..
colcon build --symlink-install
```

构建完成后会生成：
- `rclbind.so` — C 层绑定共享库
- Lua 类库脚本 — Node、Executor、ActionClient 等
- 消息生成器 — 为 `.msg/.srv/.action` 自动生成 Lua 绑定

### 第三步：创建自己的 Lua ROS2 包

#### 3.1 创建包骨架

```bash
source ~/ws_lua/install/setup.bash
cd ~/ws_lua/src
ros2 pkg create --build-type ament_cmake my_lua_project
cd my_lua_project
mkdir my_lua_project   # Lua 脚本放同名子目录
```

#### 3.2 编写 package.xml

```xml
<package format="3">
  <name>my_lua_project</name>
  <version>0.0.1</version>
  <description>My Lua ROS2 project</description>
  <buildtool_depend>ament_cmake</buildtool_depend>
  <depend>rcllua</depend>
  <depend>rcllua_cmake</depend>
  <depend>std_msgs</depend>
</package>
```

#### 3.3 编写 CMakeLists.txt

三个关键 CMake 宏：

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_lua_project)
find_package(ament_cmake REQUIRED)
find_package(rcllua REQUIRED)
find_package(rcllua_cmake REQUIRED)

# ① 声明可执行脚本 → ros2 run my_lua_project my_node
rcllua_cmake_executable(${PROJECT_NAME}/my_node.lua my_node)

# ② 将 Lua 脚本安装到 LUA_PATH → 其他脚本可 require
rcllua_cmake_install_lib(${PROJECT_NAME})

# ③ 如果写了 C 扩展 → 安装到 LUA_CPATH
# rcllua_cmake_install_clib(${PROJECT_NAME} my_clib)

ament_package()
```

#### 3.4 编写 Lua 节点

```lua
-- my_lua_project/my_node.lua
require "rcllua.rcllua"
require "rcllua.Node"

local std_msgs = require("std_msgs.msg")

local MyNode = Node {
  name = "my_node",

  init = function(self)
    self.i = 0
    self.pub = self:create_publisher(std_msgs.String, 'topic', 10)
    self.timer = self:create_timer(1.0, function()
      local msg = std_msgs.String()
      msg.data = ("Hello %d"):format(self.i)
      self.pub:publish(msg)
      self.i = self.i + 1
    end)
  end
}

rcllua:init(arg)
rcllua:spin(MyNode())
rcllua:shutdown()
```

#### 3.5 编译与运行

```bash
cd ~/ws_lua
colcon build --symlink-install --packages-select my_lua_project
source install/setup.bash
ros2 run my_lua_project my_node
```

### 第四步：各模块 API 速查

| 你需要什么能力 | 参考文件 | 关键 API |
|---------------|----------|----------|
| 发布/订阅 | `publisher.lua` / `subscription.lua` | `self:create_publisher()`, `self:create_subscription()`, `self:bind('callback')` |
| 服务/客户端 | `service.lua` / `client.lua` | `self:create_service()`, `self:create_client()`, `client:call()` / `client:call_async()` |
| 参数 | `parameters.lua` | `self:declare_parameter()`, `self:set_parameters()` |
| Action | `action_server.lua` / `action_client.lua` | `self:create_action_server()`, `self:create_action_client()`, `send_goal_async()` |
| 生命周期节点 | `lifecycle_talker.lua` | `LifecycleNode{}` + `on_configure/on_activate/...` 回调 |
| 多节点 | — | `Executor:new()` + `add_node()` + `:spin()` |
| 单元测试 | `tests/test_lib.lua` | `rcllua_unit/testing.lua` 断言 |

---

## 五、Lua 脚本安装位置与动态加载

### 两类文件，两种安装路径

#### 1. Lua 库脚本 → `lib/lua/<包名>/`

由 `rcllua_cmake_install_lib()` 宏控制：

```cmake
install(DIRECTORY ${PROJECT_NAME} DESTINATION lib/lua)
```

以 `rcllua` 包为例：
```
源码:  rcllua/rcllua/Node.lua, rcllua/rcllua/Executor.lua ...
      ↓
安装:  install/lib/lua/rcllua/Node.lua, install/lib/lua/rcllua/Executor.lua ...
```

消息生成的 `.so` 文件同理（`rcllua_std_msgs`）：
```
install/lib/lua/std_msgs/msg.so
install/lib/lua/geometry_msgs/msg.so
install/lib/lua/sensor_msgs/msg.so
...
```

#### 2. 可执行包装器 → `lib/<项目名>/<可执行名>`

由 `rcllua_cmake_executable()` 宏控制，生成的包装器是一个**极小的 Lua 脚本**：

```cmake
file(WRITE "${CMAKE_INSTALL_PREFIX}/lib/${PROJECT_NAME}/${exec_name}"
    "#!/usr/local/bin/lua\n"
    "local script = loadfile('${CMAKE_SOURCE_DIR}/${src_name}')\n"  -- 指向源码绝对路径
    "script()"
)
```

例如 `rcllua_examples`：
```
install/lib/rcllua_examples/simple_publisher   -- 一个小Lua脚本
         ↓ 内部执行
         loadfile('{源码路径}/publisher.lua')
```

> **关键点**：包装器内部硬编码了**源码的绝对路径**，这意味着即使 install 后，运行时仍读取源文件。这也是为什么推荐 `--symlink-install`：修改源码后重启即生效，无需重新编译。

### 动态加载机制

ros2_lua 完全支持动态加载，基于 **Lua 标准的 `require()` + `LUA_PATH`/`LUA_CPATH` 环境变量**。

#### 环境变量注入

`rcllua_cmake` 通过 colcon 的 `ament_environment_hooks` 机制，在 `source install/setup.bash` 时自动注入：

**LUA_PATH** (`env_hook_lua.sh.in`):
```bash
export LUA_PATH=";;$COLCON_CURRENT_PREFIX/lib/lua/?.lua"
```
→ `require("rcllua.Node")` 实际查找 `lib/lua/rcllua/Node.lua`

**LUA_CPATH** (`env_hook_lua_c.sh.in`):
```bash
export LUA_CPATH=";;$COLCON_CURRENT_PREFIX/lib/lua/?.so"
```
→ `require("std_msgs.msg")` 实际查找 `lib/lua/std_msgs/msg.so`

#### 完整的加载流程

```
用户脚本: require("rcllua.Node")
    │
    ▼
Lua 运行时搜索 LUA_PATH:
    install/lib/lua/rcllua/Node.lua  ← 找到！
    │
    ▼
Node.lua 内部: require("rcllua.rclbind")
    │
    ▼
Lua 运行时搜索 LUA_CPATH:
    install/lib/lua/rcllua/rclbind.so  ← 找到！
    │
    ▼
rclbind.so 内部已链接 rcl, rcl_action, rcl_lifecycle 等 ROS2 库
```

#### 多个工作空间的叠加

如果 source 了多个 colcon 工作空间，每个空间的 `lib/lua/` 都会被追加到 `LUA_PATH`/`LUA_CPATH` 中。Lua 会按顺序搜索，先找到先用。

### 关于"热加载"

ros2_lua **没有内置**热加载机制，但：

| 方式 | 说明 |
|------|------|
| `--symlink-install` + 重启节点 | install 目录中文件是源文件的符号链接，改源码 → 重启节点 → 立即生效 |
| 手动热加载 | 操作 `package.loaded["mymodule"] = nil` 后重新 `require()`，利用 Lua 动态特性 |
| C 共享库 | `.so` 文件加载后无法卸载（Linux 限制），必须重启进程 |

推荐做法：**`--symlink-install` + 重启节点** 是最可靠的更新方式。

---

## 六、Fast Track 总结

如果只记住三件事：

1. **定义节点**：继承 `Node{name="xx", init=function(self)...end}`，在 `init` 中创建 publisher/subscriber/timer
2. **构建脚本**：`rcllua_cmake_executable(脚本路径 可执行名)` 一行就够了
3. **运行**：照常使用 `ros2 run pkg exe` 和 `ros2 launch`，完全兼容 ROS2 工具链
