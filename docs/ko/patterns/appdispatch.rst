.. _app-dispatch:

어플리케이션 디스패칭
=======================

어플리케이션 디스패칭은 WSGI 레벨에서 복수의 플라스크 어플리케이션들을 결합하는 과정이다.
여러분은 플라스크 어플리케이션들을 더 큰 것으로 만들수 있을뿐 아니라
어떤 WSGI 어플리케이션으로도 결합할 수 있다.
이것은 여러분이 원하다면 심지어 같은 인터프리터에서 장고(Django) 와 플라스크 어플리케이션을
바로 옆에서 실행할 수 있게해준다. 이 유용함 어플리케이션들이 내부적으로 동작하는 방식에 의존한다.

:ref:`module approach <larger-applications>` 과 근본적인 차이점은 이 경우에 여러분은 
서로 완전히 분리된 동일하거나 다른 플라스크 어플리케이션들을 실행한다는 것이다.
그것들은 다른 설정에서 실행되고 WSGI 레벨에서 디스패치된다.


이 문서를 가지고 작업하기
--------------------------

아래의 각 기법들과 예제들은 어떤 WSGI 서버에서 실행가능한 ``application`` 객체이다.
운영 환경의 경우, :ref:`deployment` 를 살펴봐라.
개발 환경의 경우, 베르크쭉(Werkzeug)은 :func:`werkzeug.serving.run_simple` 에 개발용 빌트인 서버를 제공한다::

    from werkzeug.serving import run_simple
    run_simple('localhost', 5000, application, use_reloader=True)


:func:`run_simple <werkzeug.serving.run_simple>` 은 운영환경에서 사용을 의도하지 않는다.
운영환경에 맞는 기능이 갖춰진 서버(:ref:`full-blown WSGI server <deployment>`)를 사용해라. 

대화형(interactive) 디버거를 사용하기 위해서, 디버깅이 어플리케이션과 
심플 서버(simple server) 양쪽에서 활성화되어 있어야한다.
아래는 디버깅과  :func:`run_simple <werkzeug.serving.run_simple>` 를 사용한 
"hellow world" 예제이다::

    from flask import Flask
    from werkzeug.serving import run_simple

    app = Flask(__name__)
    app.debug = True

    @app.route('/')
    def hello_world():
        return 'Hello World!'

    if __name__ == '__main__':
        run_simple('localhost', 5000, app,
                   use_reloader=True, use_debugger=True, use_evalex=True)


어플리케이션 결합하기
----------------------

여러분이 완전하게 분리된 어플리케이션들을 갖고 있고 그것들이 동일한 파이썬 프로세스 위의
바로 옆에서 동작하기를 원한다면, :class:`werkzeug.wsgi.DispatcherMiddleware` 를 이용할 수 있다.
그 방식은 각 플라스크 어플리케이션이 유효한 WSGI 어플리케이션이고 디스패처 미들웨어에 의해
URL 접두어(prefix)에 기반해서 디스패치되는 하나의 더 커다란 어플리케이션으로 결합되는 것이다.

예를 들면, 여러분의 주(main) 어플리케이션을 '/'에 두고 백엔드 인터페이스는 `/backend`에 둘 수 있다::

    from werkzeug.wsgi import DispatcherMiddleware
    from frontend_app import application as frontend
    from backend_app import application as backend

    application = DispatcherMiddleware(frontend, {
        '/backend':     backend
    })


하위도메인(subdomain)으로 디스패치하기
--------------------------------------

여러분은 때때로 다른 구성으로 같은 어플리케이션에 대한 복수 개의 인스턴스를 
사용하고 싶을 때가 있을 것이다. 그 어플리케이션이 어떤 함수 안에서 생성됐고
여러분이 그 어플리케이션을 인스턴스화 하기위해 그 함수를 호출할 수 있다고 가정하면,
그런 방식은 굉장히 구현하기 쉽다. 함수로 새 인스턴스를 생성을 지원하도록 어플리케이션을
개발하기 위해서는 :ref:`app-factories` 패턴을 살펴보도록 해라.

