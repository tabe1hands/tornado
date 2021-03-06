.. currentmodule:: tornado.web

概要
========

`FriendFeed <http://friendfeed.com/>`_ のウェブサーバは、比較的シンプルな、
Pythonで書かれたノンブロッキングウェブサーバです。 FriendFeedのアプリケーションは、
`web.py <http://webpy.org/>`_ や Googleの　
`webapp <http://code.google.com/appengine/docs/python/tools/webapp/>`_
のような、小さなウェブフレームワークで書かれていますが、 追加のツールや、
ノンブロッキングなウェブサーバーやツールの強みを活かすための最適化を備えています。

`Tornado <http://github.com/facebook/tornado>`_ は、FriendFeedが頻繁に使用している
ウェブサーバやいくつかのツールのオープンソース版です。
このフレームワークは、最も主流なウェブサーバフレームワーク(や、
もちろんほとんどのPythonフレームワーク)とは違います、それはノンブロッキングだからです、
そして、適度に速いです。それはノンブロッキングで、 `epoll
<http://www.kernel.org/doc/man-pages/online/pages/man4/epoll.4.html>`_
や kqueue を使用して数千の接続を同時に扱うことができるからです、
これはリアルタイムウェブサービスに理想的なフレームワークであるということを意味します。
FrinedFeedのサーバへ接続を張り続けます。(数千のクライアントをサポートするための
サーバーのスケーリングに関するより多くの情報は、 `The C10K problem
<http://www.kegel.com/c10k.html>`_ を参照してください。)


ここでは模範的な"Hello, wold"をアプリの例にあげます:

::

    import tornado.ioloop
    import tornado.web

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, world")

    application = tornado.web.Application([
        (r"/", MainHandler),
    ])

    if __name__ == "__main__":
        application.listen(8888)
        tornado.ioloop.IOLoop.instance().start()

私たちは、モジュール間の相互依存を減らすためにコードベースを
きれいにするように努めました、そのため、全パッケージを使わずに
あなたのプロジェクトでどのモジュールも独立して使うとこが (理論上) できるはずです。

Requestハンドラとrquest引数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Tornadoウェブアプリケーションは、 `tornado.web.RequestHandler` サブクラスに
URLsやURLのパターンを割り当てます。そのURLへの HTTP ``GET`` や ``POST`` の
リクエストを扱うため、 それらのクラスに ``get()``  や ``post()`` のメソッドを
定義します。

このコードでは ルートURL ``/`` を ``MainHandler`` へ割り当て、
URLパターン ``/story/([0-9]+)`` を ``StoryHandler`` へ割り当てています。
``RequestHandler`` メソッドへ引数として正規表現グループが渡されています:

::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("You requested the main page")

    class StoryHandler(tornado.web.RequestHandler):
        def get(self, story_id):
            self.write("You requested the story " + story_id)

    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/story/([0-9]+)", StoryHandler),
    ])

``get_argument()`` メソッドを使って、クエリーストリングの引数を取得して
``POST`` のボディーをパースできます:

::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write('<html><body><form action="/" method="post">'
                       '<input type="text" name="message">'
                       '<input type="submit" value="Submit">'
                       '</form></body></html>')

        def post(self):
            self.set_header("Content-Type", "text/plain")
            self.write("You wrote " + self.get_argument("message"))

ファイルのアップロードは ``self.request.files`` を利用します、
これはファイルのリストに名前 (HTMLの ``<input type="file">`` エレメント 
の name ) を割り当てます。それぞれのファイルは、
``{"filename":..., "content_type":..., "body":...}`` のような形式の辞書になります。

もし、例えば、403 Unauthorized のようなエラーレスポンスを
クライアントへ送信したい場合は、 
単に ``tornado.web.HTTPError`` 例外を発生させるだけです:

::

    if not self.user_is_logged_in():
        raise tornado.web.HTTPError(403)

リクエストハンドラは ``self.request`` を通して
現在のリクエストを表すオプジェクトにアクセスできます。
``HTTPRequest`` オブジェクトは多くの実用的な属性が含まれています:

-  ``arguments`` - ``GET`` と ``POST`` の全ての引数
-  ``files`` - 全てのアップロードしたファイル (POST リクエストの 
   ``multipart/form-data`` を用いた)
-  ``path`` - リクエストパス ( ``?`` より前の全て)
-  ``headers`` - リクエストヘッダー

完全な属性のリストは `tornado.httpserver.HTTPRequest` のクラス定義を参照してください。

