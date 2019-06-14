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
所有这些都以 ``{% end %}`` 结束. 我们同样支持 *内嵌模版*，
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

When you are building a real application, you are going to want to use
all of the features of Tornado templates, especially template
inheritance. Read all about those features in the `tornado.template`
section (some features, including ``UIModules`` are implemented in the
`tornado.web` module)

Under the hood, Tornado templates are translated directly to Python. The
expressions you include in your template are copied verbatim into a
Python function representing your template. We don't try to prevent
anything in the template language; we created it explicitly to provide
the flexibility that other, stricter templating systems prevent.
Consequently, if you write random stuff inside of your template
expressions, you will get random Python errors when you execute the
template.

All template output is escaped by default, using the
`tornado.escape.xhtml_escape` function. This behavior can be changed
globally by passing ``autoescape=None`` to the `.Application` or
`.tornado.template.Loader` constructors, for a template file with the
``{% autoescape None %}`` directive, or for a single expression by
replacing ``{{ ... }}`` with ``{% raw ...%}``. Additionally, in each of
these places the name of an alternative escaping function may be used
instead of ``None``.

Note that while Tornado's automatic escaping is helpful in avoiding
XSS vulnerabilities, it is not sufficient in all cases.  Expressions
that appear in certain locations, such as in Javascript or CSS, may need
additional escaping.  Additionally, either care must be taken to always
use double quotes and `.xhtml_escape` in HTML attributes that may contain
untrusted content, or a separate escaping function must be used for
attributes (see e.g. http://wonko.com/post/html-escaping)

Internationalization
~~~~~~~~~~~~~~~~~~~~

The locale of the current user (whether they are logged in or not) is
always available as ``self.locale`` in the request handler and as
``locale`` in templates. The name of the locale (e.g., ``en_US``) is
available as ``locale.name``, and you can translate strings with the
`.Locale.translate` method. Templates also have the global function
call ``_()`` available for string translation. The translate function
has two forms::

    _("Translate this string")

which translates the string directly based on the current locale, and::

    _("A person liked this", "%(num)d people liked this",
      len(people)) % {"num": len(people)}

which translates a string that can be singular or plural based on the
value of the third argument. In the example above, a translation of the
first string will be returned if ``len(people)`` is ``1``, or a
translation of the second string will be returned otherwise.

The most common pattern for translations is to use Python named
placeholders for variables (the ``%(num)d`` in the example above) since
placeholders can move around on translation.

Here is a properly internationalized template::

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

By default, we detect the user's locale using the ``Accept-Language``
header sent by the user's browser. We choose ``en_US`` if we can't find
an appropriate ``Accept-Language`` value. If you let user's set their
locale as a preference, you can override this default locale selection
by overriding `.RequestHandler.get_user_locale`:

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

If ``get_user_locale`` returns ``None``, we fall back on the
``Accept-Language`` header.

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

UI modules
~~~~~~~~~~

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
