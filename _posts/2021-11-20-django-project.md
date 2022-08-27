---
layout: post
title:  "Troubleshooting & Tips for Django Projects"
subtitle: "외주 프로젝트를 진행하면서: Troubleshooting & Tips"
date:   2021-11-20 19:22:00 +0900
comments: True
---

전역하고 2년 만에 다시 개발을 하면서, 전에는 기계적으로 했던 것들도 기억이 가물가물해 일일이 찾아보면서 진행했다.

새로 프로젝트를 시작하면 실제 개발하는 시간이 3할, 오류 검색하는 시간이 7할 이상 된다고 느낄 정도로 Troubleshooting의 비중이 크다.
그러나 모든 오류에 대한 대처법을 외울 수도 없고, 또 수학문제처럼 항상 막히는 부분에서 막히기 때문에 프로젝트를 할 때마다 이를 정리하면 추후에 유용하게 쓸 수 있을 것 같다.


## 1. 프로젝트 설계

#### React + DRF (DjangoRestFramework)

본 프로젝트는 내가 백엔드, 후배가 프론트를 맡았다. 본래 django 자체는 Frontend와 Backend의 경계가 모호한 framework이다. Django의 template 및 다양한 rendering 기능들이 이를 뒷받침한다. 물론 그런 기능을 들어내고 API View만 구현할 수 있지만 꽤나 비효율적이다.
그럼에도 Back과 Front의 구분의 수요는 여전히 있기에 DRF를 이용한다.

#### Project structure

React와 DRF를 연동하는 방법은 여러가지가 있다. 첫번째로 고려한 것은 Django Project의 template 폴더에 빌드된 react site를 넣어 렌더링 하는 방법이었다. 이 경우 Django app을 2개를 만들어 하나는 api용, 하나는 main으로 frontend-side를 렌더링하는 용도로 쓰는 것이었다.
이 경우 프로젝트를 일원화하여 관리할 수 있다는 장점이 있지만, URL 매핑하기가 꽤나 곤란해진다. SPA Project에는 적절하지만 그 외에는 부적절하다고 생각해 폐기했다.

두번째는 이를 분리하여 Django는 오로지 api server만 담당하고, frontend는 빌드된 react project를 정적 웹 호스팅 서비스를 이용하는 방법이다. 이 방법이 React+DRF의 Standard라고 생각한다. 그리고 Django에서 static&media 파일들을 서빙하기 위해서는 어차피 S3 같은 호스팅이 필요하기 때문에, 한 버킷에서 둘을 한번에 관리할 수 있어 생각보다 깔끔하게 완성할 수 있다.

#### Emailing Service

본 프로젝트에서 이메일 서비스 구현을 필요로 했다. django 자체에 이메일을 보낼 수 있는 기능이 있는 것으로 아는데, 다량의 메일을 보낼 경우 매우 높은 확률로 스팸처리가 되는 이슈가 있기에, AWS SES를 이용하여 문제를 해결한다.



## 2. Django Project Setting

#### virtualenv

```python
$ python -m venv <가상환경이름>         #python3 부터 venv 모듈 내장
$ source <가상환경이름>/bin/activate
```

#### Django Setting

```python
$ pip install django
$ django-admin startproject <프로젝트이름>
$ django-admin startapp <앱이름>
```

## 3. Django 관련

#### Django Admin