Overriding RequestHandler methods
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In addition to ``get()``/``post()``/etc, certain other methods in
``RequestHandler`` are designed to be overridden by subclasses when
necessary. On every request, the following sequence of calls takes
place:

1. A new RequestHandler object is created on each request
2. ``initialize()`` is called with keyword arguments from the
   ``Application`` configuration. (the ``initialize`` method is new in
   Tornado 1.1; in older versions subclasses would override ``__init__``
   instead). ``initialize`` should typically just save the arguments
   passed into member variables; it may not produce any output or call
   methods like ``send_error``.
3. ``prepare()`` is called. This is most useful in a base class shared
   by all of your handler subclasses, as ``prepare`` is called no matter
   which HTTP method is used. ``prepare`` may produce output; if it
   calls ``finish`` (or ``send_error``, etc), processing stops here.
4. One of the HTTP methods is called: ``get()``, ``post()``, ``put()``,
   etc. If the URL regular expression contains capturing groups, they
   are passed as arguments to this method.

Here is an example demonstrating the ``initialize()`` method:

::

    class ProfileHandler(RequestHandler):
        def initialize(self, database):
            self.database = database

        def get(self, username):
            ...

    app = Application([
        (r'/user/(.*)', ProfileHandler, dict(database=database)),
        ])

Other methods designed for overriding include:

-  ``write_error(self, status_code, exc_info=None, **kwargs)`` -
   outputs HTML for use on error pages.
-  ``get_current_user(self)`` - see `User
   Authentication <#user-authentication>`_ below
-  ``get_user_locale(self)`` - returns ``locale`` object to use for the
   current user
-  ``get_login_url(self)`` - returns login url to be used by the
   ``@authenticated`` decorator (default is in ``Application`` settings)
-  ``get_template_path(self)`` - returns location of template files
   (default is in ``Application`` settings)
-  ``set_default_headers(self)`` - may be used to set additional headers
   on the response (such as a custom ``Server`` header)

エラーハンドリング
~~~~~~~~~~~~~~~~~~

`RequestHandler` からエラーを返却する方法は３通りあります:

1. `~tornado.web.RequestHandler.set_status` を手動で呼び出し、
   普通にレスポンスボディーを出力します。
2. `~RequestHandler.send_error` を呼び出す。この放棄(discards)は、
   出力をフラッシュせずに保留して、エラーページを生成するために
   `~RequestHandler.write_error` を呼びます。
3. 例外を発生する。 `tornado.web.HTTPError` は特定のステータスコードを
   生成するのに利用できます; 他の例外の全ては、500ステータスを返却します。
   例外ハンドラは、エラーページを生成するために `~RequestHandler.send_error` 
   と `~RequestHandler.write_error` を使用します。

