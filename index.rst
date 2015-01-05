flask-mail
======================================

.. module:: flask-mail

在 Web 应用中的一个最基本的功能就是能够给用户发送邮件。

**Flask-Mail** 扩展提供了一个简单的接口，可以在 `Flask`_ 应用中设置 SMTP 使得可以在视图以及脚本中发送邮件信息。

Links
-----

* `documentation <http://packages.python.org/Flask-Mail/>`_
* `source <http://github.com/mattupstate/flask-mail>`_
* :doc:`changelog </changelog>`

安装 Flask-Mail
---------------------

使用 **pip** 或者 **easy_install** 安装 Flask-Mail::

    pip install Flask-Mail

或者从版本控制系统（github）中下载最新的版本::

    git clone https://github.com/mattupstate/flask-mail.git
    cd flask-mail
    python setup.py install

如果你正在使用 **virtualenv**，假设你会安装 flask-mail 在运行你的 Flask 应用程序的同一个 virtualenv 上。


配置 Flask-Mail
----------------------

**Flask-Mail** 使用标准的 Flask 配置 API 进行配置。下面这些是可用的配置型(每一个将会在文档中进行解释):

* **MAIL_SERVER** : 默认为 **'localhost'**

* **MAIL_PORT** : 默认为 **25**

* **MAIL_USE_TLS** : 默认为 **False**

* **MAIL_USE_SSL** : 默认为 **False**

* **MAIL_DEBUG** : 默认为 **app.debug**

* **MAIL_USERNAME** : 默认为 **None**

* **MAIL_PASSWORD** : 默认为 **None**

* **MAIL_DEFAULT_SENDER** : 默认为 **None**

* **MAIL_MAX_EMAILS** : 默认为 **None**

* **MAIL_SUPPRESS_SEND** : 默认为 **app.testing**

* **MAIL_ASCII_ATTACHMENTS** : 默认为 **False**

另外，**Flask-Mail** 使用标准的 Flask 的 ``TESTING`` 配置项用于单元测试(下面会具体介绍)。

邮件是通过一个 ``Mail`` 实例进行管理::

    from flask import Flask
    from flask_mail import Mail

    app = Flask(__name__)
    mail = Mail(app)

在这个例子中所有的邮件将会使用传入到 ``Mail`` 实例中的应用程序的配置项进行发送。

或者你也可以在应用程序配置的时候设置你的 ``Mail`` 实例，通过使用 **init_app** 方法::

    mail = Mail()

    app = Flask(__name__)
    mail.init_app(app)

在这个例子中邮件将会使用 Flask 的 ``current_app`` 中的配置项进行发送。如果你有多个具有不用配置项的多个应用运行在同一程序的时候，这种设置方式是十分有用的，


发送邮件
----------------

为了能够发送邮件，首先需要创建一个 ``Message`` 实例::

    from flask_mail import Message

    @app.route("/")
    def index():

        msg = Message("Hello",
                      sender="from@example.com",
                      recipients=["to@example.com"])

你能够设置一个或者多个收件人::

    msg.recipients = ["you@example.com"]
    msg.add_recipient("somebodyelse@example.com")

如果你设置了 ``MAIL_DEFAULT_SENDER``，就不必再次填写发件人，默认情况下将会使用配置项的发件人::

    msg = Message("Hello",
                  recipients=["to@example.com"])

如果 ``sender`` 是一个二元组，它将会被分成姓名和邮件地址::

    msg = Message("Hello",
                  sender=("Me", "me@example.com"))

    assert msg.sender == "Me <me@example.com>"

邮件内容可以包含主体以及/或者 HTML::

    msg.body = "testing"
    msg.html = "<b>testing</b>"

最后，发送邮件的时候请使用 Flask 应用设置的 ``Mail`` 实例::

    mail.send(msg)


大量邮件
-----------

通常在一个 Web 应用中每一个请求会同时发送一封或者两封邮件。在某些特定的场景下，有可能会发送数十或者数百封邮件，不过这种发送工作会给交离线任务或者脚本执行。

在这种情况下发送邮件的代码会有些不同::

    with mail.connect() as conn:
        for user in users:
            message = '...'
            subject = "hello, %s" % user.name
            msg = Message(recipients=[user.email],
                          body=message,
                          subject=subject)

            conn.send(msg)


与电子邮件服务器的连接会一直保持活动状态直到所有的邮件都已经发送完成后才会关闭（断开）。

有些邮件服务器会限制一次连接中的发送邮件的上限。你可以设置重连前的发送邮件的最大数，通过配置 **MAIL_MAX_EMAILS** 。


附件
-----------

在邮件中添加附件同样非常简单::

    with app.open_resource("image.png") as fp:
        msg.attach("image.png", "image/png", fp.read())

具体细节请参看 `API`_ 。

If ``MAIL_ASCII_ATTACHMENTS`` is set to **True**, filenames will be converted to
an ASCII equivalent. This can be useful when using a mail relay that modify mail
content and mess up Content-Disposition specification when filenames are UTF-8
encoded. The conversion to ASCII is a basic removal of non-ASCII characters. It
should be fine for any unicode character that can be decomposed by NFKD into one
or more ASCII characters. If you need romanization/transliteration (i.e `ß` →
`ss`) then your application should do it and pass a proper ASCII string.


单元测试以及禁止发送邮件
---------------------------------

When you are sending messages inside of unit tests, or in a development
environment, it's useful to be able to suppress email sending.

If the setting ``TESTING`` is set to ``True``, emails will be
suppressed. Calling ``send()`` on your messages will not result in
any messages being actually sent.

Alternatively outside a testing environment you can set ``MAIL_SUPPRESS_SEND`` to **True**. This
will have the same effect.

However, it's still useful to keep track of emails that would have been
sent when you are writing unit tests.

In order to keep track of dispatched emails, use the ``record_messages``
method::

    with mail.record_messages() as outbox:

        mail.send_message(subject='testing',
                          body='test',
                          recipients=emails)

        assert len(outbox) == 1
        assert outbox[0].subject == "testing"

The **outbox** is a list of ``Message`` instances sent.

The blinker package must be installed for this method to work.

Note that the older way of doing things, appending the **outbox** to
the ``g`` object, is now deprecated.


头注入
----------------

To prevent `header injection <http://www.nyphp.org/PHundamentals/8_Preventing-Email-Header-Injection>`_ attempts to send
a message with newlines in the subject, sender or recipient addresses will result in a ``BadHeaderError``.

信号量
------------------

.. versionadded:: 0.4

**Flask-Mail** now provides signalling support through a ``email_dispatched`` signal. This is sent whenever an email is
dispatched (even if the email is not actually sent, i.e. in a testing environment).

A function connecting to the ``email_dispatched`` signal takes a ``Message`` instance as a first argument, and the Flask
app instance as an optional argument::

    def log_message(message, app):
        app.logger.debug(message.subject)

    email_dispatched.connect(log_message)


API
---

.. module:: flask_mail

.. autoclass:: Mail
   :members: send, connect, send_message

.. autoclass:: Attachment

.. autoclass:: Connection
   :members: send, send_message

.. autoclass:: Message
   :members: attach, add_recipient

.. _Flask: http://flask.pocoo.org
.. _GitHub: http://github.com/mattupstate/flask-mail
