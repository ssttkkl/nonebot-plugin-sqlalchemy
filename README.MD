<!-- markdownlint-disable MD033 MD036 MD041 -->

<p align="center">
  <a href="https://v2.nonebot.dev/"><img src="https://v2.nonebot.dev/logo.png" width="200" height="200" alt="nonebot"></a>
</p>

<div align="center">

nonebot-plugin-sqlalchemy
============

_✨ NoneBot SQLAlchemy 封装插件 ✨_

</div>

<p align="center">
  <a href="https://raw.githubusercontent.com/ssttkkl/nonebot-plugin-sqlalchemy/master/LICENSE">
    <img src="https://img.shields.io/github/license/ssttkkl/nonebot-plugin-sqlalchemy.svg" alt="license">
  </a>
  <a href="https://pypi.python.org/pypi/nonebot-plugin-sqlalchemy">
    <img src="https://img.shields.io/pypi/v/nonebot-plugin-sqlalchemy.svg" alt="pypi">
  </a>
  <img src="https://img.shields.io/badge/python-3.9+-blue.svg" alt="python">
</p>

**建议使用 https://github.com/he0119/nonebot-plugin-datastore 代替**

## Get Started

### 1、定义data_source

```python
from nonebot import get_driver, require

# 注意必须先require再import
require("nonebot_plugin_sqlalchemy")
from nonebot_plugin_sqlalchemy import DataSource

# 必须使用支持asyncio的驱动器
db_conn_url = "postgresql+asyncpg://username:password@localhost:5432/database"
data_source = DataSource(get_driver(), db_conn_url)
```

### 2、定义映射
```python
from sqlalchemy import Column, String, Integer

@data_source.registry.mapped
class UserOrm:
    __tablename__ = 'users'

    id: int = Column(Integer, primary_key=True, autoincrement=True)
    username: str = Column(String)
    password: str = Column(String)
    nickname: str = Column(String)
```

### 3、在Matcher中使用

```python
from nonebot import on_command
from nonebot.adapters.onebot.v11 import MessageEvent
from nonebot.internal.matcher import Matcher
from sqlalchemy import select

login_matcher = on_command("login")


@login_matcher.handle()
async def handler(event: MessageEvent, matcher: Matcher):
    username, password = event.get_plaintext().split(" ")

    session = data_source.session()
    
    stmt = select(UserOrm).where(UserOrm.username == username, UserOrm.password == password)
    result = await session.execute(stmt)
    user = result.scalar_one_or_none()

    if user is not None:
        await matcher.send(f"Hello, {user.nickname}")
```

通过`data_source.session()`获取AsyncSession对象，此处获取的session实际上是async_scoped_session。

在Matcher的一次执行过程中，多次调用`data_source.session()`获得的是同一个session，并且会在Matcher执行完毕后自动关闭。也就是说我们可以像下面这样使用：

```python
from nonebot import on_command
from nonebot.adapters.onebot.v11 import MessageEvent
from nonebot.internal.matcher import Matcher
from sqlalchemy import select
from typing import Optional

async def login(username: str, password: str) -> Optional[User]:
    session = data_source.session()
    
    stmt = select(UserOrm).where(UserOrm.username == username, UserOrm.password == password)
    result = await session.execute(stmt)
    user = result.scalar_one_or_none()

    return user


login_matcher = on_command("login")


@login_matcher.handle()
async def handler(event: MessageEvent, matcher: Matcher):
    username, password = event.get_plaintext().split(" ")
    user = await login(username, password)
    if user is not None:
        await matcher.send(f"Hello, {user.nickname}")
```

参考：https://docs.sqlalchemy.org/en/14/orm/extensions/asyncio.html#using-asyncio-scoped-session

注意：务必保证一次Matcher执行过程不会在不同的Task中调用`data_source.session()`获取session（即不要使用`create_task()`或`ensure_future()`创建Task），否则可能出现错误。若有这样的需求，请参考下文的方法手动创建并管理session。


### 4、在Matcher之外使用

在Matcher之外（如on_bot_connect等钩子函数中，或者是APScheduler的定时任务中）则必须通过`AsyncSession(data_source.engine)`创建session。

```python
async def do_something():
    async with AsyncSession(data_source.engine) as session:
        # ...
```