デフォルトのエラーページは、デバッグモードとスタックトレースを、
そうでなければ、 一行のエラーの概要(例えば、"500: Internal Server Error")
を含みます。カスタムエラーページを生成するためには、
`RequestHandler.write_error` をオーバーライドします。 このメソッドは、
例えば、 `~RequestHandler.write` と `~RequestHandler.render` を通して、
標準出力を生成することができます。例外で引き起こされてエラーになった場合は、 
3種類の ``exec_info`` が、キーワード引数(この例外は、 ``sys.exc_info`` では
最新の例外なることが保証されないため、 `write_error`` は、例えば、
`traceback.format_exc` の代わりに `traceback.format_exception` を
利用すべきことに注意)として渡されます。

Tornado 2.0 より前では、カスタムエラーページは 
``RequestHandler.get_error_html`` をオーバーライドすることによって
実装され、標準出力メソッド呼ぶ代わりに文字列としてエラーページを
返却していました(そして、ちょっと変わった例外用の動作もありました)。
このメソッドは、まだサポートされていますが廃止予定で、
アプリケーションは、 `RequestHandler.write_error` に変更することが
推奨されています。

Redirection
~~~~~~~~~~~

There are two main ways you can redirect requests in Tornado:
``self.redirect`` and with the ``RedirectHandler``.

You can use ``self.redirect`` within a ``RequestHandler`` method (like
``get``) to redirect users elsewhere. There is also an optional
parameter ``permanent`` which you can use to indicate that the
redirection is considered permanent.

This triggers a ``301 Moved Permanently`` HTTP status, which is useful
for e.g. redirecting to a canonical URL for a page in an SEO-friendly
manner.

The default value of ``permanent`` is ``False``, which is apt for things
like redirecting users on successful POST requests.

::

    self.redirect('/some-canonical-page', permanent=True)

``RedirectHandler`` is available for your use when you initialize
``Application``.

For example, notice how we redirect to a longer download URL on this
website:

::

    application = tornado.wsgi.WSGIApplication([
        (r"/([a-z]*)", ContentHandler),
        (r"/static/tornado-0.2.tar.gz", tornado.web.RedirectHandler,
         dict(url="http://github.com/downloads/facebook/tornado/tornado-0.2.tar.gz")),
    ], **settings)

The default ``RedirectHandler`` status code is
``301 Moved Permanently``, but to use ``302 Found`` instead, set
``permanent`` to ``False``.

::

    application = tornado.wsgi.WSGIApplication([
        (r"/foo", tornado.web.RedirectHandler, {"url":"/bar", "permanent":False}),
    ], **settings)

Note that the default value of ``permanent`` is different in
``self.redirect`` than in ``RedirectHandler``. This should make some
sense if you consider that ``self.redirect`` is used in your methods and
is probably invoked by logic involving environment, authentication, or
form submission, but ``RedirectHandler`` patterns are going to fire 100%
of the time they match the request URL.

テンプレート
~~~~~~~~~~~~

Pythonがサポートされたいくつかのテンプレート言語を使うことができますが、
Tornadoには、世の中の最も人気のあるテンプレートシステムの多くより、
もっと早く、より柔軟なテンプレート言語が搭載されています。完全なドキュメントは
`tornado.template` モジュールのドキュメントを参照してください。

Tornadoのテンプレートは、マークアップを拡張したシーケンスや式の制御に
Pythonを使うただのHTML(や、テキストベースの他の形式)です:

::

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

もし、あなたが "template.html" として、このテンプレートを保存し、
Pythonファイルと同じディレクトリに置いた場合、
このテンプレートと合わせてレンダリングすることができます:

::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            items = ["Item 1", "Item 2", "Item 3"]
            self.render("template.html", title="My title", items=items)

Tornadoのテンプレートは *control statements* と *expressions* を
サポートします。制御文は ``{%`` と ``%}`` で囲まれます、例えば、
``{% if len(items) > 2 %}`` 。式は、 ``{{`` と ``}}`` で囲まれます、
例えば、 ``{{ items[0] }}`` 。

制御文は、程度の差はありますが、Pythonの文を厳密に割り当てています。
``if``, ``for``, ``while``, や ``try`` をサポートしており、
全部 ``{% end %}`` で終わっています。 ``extends`` と ``block`` 文を 
使って *template inheritance* もサポートしており、 `tornado.template` の
ドキュメントで詳しく説明されています。

Expressions can be any Python expression, including function calls.
Template code is executed in a namespace that includes the following
objects and functions (Note that this list applies to templates rendered
using ``RequestHandler.render`` and ``render_string``. If you're using
the ``template`` module directly outside of a ``RequestHandler`` many of
these entries are not present).

-  ``escape``: alias for ``tornado.escape.xhtml_escape``
-  ``xhtml_escape``: alias for ``tornado.escape.xhtml_escape``
-  ``url_escape``: alias for ``tornado.escape.url_escape``
-  ``json_encode``: alias for ``tornado.escape.json_encode``
-  ``squeeze``: alias for ``tornado.escape.squeeze``
-  ``linkify``: alias for ``tornado.escape.linkify``
-  ``datetime``: the Python ``datetime`` module
-  ``handler``: the current ``RequestHandler`` object
-  ``request``: alias for ``handler.request``
-  ``current_user``: alias for ``handler.current_user``
-  ``locale``: alias for ``handler.locale``
-  ``_``: alias for ``handler.locale.translate``
-  ``static_url``: alias for ``handler.static_url``
-  ``xsrf_form_html``: alias for ``handler.xsrf_form_html``
-  ``reverse_url``: alias for ``Application.reverse_url``
-  All entries from the ``ui_methods`` and ``ui_modules``
   ``Application`` settings
-  Any keyword arguments passed to ``render`` or ``render_string``

When you are building a real application, you are going to want to use
all of the features of Tornado templates, especially template
inheritance. Read all about those features in the `tornado.template`
section (some features, including ``UIModules`` are implemented in the
``web`` module)

内部では、TornadoのテンプレートはPythonへ直接変換されます。
テンプレートに含める式は、テンプレートを表示するPythonの関数に
そのままコピーされています。 私たちは、テンプレート言語で
なんでも取りあえず抑止してみることはしません; 私たちは、
その他の柔軟性を提供することを明示的にして、それを作りましたが、
より厳密なテンプレートシステムは抑止します。

全テンプレートの出力はデフォルトでエスケープされます、それには 
``tornado.escape.xhtml_escape`` ファンクションを使います。
この振る舞いは、``Application`` や ``TemplateLoader`` コンストラクタに
``autoescape=None`` を渡すことで全体的に変更されます、また、 
``{% autoescape None %}`` ディレクティブを使ったテンプレートファイルや、
``{{ ... }}`` を ``{% raw ...%}`` で置き換える単式もあります。
加えて、それぞれで、 ``None`` の代わりに、
他のエスケープファンクションの名前を置き換えて使うこともできます。

Cookies and secure cookies
~~~~~~~~~~~~~~~~~~~~~~~~~~

You can set cookies in the user's browser with the ``set_cookie``
method:

::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            if not self.get_cookie("mycookie"):
                self.set_cookie("mycookie", "myvalue")
                self.write("Your cookie was not set yet!")
            else:
                self.write("Your cookie was set!")

Cookies are easily forged by malicious clients. If you need to set
cookies to, e.g., save the user ID of the currently logged in user, you
need to sign your cookies to prevent forgery. Tornado supports this out
of the box with the ``set_secure_cookie`` and ``get_secure_cookie``
methods. To use these methods, you need to specify a secret key named
``cookie_secret`` when you create your application. You can pass in
application settings as keyword arguments to your application:

::

    application = tornado.web.Application([
        (r"/", MainHandler),
    ], cookie_secret="61oETzKXQAGaYdkL5gEmGeJJFuYh7EQnp2XdTP1o/Vo=")

Signed cookies contain the encoded value of the cookie in addition to a
timestamp and an `HMAC <http://en.wikipedia.org/wiki/HMAC>`_ signature.
If the cookie is old or if the signature doesn't match,
``get_secure_cookie`` will return ``None`` just as if the cookie isn't
set. The secure version of the example above:

::

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            if not self.get_secure_cookie("mycookie"):
                self.set_secure_cookie("mycookie", "myvalue")
                self.write("Your cookie was not set yet!")
            else:
                self.write("Your cookie was set!")

User authentication
~~~~~~~~~~~~~~~~~~~

The currently authenticated user is available in every request handler
as ``self.current_user``, and in every template as ``current_user``. By
default, ``current_user`` is ``None``.

To implement user authentication in your application, you need to
override the ``get_current_user()`` method in your request handlers to
determine the current user based on, e.g., the value of a cookie. Here
is an example that lets users log into the application simply by
specifying a nickname, which is then saved in a cookie:

::

    class BaseHandler(tornado.web.RequestHandler):
        def get_current_user(self):
            return self.get_secure_cookie("user")

    class MainHandler(BaseHandler):
        def get(self):
            if not self.current_user:
                self.redirect("/login")
                return
            name = tornado.escape.xhtml_escape(self.current_user)
            self.write("Hello, " + name)

    class LoginHandler(BaseHandler):
        def get(self):
            self.write('<html><body><form action="/login" method="post">'
                       'Name: <input type="text" name="name">'
                       '<input type="submit" value="Sign in">'
                       '</form></body></html>')

        def post(self):
            self.set_secure_cookie("user", self.get_argument("name"))
            self.redirect("/")

    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
    ], cookie_secret="61oETzKXQAGaYdkL5gEmGeJJFuYh7EQnp2XdTP1o/Vo=")

