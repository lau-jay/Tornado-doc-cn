模版和UI
================

.. testsetup::

   import tornado.web

Tornado 包括一个简单，快速，灵活的模版语言。
本节介绍该语言以及相关问题比如国际化。

Tornado还可以与任何其他Python模板语言一起使用，虽然没有将这些系统一起整合到
`.RequestHandler.render`。要渲染简单的模版可以将字符串提供给 `.RequestHandler.write`。

配置模版
~~~~~~~~~~~~~~~~~~~~~

默认的，Tornado会在引用模版的 ``.py`` 文件相同的目录下搜索模版文件。如果你的模版文件在不同的目录下，
使用 ``template_path`` `应用设置
<.Application.settings>` (或者如果你不同的处理器有不同目录的模版文件覆写 `.RequestHandler.get_template_path`)。

如果需要从非本地文件系统加载模版，继承
`tornado.template.BaseLoader` 并在应用设置里将该子类的实例赋给
``template_loader`` 。

默认会缓存编译的模版；如果要即时看到模版的更改可以在应用设置字段  ``compiled_template_cache=False``
or ``debug=True`` 关闭缓存的行为并自动重新加载模版。

模版语法
~~~~~~~~~~~~~~~

一个Tornado模版只是HTML(或其他文本格式)中嵌入了Python控制代码和表达式的标记::

    <html>
       <head>
          <title>{{ title }}</title>
       </head>
       <body>
         <ul>
           {% for item in items %}
             <li>{{ escape(item) }}</li>
           {% end %}
         </ul>
       </body>
     </html>

如果你将这个模版保存到 "template.html" 并将该文件放入跟Python文件相同的目录，你可以用下面的
方式渲染模版:

.. testcode::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            items = ["Item 1", "Item 2", "Item 3"]
            self.render("template.html", title="My title", items=items)

.. testoutput::
   :hide:

Tornado模版支持的 *控制代码* 和 *表达式*。
控制代码被 ``{%`` 和 ``%}`` 包围，例如，
``{% if len(items) > 2 %}``. 表达式被 ``{{`` 和
``}}`` 包围, 例如。 ``{{ items[0] }}``.

控制代码是Python代码的子集。我们支持 ``if``, ``for``, ``while``, 和 ``try``,
所有这些都以 ``{% end %}`` 结束. 我们同样支持 *模版继承* ，
使用 ``extends`` 和 ``block`` 声明, 更多的关于这块的细节描述在 `tornado.template`。

表达式能够使用任何的Python表达式，包括函数调用。
模板代码在包含以下对象和函数的命名空间中执行（请注意，此列表适用于使用 `.RequestHandler.render` 和 `~.RequestHandler.render_string`
渲染的模板。如果你直接在 `.RequestHandler` 之外使用`tornado.template`模块，那么这些条目就不存在了).

- ``escape``: alias for `tornado.escape.xhtml_escape`
- ``xhtml_escape``: alias for `tornado.escape.xhtml_escape`
- ``url_escape``: alias for `tornado.escape.url_escape`
- ``json_encode``: alias for `tornado.escape.json_encode`
- ``squeeze``: alias for `tornado.escape.squeeze`
- ``linkify``: alias for `tornado.escape.linkify`
- ``datetime``: the Python `datetime` module
- ``handler``: the current `.RequestHandler` object
- ``request``: alias for `handler.request <.HTTPServerRequest>`
- ``current_user``: alias for `handler.current_user
  <.RequestHandler.current_user>`
- ``locale``: alias for `handler.locale <.Locale>`
- ``_``: alias for `handler.locale.translate <.Locale.translate>`
- ``static_url``: alias for `handler.static_url <.RequestHandler.static_url>`
- ``xsrf_form_html``: alias for `handler.xsrf_form_html
  <.RequestHandler.xsrf_form_html>`
- ``reverse_url``: alias for `.Application.reverse_url`
- All entries from the ``ui_methods`` and ``ui_modules``
  ``Application`` settings