- [Django Admin Customizing](https://wayhome25.github.io/django/2017/03/22/django-ep8-django-admin/)
- django-jet

  훌륭한 Admin Theme이지만 세세한 Customizing이 어려워 포기, Django admin에 추가적인 기능(이메일링) 도입을 필요로 했기에 부적절

- django admin stylize

  `site-packages/django/contrib/admin/template`에 있는 기본 template html을 참고하여, templates 폴더에 생성하면 override 가능

  {% raw %}

  ```python
  # templates/admin/base.html (example)
  {% extends "admin/base.html" %}
  {% load static %}

  {% block extrastyle %}
  <link rel="stylesheet" type="text/css" href="{% static "admin.css" %}">
  {% endblock %}
  ```

  ```python
  # templates/admin/base_site.html (example)
  {% extends "admin/base.html" %}
  {% load static %}

  {% block title %}SNULING Admin Site{% endblock %}

  {% block branding %}
  <h1 id="site-name"><a href="{% url 'admin:index' %}"><img src="{% static 'img/snuling_logo.png' %}"/></a></h1>
  {% endblock %}

  {% block nav-global %}{% endblock %}
  ```

  {% endraw %}

- [django admin, extending admin with custom views](https://stackoverflow.com/questions/35875454/django-admin-extending-admin-with-custom-views)

#### Authorization

- 로그인 필요 View Decorator for CBV

  ```python
  def login_required_cbv(method):
      def decorated(self, request, *args, **kwargs):
          if request.user.is_anonymous:
              return Response(status=status.HTTP_401_UNAUTHORIZED)
          return method(self, request, *args, **kwargs)
      return decorated
  ```

- DRF Django Authentication 원리

  1. `'rest_framework.authtoken'` `INSTALLED_APPS`에 추가 후 migration
  2. View에서 `django.contrib.auth.authentication()`을 통해 user를 식별하고, User의 Token을 가져옴
  3. 회원가입시 또는 로그인시 Token을 반환하면, User는 HTTP Request Header에 `Authorization: Token {token}` 추가
  4. Header에 해당 Token이 있을 경우 view에서 `request.user`를 통해 식별 가능.

#### 기능 구현

- [Django에서 WYSIWYG 에디터 사용하기(ckeditor)](https://bookpark.github.io/2018-02-01/django-ckeditor)
- [django signal](https://dgkim5360.tistory.com/entry/django-signal-example)

  [documentation](https://docs.djangoproject.com/en/3.2/topics/signals/)

- [Django Database Backup](https://stackoverflow.com/questions/21049330/how-to-backup-a-django-db)

#### DRF 기본

- [django rest framework 를 위한 JSON 직렬화](https://ssungkang.tistory.com/entry/Django-django-rest-framework-%EB%A5%BC-%EC%9C%84%ED%95%9C-JSON-%EC%A7%81%EB%A0%AC%ED%99%94?category=320582) - ssung.k 님
- [APIView, Mixins, generics APIView, ViewSet을 알아보자](https://ssungkang.tistory.com/entry/Django-APIView-Mixins-generics-APIView-ViewSet%EC%9D%84-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90) - ssung.k 님

#### Troubleshooting

- [React에서 Django에 API 요청 시 CORS Policy: No 'Access-Control-Allow-Origin' 에러 해결](https://developer0809.tistory.com/21)

  개발 과정에서 3000 <-> 8000 포트 간 통신 필요할때 유용

- media file url 관련 문제

  검색해봐도 마땅한 해결 방법이 없어 꽤나 고민했던 문제였다. API server를 이용할 경우, 업로드한 이미지 파일들은 django project에서 지정한 media storage로 저장되고 해당 이미지 파일들의 url(`<img>` 태그에서의 src)은 `/media/XXXX/XXX.jpg`와 같은 형태로 저장된다.
  media storage의 host가 `www.A.com`이고, 프로젝트 사이트의 host가 `www.B.com`일 경우, 프로젝트 사이트에서 불러오는 이미지의 최종 주소는 후자의 host와 앞의 endpoint를 합쳐 `www.B.com/media/XXXX/XXX.jpg`라는 엉뚱한 주소로 호출하게 되어 이미지가 보이지 않는다.

  이는 media storage와 react 사이트를 하나의 bucket으로 관리함으로서, 같은 host를 가지게 하여 해결하였다.

- Database read only error
  `sudo chown <user> <filename>`으로 권한 추가


## 4. 서버 배포, AWS 관련

- [django 에서 S3에 Static, media 파일 저장하고 사용하기](https://blog.myungseokang.dev/posts/django-use-s3/)
-
- [AWS SES Sample Code](https://docs.aws.amazon.com/ses/latest/dg/send-an-email-using-sdk-programmatically.html#send-an-email-using-sdk-programmatically-examples)

- [AWS EC2로 배포 - uWSGI, nginx 까지](https://nachwon.github.io/django-deploy-1-aws/)
  - uWSGI log error - "no such file ~~~~" : `/var/log/~~~` 에 logging 할 폴더를 만들어줘야함.

## 5. 기타 소프트웨어, 툴 관련

#### Postman

- [Postman으로 API문서 만들기](https://velog.io/@jinee/TIL-Postman%EC%9C%BC%EB%A1%9C-API%EB%AC%B8%EC%84%9C-%EB%A7%8C%EB%93%A4%EA%B8%B0-l4k5mj31rl)

#### Troubleshooting

- [PUTTY - NO SUPPORTED AUTHENTICATION METHODS AVAILABLE](https://yooniversal.github.io/blog/post176/)
  링크된 오류는 아니었고, ssh 접속 시 username을 `ubuntu`로 해주지 않아서 생기는 오류였다... (AWS 기본은 ec2-user)