You can require that the user be logged in using the `Python
decorator <http://www.python.org/dev/peps/pep-0318/>`_
``tornado.web.authenticated``. If a request goes to a method with this
decorator, and the user is not logged in, they will be redirected to
``login_url`` (another application setting). The example above could be
rewritten:

::

    class MainHandler(BaseHandler):
        @tornado.web.authenticated
        def get(self):
            name = tornado.escape.xhtml_escape(self.current_user)
            self.write("Hello, " + name)

    settings = {
        "cookie_secret": "61oETzKXQAGaYdkL5gEmGeJJFuYh7EQnp2XdTP1o/Vo=",
        "login_url": "/login",
    }
    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
    ], **settings)

If you decorate ``post()`` methods with the ``authenticated`` decorator,
and the user is not logged in, the server will send a ``403`` response.

Tornado comes with built-in support for third-party authentication
schemes like Google OAuth. See the `tornado.auth`
for more details. Check out the `Tornado Blog example application <https://github.com/facebook/tornado/tree/master/demos/blog>`_ for a
complete example that uses authentication (and stores user data in a
MySQL database).

Cross-site request forgery protection
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

`Cross-site request
forgery <http://en.wikipedia.org/wiki/Cross-site_request_forgery>`_, or
XSRF, is a common problem for personalized web applications. See the
`Wikipedia
article <http://en.wikipedia.org/wiki/Cross-site_request_forgery>`_ for
more information on how XSRF works.

