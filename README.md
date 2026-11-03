# 知行教室 · 教室课程管理系统

面向学生、教师、教务管理员的教室课程协同平台。支持教室实时空闲查询、教师借用教室、课程/教室/借用管理和按周课表。

本仓库包含两种运行方式：

- **`app.py`**：单文件版，包含完整后端和前端资源，无需安装第三方库，直接运行。
- **`zhixing-classroom.zip`**：完整多文件工程，包含 `static/`、`tests/`、`docs/` 和启动脚本，适合查看源码或二次开发。

## 快速运行

环境要求：Python 3.10+，无需安装依赖。

```bash
python app.py
```

启动后访问：

```text
http://127.0.0.1:8000
```

首次运行会自动生成：

- `data/classroom.db`：SQLite 数据库和演示数据
- `.static_runtime/`：单文件版自动释放的前端资源

这两个目录已加入 `.gitignore`，不会提交。

## 演示账号

| 角色 | 账号 | 密码 | 姓名 |
|---|---|---|---|
| 学生 | `student` | `123456` | 陈晨 |
| 教师 | `teacher` | `123456` | 张伟 |
| 管理员 | `admin` | `123456` | 周明 |

登录页也提供三个一键登录卡片。其他演示账号：`lina`、`wangqiang`、`liuyang`、`zhaomin`，密码相同。

## 功能

### 学生端
- 实时查看教室空闲 / 使用中 / 维护中状态
- 按日期、时间、教学楼、容量、类型筛选
- 查看教室当日课程和借用时间轴
- 查看本班周课表、今日课程

### 教师端
- 实时教室总览
- 按日期和时段查找整段空闲的教室
- 借用教室，自动校验教室、教师、班级和借用冲突
- 教师时间冲突时提供最近可用时段建议
- 查看、取消自己的借用
- 查看本周授课课表

### 管理员端
- 课程增删改查
- 自动拦截教室冲突、教师冲突、班级冲突和借用冲突
- 教室档案维护与开放 / 维护状态管理
- 全校借用记录查看与取消
- 学期名称和教学周设置

## 技术实现

- Python 标准库 HTTP 服务
- SQLite 数据存储（WAL 模式）
- 原生 HTML / CSS / JavaScript
- PBKDF2 密码哈希、签名会话 Cookie、CSRF 校验
- 教室状态每 10 秒自动刷新
- 响应式布局，支持桌面和移动端

## 完整源码

`zhixing-classroom.zip` 解压后目录结构：

```text
app.py                 启动入口
database.py            建表、密码、演示数据
services.py            认证、空闲计算、课程与借用规则
web.py                 JSON API、静态资源、会话与 CSRF
static/                前端页面、样式和脚本
tests/                 业务测试
docs/                  截图和说明
start.bat              Windows 启动脚本
README.md              完整说明
```

```bash
unzip zhixing-classroom.zip
cd zhixing-classroom
python app.py
```

运行测试：

```bash
python -m unittest discover -s tests -v
```
