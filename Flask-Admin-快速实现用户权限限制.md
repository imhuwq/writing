---
title: Flask-Admin 快速实现用户权限限制
date: 2016-09-17 06:17:55
categories:
- 技术
tags:
- python
- web 开发
- flask
---

这个办法基于以下两个事实：  
1. Flask Admin 本身是一个 Flask 蓝图  
2. Flask 的 ModelView 类可以通过 `can_delete = True/False` 等来关闭/开启相应的操作
所以实现的办法是，给 Flask-Admin 蓝图的 `before_request_funcs` 注册一个 `check_user_permission` 函数，在每次请求之前根据用户的权限来刷新 ModelView 的设置。  
而蓝图的 `before_request` 必须要在 `app.register_blueprint` 调用之前，根据 Flask-Admin 创建蓝图和注册蓝图的流程来看，复写 ModelView 的 `create_blueprint` 这个函数就可以了。  
<!-- more -->
```python
class BaseModelView(ModelView):
    def is_accessible(self):
        return current_user.is_authenticated

    def inaccessible_callback(self, name, **kwargs):
        return abort(404)

    def check_permission(self):
        """根据用户的权限来重新设置 create, edit, delete 以及 column_editable_list 的设置"""
        self.can_create = True
        self.can_edit = True
        self.can_delete = True

        if not self.column_editable_list_bk and self.column_editable_list:
            self.column_editable_list_bk = self.column_editable_list[:]

        if not current_user.has_create_access:
            self.can_create = False
            self.can_edit = False
            self.can_delete = False
            self.column_editable_list = list()
        elif not current_user.has_edit_access:
            self.can_edit = False
            self.can_delete = False
            self.column_editable_list = list()
        elif not current_user.has_delete_access:
            self.can_delete = False
            self.column_editable_list = self.column_editable_list_bk

    def create_blueprint(self, admin):
        self.admin = admin
        self.column_editable_list_bk = list()
       ...
       # 以上部分都相同
        if self.name is None:
            self.name = self._prettify_class_name(self.__class__.__name__)
        self.blueprint = Blueprint(self.endpoint, __name__,
                       url_prefix=self.url,
                       subdomain=self.admin.subdomain,
                       template_folder=op.join('templates', self.admin.template_mode),
                       static_folder=self.static_folder,
                       static_url_path=self.static_url_path)
        # 主要是这一行
        self.blueprint.before_request(self.check_permission)
        # 以下都相同
        ...
        return self.blueprint
```

这个实现办法还依赖于 Flask-Login 实现的用户登录机制，用户的权限验证在用户的 Model 中进行定义。