The generally accepted solution to prevent XSRF is to cookie every user
with an unpredictable value and include that value as an additional
argument with every form submission on your site. If the cookie and the
value in the form submission do not match, then the request is likely
forged.

Tornado comes with built-in XSRF protection. To include it in your site,
include the application setting ``xsrf_cookies``:

::

    settings = {
        "cookie_secret": "61oETzKXQAGaYdkL5gEmGeJJFuYh7EQnp2XdTP1o/Vo=",
        "login_url": "/login",
        "xsrf_cookies": True,
    }
    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
    ], **settings)

If ``xsrf_cookies`` is set, the Tornado web application will set the
``_xsrf`` cookie for all users and reject all ``POST``, ``PUT``, and
``DELETE`` requests that do not contain a correct ``_xsrf`` value. If
you turn this setting on, you need to instrument all forms that submit
via ``POST`` to contain this field. You can do this with the special
function ``xsrf_form_html()``, available in all templates:

::

    <form action="/new_message" method="post">
      {{ xsrf_form_html() }}
      <input type="text" name="message"/>
      <input type="submit" value="Post"/>
    </form>

If you submit AJAX ``POST`` requests, you will also need to instrument
your JavaScript to include the ``_xsrf`` value with each request. This
is the `jQuery <http://jquery.com/>`_ function we use at FriendFeed for
AJAX ``POST`` requests that automatically adds the ``_xsrf`` value to
all requests:

::

    function getCookie(name) {
        var r = document.cookie.match("\\b" + name + "=([^;]*)\\b");
        return r ? r[1] : undefined;
    }

    jQuery.postJSON = function(url, args, callback) {
        args._xsrf = getCookie("_xsrf");
        $.ajax({url: url, data: $.param(args), dataType: "text", type: "POST",
            success: function(response) {
            callback(eval("(" + response + ")"));
        }});
    };

For ``PUT`` and ``DELETE`` requests (as well as ``POST`` requests that
do not use form-encoded arguments), the XSRF token may also be passed
via an HTTP header named ``X-XSRFToken``.

If you need to customize XSRF behavior on a per-handler basis, you can
override ``RequestHandler.check_xsrf_cookie()``. For example, if you
have an API whose authentication does not use cookies, you may want to
disable XSRF protection by making ``check_xsrf_cookie()`` do nothing.
However, if you support both cookie and non-cookie-based authentication,
it is important that XSRF protection be used whenever the current
request is authenticated with a cookie.

Static files and aggressive file caching
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can serve static files from Tornado by specifying the
``static_path`` setting in your application:

::

    settings = {
        "static_path": os.path.join(os.path.dirname(__file__), "static"),
        "cookie_secret": "61oETzKXQAGaYdkL5gEmGeJJFuYh7EQnp2XdTP1o/Vo=",
        "login_url": "/login",
        "xsrf_cookies": True,
    }
    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/login", LoginHandler),
        (r"/(apple-touch-icon\.png)", tornado.web.StaticFileHandler,
         dict(path=settings['static_path'])),
    ], **settings)

This setting will automatically make all requests that start with
``/static/`` serve from that static directory, e.g.,
`http://localhost:8888/static/foo.png <http://localhost:8888/static/foo.png>`_
will serve the file ``foo.png`` from the specified static directory. We
also automatically serve ``/robots.txt`` and ``/favicon.ico`` from the
static directory (even though they don't start with the ``/static/``
prefix).

In the above settings, we have explicitly configured Tornado to serve
``apple-touch-icon.png`` “from” the root with the ``StaticFileHandler``,
though it is physically in the static file directory. (The capturing
group in that regular expression is necessary to tell
``StaticFileHandler`` the requested filename; capturing groups are
passed to handlers as method arguments.) You could do the same thing to
serve e.g. ``sitemap.xml`` from the site root. Of course, you can also
avoid faking a root ``apple-touch-icon.png`` by using the appropriate
``<link />`` tag in your HTML.

To improve performance, it is generally a good idea for browsers to
cache static resources aggressively so browsers won't send unnecessary
``If-Modified-Since`` or ``Etag`` requests that might block the
rendering of the page. Tornado supports this out of the box with *static
content versioning*.

