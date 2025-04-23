---
title: "drf-spectacular 에서 query param을 이용한 version 별 api 표기"
date: 2025-04-23T16:53:56+09:00
draft: false
---

API 개발 하다보면 하위호환성을 지키기 위해 versioning 을 해야할 때가 있다.

drf 에서는 여러가지 versioning 방법을 제공 중인데, swagger url 을 설정하기 번거롭다.

때문에 drf 에서는 원하는 versioning 을 정의하고 swagger 에서는 QueryParameterVersioning 으로 접근 가능한 방법을 소개한다.

### drf setting

```python
REST_FRAMEWORK = {
    ...
    "DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.AcceptHeaderVersioning",
    "DEFAULT_VERSION": "1",
    "ALLOWED_VERSIONS": ["1", "2"],
}
```

예제에서는 drf 에서는 accept header 를 이용한 versioning 을 사용한다. 그 외는 [문서 참조](https://www.django-rest-framework.org/api-guide/versioning/)

### drf-spectacular url 정의 및 setting

```python
from rest_framework.versioning import QueryParameterVersioning

urlpatterns += [
    ...
    path('schema/', SpectacularAPIView.as_view(versioning_class=QueryParameterVersioning), name='schema'),
    path('schema/swagger-ui/', SpectacularSwaggerView.as_view(url_name='api:schema'), name='swagger-ui'),
    path('schema/redoc/', SpectacularRedocView.as_view(url_name='api:schema'), name='redoc'),
]
```

swagger-ui 페이지가 query param에 의해 version 을 변경할 수 있도록 view 에 versioning_class 를 QueryParameterVersioning 으로 정의한다.

이제 schema view 에 한 해서 query param으로 version 을 변경 할 수 있다.

키는 default 'version' 이고, 값은 drf setting 에 정의한 ALLOWED_VERSIONS 이다.

```python
SPECTACULAR_SETTINGS = {
    ...
    "SERVE_INCLUDE_SCHEMA": False,
    "SWAGGER_UI_SETTINGS": '''{
        deepLinking: true,
        urls: [{url: "/api/schema/?version=1", name: "v1"}, {url: "/api/schema/?version=2", name: "v2"}],
        presets: [SwaggerUIBundle.presets.apis, SwaggerUIStandalonePreset],
        layout: "StandaloneLayout",
        persistAuthorization: true,  
        displayOperationId: true,
        filter: true 
    }''',
}
```

swagger-ui 페이지에 상단바를 추가하고 urls 를 변경할 수 있도록 한다.

### view 의 extend_schema 추가

```python
@extend_schema(
    summary="조회",
    versions=["1"],
    responses={
        status.HTTP_200_OK: V1Serializer,
    },
)
@extend_schema(
    summary="조회 v2",
    versions=["2"],
    responses={
        status.HTTP_200_OK: V2Serializer,
    },
)
def retrieve(self, request):
    if request.verion == "1":
        ...
    elif request.version == "2":
        ...
```

위와 같이 구성하면 swagger-ui 페이지에서 상단 바를 이용하여 version 을 변경할 경우 

변경된 version 에 맞게 extend_schema 에 정의 된 구성이 나오게 된다.

마찬가지로 try it out 을 통해 실행 해 볼 경우 accept header 에 그에 맞는 version 이 붙게 되어 테스트 가능하다.