매우 일반적인 예제는 하위도메인 별로 어플리케이션을 생성하는 것이다.
예를 들면, 여러분은 어플리케이션의 모든 하위도메인에 대한 모든 요청을
디스패치 하도록 웹서버를 구성하고 사용자 지정 인스턴스를 생성하기 위해
하위도메인 정보를 사용한다. 일단 여러분의 웹서버가 모든 하위도메인의 요청을
받아들이도록(listen) 설정하면, 여러분은 동적인 어플리케이션 생성을 할 수있는
매우 간단한 WSGI 어플리케이션을 사용할 수 있다.

이 관점에서 추상화의 최적의 레벨은 WSGI 계층이다. 여러분은 들어오는 요청을 
보고 그것을 여러분의 플라스크 어플리케이션으로 위임하는 여러분 자신만의 
WSGI 어플리케이션을 작성할 수 있다. 그 어플리케이션이 아직 존재하지 않는다면,
그것은 동적으로 생성되고 기억된다::

    from threading import Lock

    class SubdomainDispatcher(object):

        def __init__(self, domain, create_app):
            self.domain = domain
            self.create_app = create_app
            self.lock = Lock()
            self.instances = {}

        def get_application(self, host):
            host = host.split(':')[0]
            assert host.endswith(self.domain), 'Configuration error'
            subdomain = host[:-len(self.domain)].rstrip('.')
            with self.lock:
                app = self.instances.get(subdomain)
                if app is None:
                    app = self.create_app(subdomain)
                    self.instances[subdomain] = app
                return app

        def __call__(self, environ, start_response):
            app = self.get_application(environ['HTTP_HOST'])
            return app(environ, start_response)


그리고나서 이 디스패쳐는 아래와 같이 사용될 수 있다::

    from myapplication import create_app, get_user_for_subdomain
    from werkzeug.exceptions import NotFound

    def make_app(subdomain):
        user = get_user_for_subdomain(subdomain)
        if user is None:
            # if there is no user for that subdomain we still have
            # to return a WSGI application that handles that request.
            # We can then just return the NotFound() exception as
            # application which will render a default 404 page.
            # You might also redirect the user to the main page then
            return NotFound()

        # otherwise create the application for the specific user
        return create_app(user)

    application = SubdomainDispatcher('example.com', make_app)


경로로 디스패치하기
-------------------

URL 경로로 디스패치하는 것도 하위도메인과 굉장히 유사하다. 하위도메인 헤더를
확인하기 위해 `Host` 헤더를 보는 것 대신, 간단히 첫 번째 슬래쉬(/)까지의 
요청 경로를 보는 것이다::

    from threading import Lock
    from werkzeug.wsgi import pop_path_info, peek_path_info

    class PathDispatcher(object):

        def __init__(self, default_app, create_app):
            self.default_app = default_app
            self.create_app = create_app
            self.lock = Lock()
            self.instances = {}

        def get_application(self, prefix):
            with self.lock:
                app = self.instances.get(prefix)
                if app is None:
                    app = self.create_app(prefix)
                    if app is not None:
                        self.instances[prefix] = app
                return app

        def __call__(self, environ, start_response):
            app = self.get_application(peek_path_info(environ))
            if app is not None:
                pop_path_info(environ)
            else:
                app = self.default_app
            return app(environ, start_response)

경로와 하위도메인 디스패치의 큰 차이점은 경로 디스패치는 생성 함수가 `None`
을 반환하면 다른 어플리케이션으로 넘어갈 수 있다는 것이다::

    from myapplication import create_app, default_app, get_user_for_prefix

    def make_app(prefix):
        user = get_user_for_prefix(prefix)
        if user is not None:
            return create_app(user)

    application = PathDispatcher(default_app, make_app)