To use this feature, use the ``static_url()`` method in your templates
rather than typing the URL of the static file directly in your HTML:

::

    <html>
       <head>
          <title>FriendFeed - {{ _("Home") }}</title>
       </head>
       <body>
         <div><img src="{{ static_url("images/logo.png") }}"/></div>
       </body>
     </html>

The ``static_url()`` function will translate that relative path to a URI
that looks like ``/static/images/logo.png?v=aae54``. The ``v`` argument
is a hash of the content in ``logo.png``, and its presence makes the
Tornado server send cache headers to the user's browser that will make
the browser cache the content indefinitely.

Since the ``v`` argument is based on the content of the file, if you
update a file and restart your server, it will start sending a new ``v``
value, so the user's browser will automatically fetch the new file. If
the file's contents don't change, the browser will continue to use a
locally cached copy without ever checking for updates on the server,
significantly improving rendering performance.

In production, you probably want to serve static files from a more
optimized static file server like `nginx <http://nginx.net/>`_. You can
configure most any web server to support these caching semantics. Here
is the nginx configuration we use at FriendFeed:

::

    location /static/ {
        root /var/friendfeed/static;
        if ($query_string) {
            expires max;
        }
     }

Localization
~~~~~~~~~~~~

The locale of the current user (whether they are logged in or not) is
always available as ``self.locale`` in the request handler and as
``locale`` in templates. The name of the locale (e.g., ``en_US``) is
available as ``locale.name``, and you can translate strings with the
``locale.translate`` method. Templates also have the global function
call ``_()`` available for string translation. The translate function
has two forms:

::

    _("Translate this string")

which translates the string directly based on the current locale, and

::

    _("A person liked this", "%(num)d people liked this",
      len(people)) % {"num": len(people)}

which translates a string that can be singular or plural based on the
value of the third argument. In the example above, a translation of the
first string will be returned if ``len(people)`` is ``1``, or a
translation of the second string will be returned otherwise.

The most common pattern for translations is to use Python named
placeholders for variables (the ``%(num)d`` in the example above) since
placeholders can move around on translation.

Here is a properly localized template:

::

    <html>
       <head>
          <title>FriendFeed - {{ _("Sign in") }}</title>
       </head>
       <body>
         <form action="{{ request.path }}" method="post">
           <div>{{ _("Username") }} <input type="text" name="username"/></div>
           <div>{{ _("Password") }} <input type="password" name="password"/></div>
           <div><input type="submit" value="{{ _("Sign in") }}"/></div>
           {{ xsrf_form_html() }}
         </form>
       </body>
     </html>

By default, we detect the user's locale using the ``Accept-Language``
header sent by the user's browser. We choose ``en_US`` if we can't find
an appropriate ``Accept-Language`` value. If you let user's set their
locale as a preference, you can override this default locale selection
by overriding ``get_user_locale`` in your request handler:

::

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

If ``get_user_locale`` returns ``None``, we fall back on the
``Accept-Language`` header.

You can load all the translations for your application using the
``tornado.locale.load_translations`` method. It takes in the name of the
directory which should contain CSV files named after the locales whose
translations they contain, e.g., ``es_GT.csv`` or ``fr_CA.csv``. The
method loads all the translations from those CSV files and infers the
list of supported locales based on the presence of each CSV file. You
typically call this method once in the ``main()`` method of your server:

::

    def main():
        tornado.locale.load_translations(
            os.path.join(os.path.dirname(__file__), "translations"))
        start_server()

You can get the list of supported locales in your application with
``tornado.locale.get_supported_locales()``. The user's locale is chosen
to be the closest match based on the supported locales. For example, if
the user's locale is ``es_GT``, and the ``es`` locale is supported,
``self.locale`` will be ``es`` for that request. We fall back on
``en_US`` if no close match can be found.

See the `tornado.locale`
documentation for detailed information on the CSV format and other
localization methods.

UI modules
~~~~~~~~~~

Tornado supports *UI modules* to make it easy to support standard,
reusable UI widgets across your application. UI modules are like special
functional calls to render components of your page, and they can come
packaged with their own CSS and JavaScript.

For example, if you are implementing a blog, and you want to have blog
entries appear on both the blog home page and on each blog entry page,
you can make an ``Entry`` module to render them on both pages. First,
create a Python module for your UI modules, e.g., ``uimodules.py``:

::

    class Entry(tornado.web.UIModule):
        def render(self, entry, show_comments=False):
            return self.render_string(
                "module-entry.html", entry=entry, show_comments=show_comments)