- Any keyword arguments passed to `~.RequestHandler.render` or
  `~.RequestHandler.render_string`

当您构建一个真正的应用程序时，您将要使用Tornado模板的所有功能，尤其是模板
继承。 阅读所有关于 `tornado.template` 中的功能
部分（一些功能，包括 ``UIModules`` 在 `tornado.web` 模块中实现

在内部，Tornado是直接将模版转换为Python代码。
在模板中包含的表达式将逐字复制到表示模板的Python函数中。
T我们不会试图阻止模板语言中的任何内容; 我们明确地创建了它，以提供其他更严格的模板系统所阻止的灵活性。
因此，如果在模板表达式中编写随机内容，则在执行模板时会出现相应的Python错误。

所有的模版的输出默认都是使用 `tornado.escape.xhtml_escape` 函数转义的。
这个默认的行为可以通过更改 `.Application` 中的设置全局变量 ``autoescape=None`` 或
`.tornado.template.Loader` 构造器更改, 在文件中使用
``{% autoescape None %}`` 指令directive, 或者在单个表达式中用  ``{% raw ...%}``
替换 ``{{ ... }}`` 。另外，在这些地方的每一个中，可以使用转义函数的名称替代 ``None`` 。


请注意，虽然Tornado的自动转义有助于避免XSS漏洞，但并不是在所有情况下都是能避免的。
出现在某些位置的表达式（例如Javascript或CSS）可能需要额外的转义。
此外，必须注意始终在可能包含不受信任内容的HTML属性中使用双引号和 `.xhtml_escape`，
或者必须对属性单独的使用转义函数。(看例子 http://wonko.com/post/html-escaping)

国际化
~~~~~~~~~~~~~~~~~~~~

当前用户的本地语言环境(无论他们是否登录)总是通过请求处理器的 ``self.locale`` 和
模板中的 ``locale`` 来得到。本地语言环境的编码（比如, ``en_US`` ）可以通过
 ``local.name`` 来获取，此外还可以通过翻译方法 ``.Locale.translate`` 来翻译字符串。
模板还具有可用于字符串翻译的全局函数, 调用 ``_()`` 可以翻译字符串。
这个翻译函数有种用法::

    _("Translate this string")

直接根据当前语言环境翻译字符串, 和::

    _("A person liked this", "%(num)d people liked this",
      len(people)) % {"num": len(people)}

根据第三个参数的值转换字符串，可以是单数或复数。 在上面的示例中，
如果 ``len（people）`` 为 ``1`` ，则将返回第一个字符串的翻译，
否则将返回第二个字符串的翻译。

最常见的翻译模式是使用Python的字符串格式化的方式（在上面的示例中为 ``％(num)d`` ），
因为占位符可以放在待翻译的字符串的任意位置，比较灵活。

这里是一个使用恰当的国际化模版::

    <html>
       <head>
          <title>FriendFeed - {{ _("Sign in") }}</title>
       </head>
       <body>
         <form action="{{ request.path }}" method="post">
           <div>{{ _("Username") }} <input type="text" name="username"/></div>
           <div>{{ _("Password") }} <input type="password" name="password"/></div>
           <div><input type="submit" value="{{ _("Sign in") }}"/></div>
           {% module xsrf_form_html() %}
         </form>
       </body>
     </html>

默认的，我们使用用户浏览器的发送的http头部中的 ``Accept-Language`` 头部字段检测用户的语言环境。
如果没有发现 ``Accept-Language `` 值那么我们选择 ``en_US`` 。
如果您让用户将其语言环境设置为首选项，则可以通  `.RequestHandler.get_user_locale`:
覆盖默认的语言环境选择。

.. testcode::

    class BaseHandler(tornado.web.RequestHandler):
        def get_current_user(self):
            user_id = self.get_secure_cookie("user")
            if not user_id: return None
            return self.backend.get_user_by_id(user_id)

        def get_user_locale(self):
            if "locale" not in self.current_user.prefs:
                # Use the Accept-Language header
                return None
            return self.current_user.prefs["locale"]

.. testoutput::
   :hide:

如果 ``get_user_locale`` 返回 ``None``, 我们选择we fall back on the
我们讲退化到使用 ``Accept-Language`` HTTP头部字段。

The `tornado.locale` module supports loading translations in two
formats: the ``.mo`` format used by `gettext` and related tools, and a
simple ``.csv`` format.  An application will generally call either
`tornado.locale.load_translations` or
`tornado.locale.load_gettext_translations` once at startup; see those
methods for more details on the supported formats..

You can get the list of supported locales in your application with
`tornado.locale.get_supported_locales()`. The user's locale is chosen
to be the closest match based on the supported locales. For example, if
the user's locale is ``es_GT``, and the ``es`` locale is supported,
``self.locale`` will be ``es`` for that request. We fall back on
``en_US`` if no close match can be found.

.. _ui-modules:

UI 模块
~~~~~~~~~~

Tornado支持 *UI modules* 
Tornado supports *UI modules* to make it easy to support standard,
reusable UI widgets across your application. UI modules are like special
function calls to render components of your page, and they can come
packaged with their own CSS and JavaScript.

For example, if you are implementing a blog, and you want to have blog
entries appear on both the blog home page and on each blog entry page,
you can make an ``Entry`` module to render them on both pages. First,
create a Python module for your UI modules, e.g., ``uimodules.py``::

    class Entry(tornado.web.UIModule):
        def render(self, entry, show_comments=False):
            return self.render_string(
                "module-entry.html", entry=entry, show_comments=show_comments)

Tell Tornado to use ``uimodules.py`` using the ``ui_modules`` setting in
your application::

    from . import uimodules

    class HomeHandler(tornado.web.RequestHandler):
        def get(self):
            entries = self.db.query("SELECT * FROM entries ORDER BY date DESC")
            self.render("home.html", entries=entries)

    class EntryHandler(tornado.web.RequestHandler):
        def get(self, entry_id):
            entry = self.db.get("SELECT * FROM entries WHERE id = %s", entry_id)
            if not entry: raise tornado.web.HTTPError(404)
            self.render("entry.html", entry=entry)

    settings = {
        "ui_modules": uimodules,
    }
    application = tornado.web.Application([
        (r"/", HomeHandler),
        (r"/entry/([0-9]+)", EntryHandler),
    ], **settings)

Within a template, you can call a module with the ``{% module %}``
statement.  For example, you could call the ``Entry`` module from both
``home.html``::

    {% for entry in entries %}
      {% module Entry(entry) %}
    {% end %}

and ``entry.html``::

    {% module Entry(entry, show_comments=True) %}

Modules can include custom CSS and JavaScript functions by overriding
the ``embedded_css``, ``embedded_javascript``, ``javascript_files``, or
``css_files`` methods::

    class Entry(tornado.web.UIModule):
        def embedded_css(self):
            return ".entry { margin-bottom: 1em; }"

        def render(self, entry, show_comments=False):
            return self.render_string(
                "module-entry.html", show_comments=show_comments)

Module CSS and JavaScript will be included once no matter how many times
a module is used on a page. CSS is always included in the ``<head>`` of
the page, and JavaScript is always included just before the ``</body>``
tag at the end of the page.

When additional Python code is not required, a template file itself may
be used as a module. For example, the preceding example could be
rewritten to put the following in ``module-entry.html``::

    {{ set_resources(embedded_css=".entry { margin-bottom: 1em; }") }}
    <!-- more template html... -->

This revised template module would be invoked with::

    {% module Template("module-entry.html", show_comments=True) %}

The ``set_resources`` function is only available in templates invoked
via ``{% module Template(...) %}``. Unlike the ``{% include ... %}``
directive, template modules have a distinct namespace from their
containing template - they can only see the global template namespace
and their own keyword arguments.
