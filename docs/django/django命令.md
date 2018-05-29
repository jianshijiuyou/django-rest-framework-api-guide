<hr>

安装

```
pip install django[=x.xx.x]
```

项目管理

| 命令 | 作用
|:------------
| `django-admin startproject [project name]` | 创建项目
| `django-admin startapp [app name]` | 创建应用
| `python manage.py makemigrations` | 创建数据库迁移文件
| `python manage.py migrate` | 将生成的迁移文件应用到数据库
| `python manage.py createsuperuser` | 创建管理员


调试

| 命令 | 作用
|:------------
| `python manage.py runserver` | 启动项目，默认监听 `8000` 端口
| `python manage.py runserver 0:8080` | 自定义 ip 和 port
| `python manage.py sqlmigrate [app name] [0001]` | 查看迁移时执行的 sql 指令
| `python manage.py shell` | shell
| `python manage.py test` | 执行测试
| `python manage.py test [app name]` | 测试指定应用
| `python manage.py test [appname.tests.test_xxx_xx]` | 测试指定文件（不能加 `py` 后缀）
| `python manage.py test [--verbosity=2]` | `Verbosity` 决定了将要打印到控制台的通知和调试信息量; <br>`0` 是无输出，`1` 是正常输出，`2` 是详细输出。