Tell Tornado to use ``uimodules.py`` using the ``ui_modules`` setting in
your application:

::

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

Within ``home.html``, you reference the ``Entry`` module rather than
printing the HTML directly:

::

    {% for entry in entries %}
      {% module Entry(entry) %}
    {% end %}

Within ``entry.html``, you reference the ``Entry`` module with the
``show_comments`` argument to show the expanded form of the entry:

::

    {% module Entry(entry, show_comments=True) %}

Modules can include custom CSS and JavaScript functions by overriding
the ``embedded_css``, ``embedded_javascript``, ``javascript_files``, or
``css_files`` methods:

::

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
rewritten to put the following in ``module-entry.html``:

::

    {{ set_resources(embedded_css=".entry { margin-bottom: 1em; }") }}
    <!-- more template html... -->

This revised template module would be invoked with

::

    {% module Template("module-entry.html", show_comments=True) %}

The ``set_resources`` function is only available in templates invoked
via ``{% module Template(...) %}``. Unlike the ``{% include ... %}``
directive, template modules have a distinct namespace from their
containing template - they can only see the global template namespace
and their own keyword arguments.

Non-blocking, asynchronous requests
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When a request handler is executed, the request is automatically
finished. Since Tornado uses a non-blocking I/O style, you can override
this default behavior if you want a request to remain open after the
main request handler method returns using the
``tornado.web.asynchronous`` decorator.

When you use this decorator, it is your responsibility to call
``self.finish()`` to finish the HTTP request, or the user's browser will
simply hang:

::

    class MainHandler(tornado.web.RequestHandler):
        @tornado.web.asynchronous
        def get(self):
            self.write("Hello, world")
            self.finish()

Here is a real example that makes a call to the FriendFeed API using
Tornado's built-in asynchronous HTTP client:

::

    class MainHandler(tornado.web.RequestHandler):
        @tornado.web.asynchronous
        def get(self):
            http = tornado.httpclient.AsyncHTTPClient()
            http.fetch("http://friendfeed-api.com/v2/feed/bret",
                       callback=self.on_response)

        def on_response(self, response):
            if response.error: raise tornado.web.HTTPError(500)
            json = tornado.escape.json_decode(response.body)
            self.write("Fetched " + str(len(json["entries"])) + " entries "
                       "from the FriendFeed API")
            self.finish()

When ``get()`` returns, the request has not finished. When the HTTP
client eventually calls ``on_response()``, the request is still open,
and the response is finally flushed to the client with the call to
``self.finish()``.

For a more advanced asynchronous example, take a look at the `chat
example application
<https://github.com/facebook/tornado/tree/master/demos/chat>`_, which
implements an AJAX chat room using `long polling
<http://en.wikipedia.org/wiki/Push_technology#Long_polling>`_.  Users
of long polling may want to override ``on_connection_close()`` to
clean up after the client closes the connection (but see that method's
docstring for caveats).

Asynchronous HTTP clients
~~~~~~~~~~~~~~~~~~~~~~~~~

