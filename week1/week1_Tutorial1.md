# Tutorial 1:Serialization

## Setting up a new enviroment

```python
#가상환경 생성, venv사용 
python -m venv env
env\Scripts\activate # Window
#가상환경 안에 필요한 패키지 설치
pip install django
pip install djangorestframework
pip install pygments # 코드 하이라이팅 하기위한 패키지

```
> NOTE: 가상환경을 종료하고 싶으면 deactivate 

> 가상환경 문서 - [virtualenv 문서](https://virtualenv.pypa.io/en/latest/index.html)

## Getting started
```python
cd~
django-admin startproject tutorial
cd tutorial

#Web API생성
python manage.py startapp snippets

#tutorial/settings.py에 INSTALLED_APPS 추가
'rest_framework'.
'snippets.apps.SnippetsConfig',

```

## creating a model to work with
code snippets를 저장하기위해 사용되는 Snippets모델을 만들어야 합니다. Snippets/models.py를 수정

```python
from django.db import models
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted((item, item) for item in get_all_styles())


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    class Meta:
        ordering = ('created',)
```
모델 작성 뒤 snippets model에 대한 migration필요, 그 뒤 데이터베이스 동기화

```pyhton
python manage.py makemigrations snippets
python manage.py migrate
```

## creating a Serializer class
Web API에서 처음으로 해야할 일은 snippet인스턴스를 JSON과 같은 형태로 시리얼라이즈, 디시리얼라이즈 하는 방법을 제공, Django의 form과 비슷한 역할을 하는 serializers를 정의하여 사용할 수 있다. snippets 디렉토리 내부에 serializers.py파일을 생성하여 코드 삽입.
```python
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES

class SnipptSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template':'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    def create(self, validated_data):
        '''
        검증한 데이터로 새 'Snippet'인스턴스를 생성하여 리턴
        '''
        return Snippet.objects.create(**validated_data)

    def update(self, instance, valied_data):
        '''
        검증한 데이터로 기존 'Snippet'인스턴스를 업데이트 한 후 리턴
        '''
        instance.title = validated_data.get('title',instance.title)
        instance.code = validated_data.get('code',instance.code)
        instance.linenos = validated_data.get('linenos',instance.linenos)
        instance.language = validated_data.get('language',instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance


```
serializer 클래스의 첫파트는 필드를 정의하는 것 입니다. create(), update()함수는 serializer.save()가 호출되었을 때 인스턴스가 생성되거나 수정되는 방법을 정의

required, max_length, default와 같이 다양한 필드에 비슷한 확인 플래그가 있는 것들은 serializer클래스는 Django의 Form클래스와 매우 비슷합니다. 

필드 플래그는 HTML을 랜더링할 때와 같이 특정 상황에서 serializer가 어떻게 출력되는지 컨트롤 해줍니다. 위에있는 {'base_template':'textarea.html'}플래그는 Django Form의 widget=widgets.Textarea와 동일합니다. 해당 플래그는 탐색가능한 API를 표시하는 방법을 제어하는데 유용

We can actually also save ourselves some time by using the ModelSerializer class, as we'll see later, but for now we'll keep our serializer definition explicit. - ModelSerializer 클래스를 사용하면 이런 기능을 일일히 구현하지 않아도된다.

## Working with Serializers
Django Shell 띄우기

```pyhton
python manage.py shell
```

필요한 패키지들을 import하고, 코드조각(snippet 클래스의 인스턴스)를 만들어 봅시다.
```python
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

snippet = Snippet(code='foo = "bar"\n')
snippet.save()

snippet = Snippet(code='print("hello, world")\n')
snippet.save()
```

snippet 인스턴스가 만들어졌으니, 이 인스턴스중 하나를 직렬화
```python
serializer = SnippetSerializer(snippet)
serializer.data
# {'id': 2, 'title': '', 'code': 'print("hello, world")\n', 'linenos': False, 'language': 'python', 'style': 'friendly'}

```

모델 인스턴스를 파이썬의 데이터 타입으로 변환했습니다. 직렬화 과정을 마무리 하려면 이 데이터를 json으로 변환
```python
content = JSONRenderer().render(serializer.data)  
content
# b'{"pk":2,"title":"","code":"print \\"hello, world\\"\\n","linenos":false,"language":"python","style":"friendly"}'
```

반직렬화
먼저, 파이썬 데이터 타입을 파싱
```python
from django.utils.six import BytesIO

stream = BytesIO(content)  
data = JSONParser().parse(stream)
```

이 데이터를 인스턴스화
```python
serializer = SnippetSerializer(data=data)  
serializer.is_valid()  
# True
serializer.validated_data  
# OrderedDict([('title', ''), ('code', 'print "hello, world"'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
serializer.save()  
# <Snippet: Snippet object>
```

뷰를 작성할 뿐만 아니라 serializer를 사용하는 방식도 form을 다루는 방식과 유사
모델의인스턴스 뿐만 아니라 쿼리셋도 직렬화 가능 serializer의 인자에 many=True만추가 하면됨

```python
serializer = SnippetSerializer(Snippet.objects.all(), many=True)  
serializer.data
[OrderedDict([('pk', 1), ('title', ''), ('code', 'foo = "bar"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('pk', 2), ('title', ''), ('code', 'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('pk', 3), ('title', ''), ('code', 'print "hello, world"'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])]
```
# Using ModelSerializers

SnippetSerializer 클래스는 snippet 모델의 정보들을 그대로 복사합니다. Django에서 form클래스와 ModelForm 클래스를 제공하듯이, REST프레임워크에서도 Serializer 클래스와 ModelSerializer 클래스 제공

## 앞에서 만든 Serializer가 ModelSerializer 클래스를 사용하도록 리팩토링

```python
# snippets - serializers.py
# SnippetSerializer 클래스 수정

class SnippetSerializer(serializers.ModelSerializer):  
    class Meta:
        model = Snippet
        fields = ('id', 'title', 'code', 'linenos', 'language', 'style')
```

serializer에 프로퍼티 하나만 정의한 후 시리얼라이저 인스턴스를 출력해보면 모든 필드 확인가능

```python 
# python manage.py shell

>>> from snippets.serializers import SnippetSerializer
>>> serializer = SnippetSerializer()
>>> print(repr(serializer))
SnippetSerializer():  
    id = IntegerField(label='ID', read_only=True)
    title = CharField(allow_blank=True, max_length=100, required=False)
    code = CharField(style={'base_template': 'textarea.html'})
    linenos = BooleanField(required=False)
    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...
```
ModelSerializer 클래스는 Serializer 클래스의 단축버전
> - 필드를 자동으로 인식
> - create() 메소드와 update() 메소드가 이미 구현

# Writing regular Django views using our Serializer
앞에서 새로 만든 serializer class를 뷰에서 어떻게사용하나?
restframeowrk를 사용하지않고, 일반적인 Django 뷰 형태로 만듭니다.

HttpResponse의 하위 클래스를 만들고, 받은 데이터를 모두 json형태로 반환
```python
#snippets/views.py

from django.http import HttpResponse  
from django.views.decorators.csrf import csrf_exempt  
from rest_framework.renderers import JSONRenderer  
from rest_framework.parsers import JSONParser  
from snippets.models import Snippet  
from snippets.serializers import SnippetSerializer

class JSONResponse(HttpResponse):  
    """
    콘텐츠를 JSON으로 변환한 후 HttpResponse 형태로 반환합니다.
    """
    def __init__(self, data, **kwargs):
        content = JSONRenderer().render(data)
        kwargs['content_type'] = 'application/json'
        super(JSONResponse, self).__init__(content, **kwargs)
```
우리가 만들 API의 최상단에서는 저장된 코드 조각을 모두 보여주며, 새 코드 조각을 만들 수도 있습니다.
```python
@csrf_exempt
def snippet_list(request):  
    """
    코드 조각을 모두 보여주거나 새 코드 조각을 만듭니다.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JSONResponse(serializer.data)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data, status=201)
        return JSONResponse(serializer.errors, status=400)
```
>JSONReponse 클래스와 동등한 위치에 있어야한다.

인증되지 않은 사용자도 이뷰에 POST를 할 수 있도록 csrf_exempt데코레이터를 적어둠
보통의 경우 필요 없을 수도 있고, REST프레임워크의 뷰가 이보다 더 정밀한 속성을 제공하기도 하지만, 일단 여기서는 우리가 구현하고 싶은 기능을 csrf_exempt가 담당

코드 조각 하나늘 보여줄 뷰도 필요합니다. 또 이 코드 조각을 업데이트하거나 삭제할 수도 있어야 합니다.

```python
@csrf_exempt
def snippet_detail(request, pk):  
    """
    코드 조각 조회, 업데이트, 삭제
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return JSONResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data)
        return JSONResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        snippet.delete()
        return HttpResponse(status=204)
```
> JSONResponse 클래스와 동등한 위치에 있어야한다.
마지막으로 이뷰들과 URL 연결
```python
# snippets - urls.py

from django.conf.urls import url
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
]
```
json의 내용이 깨졌거나 뷰가 처리 할 수 없는 메소드 요청인 경우, '500 서버 오류'를 보게 될 것입니다.

# Testing our first attempt at a Web API
코드 조각을 보여주는 서버 구동
셀 종료
```python
quit()
```
Django의 개발 서버 작동
```python
python manage.py runserver
```
다른 터미널 창에서 서버를 테스트, 테스트에는 curl이나 httpie를 사용, Httpie는 파이썬으로 작성된 사용자 친화적인 http클라이언트
```python
pip install httpie
```
마지막으로 코드 조각 전체를 가져옵니다.
```python

http http://127.0.0.1:8000/snippets/

HTTP/1.1 200 OK  
...
[
  {
    "id": 1,
    "title": "",
    "code": "foo = \"bar\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  },
  {
    "id": 2,
    "title": "",
    "code": "print \"hello, world\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  }
]
```
id를 지정하여 특정 코드 조각만 가져올 수있습니다.
```python
http http://127.0.0.1:8000/snippets/2/

HTTP/1.1 200 OK  
...
{
  "id": 2,
  "title": "",
  "code": "print \"hello, world\"\n",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}
```
웹브라우저에서도 똑같은 JSON데이터 확인 가능

### 새로알게된 사실
```
   ChoiceField
    -- set of choices의 value를 받을 수 있는 필드
    choices-... arg를 가진 ModelSerializer를 사용하면 자동생성
    Signature : ChoiceField(chocies)
        - choices:(key, display_name): 튜플의 리스트
        - alow_blank: default는 False, True면 empty string도 valid value로 취급됨. False면 validation error
        - html_cutoff: default=None, 만약 세팅되면 HTML 셀렉트 드롭다운에서 나타나는 최대 숫자 조절 가능
        - html_cutoff_text: ???????????
        allow_blank, allow_null: 둘다 사용 가능하지만 둘다 쓰지말고 한개만 쓸 것
        allow_blank는 textural choices에, allow_null은 numeric, non-textual choices에 쓰길 권유

    직렬화
    1. 어떠한 정보가 들어가있는 객체를 만들면 Object형태로 메모리에 저장
    2. 해당 Object를 가공 없이 이동하고 네트워크로 전송을 할 수 없음
    3. 그렇기 때문에 "직렬화"과정을 거쳐 "XML/JSON"으로의 변환
    4. "XML/JSON"을 다시 Object로 뽑아 내는 것을 "반직렬화"
        - 저장된 데이터를 다시 파싱하여 적절한 멤버 변수에 넣어주는 역할

```
>   [Field](https://www.django-rest-framework.org/api-guide/fields/#choicefield)

>  [HTTP상태코드](https://paikgyeong.tistory.com/m/5)

# Question

1. INSTALLED_APPS 에서 snippets 앱 추가할때 'snippets'와 'snippets.apps.SnippetsConfig'의 차이
2. serializers.py 에서 serializer, instance, validated_data의 의미
3. We can actually also save ourselves some time by using the ModelSerializer class, as we'll see later, but for now we'll keep our serializer definition explicit. - ModelSerializer 클래스를 사용하면 이런 기능을 일일히 구현하지 않아도된다. - 이런 기능(?), 그럼왜 serializer를 쓰지? 그냥 ModelSerializer만 사용하면 되지않나.