Tornado includes two non-blocking HTTP client implementations:
``SimpleAsyncHTTPClient`` and ``CurlAsyncHTTPClient``. The simple client
has no external dependencies because it is implemented directly on top
of Tornado's ``IOLoop``. The Curl client requires that ``libcurl`` and
``pycurl`` be installed (and a recent version of each is highly
recommended to avoid bugs in older version's asynchronous interfaces),
but is more likely to be compatible with sites that exercise little-used
parts of the HTTP specification.

Each of these clients is available in its own module
(``tornado.simple_httpclient`` and ``tornado.curl_httpclient``), as well
as via a configurable alias in ``tornado.httpclient``.
``SimpleAsyncHTTPClient`` is the default, but to use a different
implementation call the ``AsyncHTTPClient.configure`` method at startup:

::

    AsyncHTTPClient.configure('tornado.curl_httpclient.CurlAsyncHTTPClient')

Third party authentication
~~~~~~~~~~~~~~~~~~~~~~~~~~

Tornado's ``auth`` module implements the authentication and
authorization protocols for a number of the most popular sites on the
web, including Google/Gmail, Facebook, Twitter, and FriendFeed.
The module includes methods to log users in via these sites and, where
applicable, methods to authorize access to the service so you can, e.g.,
download a user's address book or publish a Twitter message on their
behalf.

Here is an example handler that uses Google for authentication, saving
the Google credentials in a cookie for later access:

::

    class GoogleHandler(tornado.web.RequestHandler, tornado.auth.GoogleMixin):
        @tornado.web.asynchronous
        def get(self):
            if self.get_argument("openid.mode", None):
                self.get_authenticated_user(self._on_auth)
                return
            self.authenticate_redirect()

        def _on_auth(self, user):
            if not user:
                self.authenticate_redirect()
                return
            # Save the user with, e.g., set_secure_cookie()

See the `tornado.auth` module documentation for more details.

Debug mode and automatic reloading
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you pass ``debug=True`` to the ``Application`` constructor, the app
will be run in debug mode. In this mode, templates will not be cached
and the app will watch for changes to its source files and reload itself
when anything changes. This reduces the need to manually restart the
server during development. However, certain failures (such as syntax
errors at import time) can still take the server down in a way that
debug mode cannot currently recover from.

Debug mode is not compatible with ``HTTPServer``'s multi-process mode.
You must not give ``HTTPServer.start`` an argument greater than 1 if you
are using debug mode.

The automatic reloading feature of debug mode is available as a
standalone module in ``tornado.autoreload``, and is optionally used by
the test runner in ``tornado.testing.main``.

Running Tornado in production
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

At FriendFeed, we use `nginx <http://nginx.net/>`_ as a load balancer
and static file server. We run multiple instances of the Tornado web
server on multiple frontend machines. We typically run one Tornado
frontend per core on the machine (sometimes more depending on
utilization).

When running behind a load balancer like nginx, it is recommended to
pass ``xheaders=True`` to the ``HTTPServer`` constructor. This will tell
Tornado to use headers like ``X-Real-IP`` to get the user's IP address
instead of attributing all traffic to the balancer's IP address.

This is a barebones nginx config file that is structurally similar to
the one we use at FriendFeed. It assumes nginx and the Tornado servers
are running on the same machine, and the four Tornado servers are
running on ports 8000 - 8003:

::

    user nginx;
    worker_processes 1;

    error_log /var/log/nginx/error.log;
    pid /var/run/nginx.pid;

    events {
        worker_connections 1024;
        use epoll;
    }

    http {
        # Enumerate all the Tornado servers here
        upstream frontends {
            server 127.0.0.1:8000;
            server 127.0.0.1:8001;
            server 127.0.0.1:8002;
            server 127.0.0.1:8003;
        }

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        access_log /var/log/nginx/access.log;

        keepalive_timeout 65;
        proxy_read_timeout 200;
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        gzip on;
        gzip_min_length 1000;
        gzip_proxied any;
        gzip_types text/plain text/html text/css text/xml
                   application/x-javascript application/xml
                   application/atom+xml text/javascript;

        # Only retry if there was a communication error, not a timeout
        # on the Tornado server (to avoid propagating "queries of death"
        # to all frontends)
        proxy_next_upstream error;

        server {
            listen 80;

            # Allow file uploads
            client_max_body_size 50M;

            location ^~ /static/ {
                root /var/www;
                if ($query_string) {
                    expires max;
                }
            }
            location = /favicon.ico {
                rewrite (.*) /static/favicon.ico;
            }
            location = /robots.txt {
                rewrite (.*) /static/robots.txt;
            }

            location / {
                proxy_pass_header Server;
                proxy_set_header Host $http_host;
                proxy_redirect false;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Scheme $scheme;
                proxy_pass http://frontends;
            }
        }
    }

WSGI and Google AppEngine
~~~~~~~~~~~~~~~~~~~~~~~~~

Tornado comes with limited support for `WSGI <http://wsgi.org/>`_.
However, since WSGI does not support non-blocking requests, you cannot
use any of the asynchronous/non-blocking features of Tornado in your
application if you choose to use WSGI instead of Tornado's HTTP server.
Some of the features that are not available in WSGI applications:
``@tornado.web.asynchronous``, the ``httpclient`` module, and the
``auth`` module.

You can create a valid WSGI application from your Tornado request
handlers by using ``WSGIApplication`` in the ``wsgi`` module instead of
using ``tornado.web.Application``. Here is an example that uses the
built-in WSGI ``CGIHandler`` to make a valid `Google
AppEngine <http://code.google.com/appengine/>`_ application:

::

    import tornado.web
    import tornado.wsgi
    import wsgiref.handlers

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, world")

    if __name__ == "__main__":
        application = tornado.wsgi.WSGIApplication([
            (r"/", MainHandler),
        ])
        wsgiref.handlers.CGIHandler().run(application)

See the `appengine example application
<https://github.com/facebook/tornado/tree/master/demos/appengine>`_ for a
full-featured AppEngine app built on Tornado.

