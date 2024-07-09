## FastAPI Best Practices <!-- omit from toc -->
스타트업에서 사용하는 모범 사례와 관례에 대한 의견 목록입니다.

지난 몇 년 동안 프로덕션을 운영하면서 개발자 경험에 큰 영향을 미친 좋은 결정과 나쁜 결정을 내려왔습니다.
그중 일부는 공유할 만한 가치가 있습니다.

## Contents  <!-- omit from toc -->
- [프로젝트 구조](#프로젝트-구조)
- [비동기 Routes](#비동기-routes)
  - [I/O 집약적 작업](#io-집약적-작업)
  - [CPU 집약적 작업](#cpu-집약적-작업)
- [Pydantic](#pydantic)
  - [Pydantic 초월](#Pydantic-초월)
  - [사용자 정의 Base Model](#사용자-정의-Base-Model)
  - [Pydantic BaseSettings 분리](#Pydantic-BaseSettings-분리)
- [종속성](#종속성)
  - [종속성 주입 저 너머로](#종속성-주입-저-너머로)
  - [종속성 연결](#종속성-연결)
  - [종속성 분리 및 재사용. 종속성 호출이 캐시됨](#종속성-분리-및-재사용-종속성-호출이-캐시됨)
  - [가급적 async 종속성을 우선](#가급적-async-종속성을-우선)
- [기타](#기타)
  - [REST를 따릅시다](#REST를-따릅시다)
  - [FastAPI 응답 직렬화](#FastAPI-응답-직렬화)
  - [동기화 SDK를 사용해야 하는 경우 스레드 풀에서 실행합니다.](#동기화-SDK를-사용해야-하는-경우-스레드-풀에서-실행합니다)
  - [ValueErrors가 Pydantic ValidationError가 될 수 있습니다.](#ValueErrors가-Pydantic-ValidationError가-될-수-있습니다)
  - [Docs](#docs)
  - [DB 키 명명 규칙 설정](#DB-키-명명-규칙-설정)
  - [Migrations. Alembic](#migrations-alembic)
  - [DB 명명 규칙 설정](#DB-명명-규칙-설정)
  - [SQL 우선. Pydantic 차선](#SQL-우선-Pydantic-차선)
  - [개발 0일차부터 테스트 클라이언트를 비동기화 합시다.](#개발-0일차부터-테스트-클라이언트를-비동기화-합시다)
  - [ruff를 사용합시다](#ruff를-사용합시다)
- [보너스 섹션](#보너스-섹션)

## 프로젝트 구조
프로젝트를 구성하는 방법에는 여러 가지가 있지만, 가장 좋은 구조는 일관되고 간단하며 돌발 상황이 없는 구조입니다.

많은 예제 프로젝트와 튜토리얼은 파일 유형(예: CRUD, 라우터, 모델)별로 프로젝트를 나누는데, 이는 마이크로서비스나 범위가 적은 프로젝트에 적합합니다. 하지만 이 접근 방식은 도메인과 모듈이 많은 모놀리스에는 적합하지 않았습니다.

이 문서에서 제안하는 [프로젝트 구조](#프로젝트-구조)는 확장 및 진화에 용이하며 Netflix의 [Dispatch](https://github.com/Netflix/dispatch)에서 영감을 받아 약간의 수정을 통해 탄생했습니다.
```
fastapi-project
├── alembic/
├── src
│   ├── auth
│   │   ├── router.py
│   │   ├── schemas.py  # pydantic models
│   │   ├── models.py  # db models
│   │   ├── dependencies.py
│   │   ├── config.py  # local configs
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   ├── service.py
│   │   └── utils.py
│   ├── aws
│   │   ├── client.py  # client model for external service communication
│   │   ├── schemas.py
│   │   ├── config.py
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   └── utils.py
│   └── posts
│   │   ├── router.py
│   │   ├── schemas.py
│   │   ├── models.py
│   │   ├── dependencies.py
│   │   ├── constants.py
│   │   ├── exceptions.py
│   │   ├── service.py
│   │   └── utils.py
│   ├── config.py  # global configs
│   ├── models.py  # global models
│   ├── exceptions.py  # global exceptions
│   ├── pagination.py  # global module e.g. pagination
│   ├── database.py  # db connection related stuff
│   └── main.py
├── tests/
│   ├── auth
│   ├── aws
│   └── posts
├── templates/
│   └── index.html
├── requirements
│   ├── base.txt
│   ├── dev.txt
│   └── prod.txt
├── .env
├── .gitignore
├── logging.ini
└── alembic.ini
```
1. 모든 도메인 디렉토리를 `src` 폴더에 저장합니다.
   1. `src/` - 앱의 최상위 레벨, 공통 모델, 구성 및 상수 등을 포함합니다.
   2. `src/main.py` - 프로젝트의 루트로, FastAPI 앱을 초기화합니다.
2. 각 패키지에는 자체 라우터, 스키마, 모델 등이 있습니다.
   1. `router.py` - 모든 엔드포인트가 있는 각 모듈의 핵심입니다.
   2. `schemas.py` - Pydantic 모델용
   3. `models.py` - DB 모델용
   4. `service.py` - 모듈별 비즈니스 로직  
   5. `dependencies.py` - 라우터 종속성
   6. `constants.py` - 모듈별 상수 및 오류 코드
   7. `config.py` - 예: 환경 변수
   8. `utils.py` - 非비즈니스 로직 함수(예: 응답 정규화, 데이터 강화 등)
   9. `exceptions.py` - 모듈별 예외 처리(예: `PostNotFound`, `InvalidUserData`)
3. 패키지에 다른 패키지의 서비스나 종속성 또는 상수가 필요한 경우 - 명시적인 모듈 이름으로 가져옵니다.
```python
from src.auth import constants as auth_constants
from src.notifications import service as notification_service
from src.posts.constants import ErrorCode as PostsErrorCode  # 각 패키지의 상수 모듈에 표준 오류 코드가 있는 경우 
```

## 비동기 Routes
FastAPI는 애초에 비동기 프레임워크입니다. 비동기 I/O 작업과 함께 작동하도록 설계되었기 때문에 매우 빠릅니다. 

그러나 FastAPI는 `비동기` routes만 사용하도록 제한하지 않으며, 개발자는 `동기` routes도 사용할 수 있습니다. 초보 개발자는 이 두 가지가 동일하다고 혼동할 수 있지만 그렇지 않습니다.


### I/O 집약적 작업
내부적으로 FastAPI는 비동기 및 동기 I/O 작업을 모두 [효과적으로 처리](https://fastapi.tiangolo.com/async/#path-operation-functions) 할 수 있습니다. 
- FastAPI는 [스레드풀](https://en.wikipedia.org/wiki/Thread_pool)에서 `sync` routes를 실행하며, `blocking I/O 작업`은 [이벤트 루프](https://docs.python.org/3/library/asyncio-eventloop.html)가 작업을 실행하는 것을 멈추지 않습니다.
- Routes가 `async`로 정의된 경우 `await`를 통해 정기적으로 호출되며, FastAPI는 사용자가 `non-blocking I/O 작업`만 수행할 것임을 신뢰합니다.

주의할 점은 사용자가 해당 신뢰에 배신하여 `async` routes 내에서 blocking 작업을 실행하면 해당 blocking 작업이 완료될 때까지 이벤트 루프가 다음 작업을 실행할 수 없다는 것입니다.
```python
import asyncio
import time

from fastapi import APIRouter


router = APIRouter()


@router.get("/terrible-ping")
async def terrible_ping():
    time.sleep(10) # I/O blocking operation for 10 seconds, the whole process will be blocked
    
    return {"pong": True}

@router.get("/good-ping")
def good_ping():
    time.sleep(10) # I/O blocking operation for 10 seconds, but in a separate thread for the whole `good_ping` route

    return {"pong": True}

@router.get("/perfect-ping")
async def perfect_ping():
    await asyncio.sleep(10) # non-blocking I/O operation

    return {"pong": True}

```
**호출시 일어나는 일들:**
1. `GET /terrible-ping`
   1. FastAPI 서버가 요청을 수신하고 처리를 시작합니다.
   2. 서버의 이벤트 루프와 대기열의 모든 작업은 `time.sleep()`이 완료될 때까지 대기합니다.
      1. 서버는 `time.sleep()`이 I/O 작업이 아니라고 판단하여 완료될 때까지 대기합니다.
      2. 대기하는 동안 서버는 새로운 요청을 수락하지 않습니다.
   3. 서버가 응답을 반환합니다.
      1. 응답 후 서버가 새 요청을 수락하기 시작합니다.
2. `GET /good-ping`
   1. FastAPI 서버가 요청을 수신하고 처리를 시작합니다.
   2. FastAPI는 전체 경로 `good_ping`을 작업자 스레드가 함수를 실행할 스레드 풀로 보냅니다.
   3. `good_ping`이 실행되는 동안 이벤트 루프는 큐에서 다음 작업을 선택하고 작업합니다(예: 새 요청 수락, DB 호출).
      - 메인 스레드(예: FastAPI 앱)와는 별개로 작업자 스레드는 `time.sleep()`이 완료될 때까지 대기합니다.
      - 동기화 작업은 메인 스레드가 아닌 사이드 스레드만 차단합니다.
   4. `good_ping`이 작업을 완료하면 서버는 클라이언트에 응답을 반환합니다.
3. `GET /perfect-ping`
   1. FastAPI 서버가 요청을 수신하고 처리를 시작합니다.
   2. FastAPI가 `asyncio.sleep(10)`을 기다립니다.
   3. 이벤트 루프가 대기열에서 다음 작업을 선택하고 작업합니다(예: 새 요청 수락, DB 호출).
   4. `asyncio.sleep(10)`이 완료되면 서버는 route 실행을 완료하고 클라이언트에 응답을 반환합니다.

> [!WARNING]
> 스레드 풀에 대한 참고 사항:
>  - 스레드는 코루틴보다 더 많은 리소스를 필요로 하므로 비동기 I/O 작업만큼 저렴하지 않습니다.
>  - 스레드 풀에는 스레드 수가 제한되어 있으므로 스레드가 부족해지면 앱이 느려질 수 있습니다. [Read more](https://github.com/Kludex/fastapi-tips?tab=readme-ov-file#2-be-careful-with-non-async-functions) (external link)

### CPU 집약적 작업
두 번째 주의 사항은 'non-blocking awaitables'이거나 스레드 풀로 전송되는 작업은 I/O 집약적인 작업(예: 파일 열기, DB 호출, 외부 API 호출)이어야 한다는 점입니다.
- CPU 집약적인 작업(예: 무거운 계산, 데이터 처리, 비디오 트랜스코딩)을 기다리는 것은 CPU가 작업을 완료하기 위해 작동해야 하는 반면,
I/O 작업은 외부에 있고 서버는 해당 작업이 완료될 때까지 기다리는 동안 아무것도 하지 않으므로 다음 작업으로 이동할 수 있기 때문에 쓸모가 없습니다.
- 다른 스레드에서 CPU 집약적인 작업을 실행하는 것도 [GIL](https://realpython.com/python-gil/)로 인해 효과적이지 않습니다.
요컨대, `GIL`은 한 번에 하나의 스레드만 작동하도록 허용하므로 CPU 작업에는 쓸모가 없습니다.
- CPU 집약적인 작업을 최적화하려면 다른 프로세스의 작업자에게 보내야 합니다.

**혼란스러워하는 사용자들의 StackOverflow 질문**
1. https://stackoverflow.com/questions/62976648/architecture-flask-vs-fastapi/70309597#70309597
   - 여기에서 [제 답변](https://stackoverflow.com/a/70309597/6927498)을 확인할 수 있습니다.
2. https://stackoverflow.com/questions/65342833/fastapi-uploadfile-is-slow-compared-to-flask
3. https://stackoverflow.com/questions/71516140/fastapi-runs-api-calls-in-serial-instead-of-parallel-fashion

## Pydantic
### Pydantic 초월
Pydantic에는 데이터 유효성 검사 및 변환을 위한 다양한 기능이 있습니다.

기본값이 있는 필수 및 비필수 필드와 같은 일반적인 기능 외에도
Pydantic에는 정규식, 열거형, 문자열 조작, 이메일 유효성 검사 등과 같은 포괄적인 데이터 처리 도구가 내장되어 있습니다.
```python
from enum import Enum
from pydantic import AnyUrl, BaseModel, EmailStr, Field


class MusicBand(str, Enum):
   AEROSMITH = "AEROSMITH"
   QUEEN = "QUEEN"
   ACDC = "AC/DC"


class UserBase(BaseModel):
    first_name: str = Field(min_length=1, max_length=128)
    username: str = Field(min_length=1, max_length=128, pattern="^[A-Za-z0-9-_]+$")
    email: EmailStr
    age: int = Field(ge=18, default=None)  # must be greater or equal to 18
    favorite_band: MusicBand | None = None  # only "AEROSMITH", "QUEEN", "AC/DC" values are allowed to be inputted
    website: AnyUrl | None = None
```
### 사용자 정의 Base Model
제어 가능한 글로벌 Base Model을 사용하면 앱 내의 모든 모델을 사용자 지정할 수 있습니다. 예를 들어 표준 날짜/시간 형식을 적용하거나 Base Model의 모든 하위 클래스에 공통 메서드를 도입할 수 있습니다.
```python
from datetime import datetime
from zoneinfo import ZoneInfo

from fastapi.encoders import jsonable_encoder
from pydantic import BaseModel, ConfigDict


def datetime_to_gmt_str(dt: datetime) -> str:
    if not dt.tzinfo:
        dt = dt.replace(tzinfo=ZoneInfo("UTC"))

    return dt.strftime("%Y-%m-%dT%H:%M:%S%z")


class CustomModel(BaseModel):
    model_config = ConfigDict(
        json_encoders={datetime: datetime_to_gmt_str},
        populate_by_name=True,
    )

    def serializable_dict(self, **kwargs):
        """Return a dict which contains only serializable fields."""
        default_dict = self.model_dump()

        return jsonable_encoder(default_dict)


```
위의 예에서는 글로벌 Base Model을 만들기로 했습니다:
In the example above, we have decided to create a global base model that:
- 모든 날짜/시간 필드를 명시적인 표준 시간대를 사용하여 표준 형식으로 직렬화합니다.
- 직렬화 가능한 필드만 있는 딕셔너리를 반환하는 메서드를 제공합니다.

### Pydantic BaseSettings 분리
BaseSettings는 환경 변수를 읽기 위한 훌륭한 혁신이었지만, 전체 앱에 대해 하나의 BaseSettings를 사용하면 시간이 지남에 따라 지저분해질 수 있습니다. 유지 관리와 정리를 개선하기 위해 여러 모듈과 도메인에 BaseSettings를 분할했습니다.
```python
# src.auth.config
from datetime import timedelta

from pydantic_settings import BaseSettings


class AuthConfig(BaseSettings):
    JWT_ALG: str
    JWT_SECRET: str
    JWT_EXP: int = 5  # minutes

    REFRESH_TOKEN_KEY: str
    REFRESH_TOKEN_EXP: timedelta = timedelta(days=30)

    SECURE_COOKIES: bool = True


auth_settings = AuthConfig()


# src.config
from pydantic import PostgresDsn, RedisDsn, model_validator
from pydantic_settings import BaseSettings

from src.constants import Environment


class Config(BaseSettings):
    DATABASE_URL: PostgresDsn
    REDIS_URL: RedisDsn

    SITE_DOMAIN: str = "myapp.com"

    ENVIRONMENT: Environment = Environment.PRODUCTION

    SENTRY_DSN: str | None = None

    CORS_ORIGINS: list[str]
    CORS_ORIGINS_REGEX: str | None = None
    CORS_HEADERS: list[str]

    APP_VERSION: str = "1.0"


settings = Config()

```

## 종속성
### 종속성 주입 저 너머로
Pydantic은 훌륭한 스키마 유효성 검사기이지만 데이터베이스나 외부 서비스를 호출하는 복잡한 유효성 검사의 경우 충분하지 않습니다.

FastAPI 문서는 종속성을 엔드포인트에 대한 종속성 주입 위주로 제시하지만, request 유효성 검사에도 탁월합니다.

종속성은 데이터베이스 제약 조건에 대해 데이터를 검증하는 데 사용할 수 있습니다(예: 이메일이 이미 존재하는지 확인, 사용자가 있는지 확인 등).
```python
# dependencies.py
async def valid_post_id(post_id: UUID4) -> dict[str, Any]:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()

    return post


# router.py
@router.get("/posts/{post_id}", response_model=PostResponse)
async def get_post_by_id(post: dict[str, Any] = Depends(valid_post_id)):
    return post


@router.put("/posts/{post_id}", response_model=PostResponse)
async def update_post(
    update_data: PostUpdate,  
    post: dict[str, Any] = Depends(valid_post_id), 
):
    updated_post = await service.update(id=post["id"], data=update_data)
    return updated_post


@router.get("/posts/{post_id}/reviews", response_model=list[ReviewsResponse])
async def get_post_reviews(post: dict[str, Any] = Depends(valid_post_id)):
    post_reviews = await reviews_service.get_by_post_id(post["id"])
    return post_reviews
```
데이터 유효성 검사를 종속성에 적용하지 않았다면 모든 엔드포인트에 대해 `post_id`가 존재하는지 확인하고
각각에 대해 동일한 테스트를 작성해야 했을 것입니다.

### 종속성 연결
종속성은 다른 종속성을 사용하여 유사한 로직에 대한 코드 반복을 피할 수 있습니다.
```python
# dependencies.py
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt

async def valid_post_id(post_id: UUID4) -> dict[str, Any]:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()

    return post


async def parse_jwt_data(
    token: str = Depends(OAuth2PasswordBearer(tokenUrl="/auth/token"))
) -> dict[str, Any]:
    try:
        payload = jwt.decode(token, "JWT_SECRET", algorithms=["HS256"])
    except JWTError:
        raise InvalidCredentials()

    return {"user_id": payload["id"]}


async def valid_owned_post(
    post: dict[str, Any] = Depends(valid_post_id), 
    token_data: ddict[str, Any] = Depends(parse_jwt_data),
) -> dict[str, Any]:
    if post["creator_id"] != token_data["user_id"]:
        raise UserNotOwner()

    return post

# router.py
@router.get("/users/{user_id}/posts/{post_id}", response_model=PostResponse)
async def get_user_post(post: dict[str, Any] = Depends(valid_owned_post)):
    return post

```
### 종속성 분리 및 재사용. 종속성 호출이 캐시됨
종속성은 여러 번 재사용할 수 있으며 다시 계산되지 않습니다. -  FastAPI는 기본적으로 요청 범위 내에서 종속성의 결과를 캐시합니다.
즉, 하나의 경로에서 `valid_post_id`가 여러 번 호출되는 경우 한 번만 호출됩니다.

이를 알면 더 작은 도메인에서 작동하고 다른 경로에서 재사용하기 쉬운 여러 개의 작은 함수로 종속성을 분리할 수 있습니다.
예를 들어, 아래 코드에서는 `parse_jwt_data`를 세 번 사용하고 있습니다:
1. `valid_owned_post`
2. `valid_active_creator`
3. `get_user_post`,

하지만 `parse_jwt_data`는 첫 번째 호출에서 한 번만 호출됩니다.

```python
# dependencies.py
from fastapi import BackgroundTasks
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt

async def valid_post_id(post_id: UUID4) -> Mapping:
    post = await service.get_by_id(post_id)
    if not post:
        raise PostNotFound()

    return post


async def parse_jwt_data(
    token: str = Depends(OAuth2PasswordBearer(tokenUrl="/auth/token"))
) -> dict:
    try:
        payload = jwt.decode(token, "JWT_SECRET", algorithms=["HS256"])
    except JWTError:
        raise InvalidCredentials()

    return {"user_id": payload["id"]}


async def valid_owned_post(
    post: Mapping = Depends(valid_post_id), 
    token_data: dict = Depends(parse_jwt_data),
) -> Mapping:
    if post["creator_id"] != token_data["user_id"]:
        raise UserNotOwner()

    return post


async def valid_active_creator(
    token_data: dict = Depends(parse_jwt_data),
):
    user = await users_service.get_by_id(token_data["user_id"])
    if not user["is_active"]:
        raise UserIsBanned()
    
    if not user["is_creator"]:
       raise UserNotCreator()
    
    return user
        

# router.py
@router.get("/users/{user_id}/posts/{post_id}", response_model=PostResponse)
async def get_user_post(
    worker: BackgroundTasks,
    post: Mapping = Depends(valid_owned_post),
    user: Mapping = Depends(valid_active_creator),
):
    """Get post that belong the active user."""
    worker.add_task(notifications_service.send_email, user["id"])
    return post

```

### 가급적 async 종속성을 우선
FastAPI는 `sync` 및 `async` 종속성을 모두 지원하며, 대기할 필요가 없는 경우 `sync` 종속성을 사용하고 싶은 유혹이 있지만 이는 최선의 선택이 아닐 수 있습니다.
Routes와 마찬가지로 `sync` 종속성은 스레드 풀에서 실행됩니다. 그리고 여기서도 스레드에는 가격과 제한이 따르며, 작은 非 I/O 작업만 수행하는 경우 중복됩니다.

[See more](https://github.com/Kludex/fastapi-tips?tab=readme-ov-file#9-your-dependencies-may-be-running-on-threads) (external link)


## 기타
### REST를 따릅시다
RESTful API를 개발하면 이와 같은 routes에서 종속성을 더 쉽게 재사용할 수 있습니다:
   1. `GET /courses/:course_id`
   2. `GET /courses/:course_id/chapters/:chapter_id/lessons`
   3. `GET /chapters/:chapter_id`

유일한 주의 사항은 path에 동일한 변수 이름을 사용해야 한다는 것입니다:
The only caveat is to use the same variable names in the path:

- 주어진 'profile_id'의 존재 여부를 확인하는 `GET /profiles/:profile_id`와
`GET /creators/:creator_id` 엔드포인트가 둘 있지만 `GET /creators/:creator_id`도
프로필이 크리에이터인지 확인하는 경우 `creator_id` path 변수 이름을
`profile_id`로 바꾸고 이 두 종속성을 연결하는 것이 더 낫습니다.
```python
# src.profiles.dependencies
async def valid_profile_id(profile_id: UUID4) -> Mapping:
    profile = await service.get_by_id(profile_id)
    if not profile:
        raise ProfileNotFound()

    return profile

# src.creators.dependencies
async def valid_creator_id(profile: Mapping = Depends(valid_profile_id)) -> Mapping:
    if not profile["is_creator"]:
       raise ProfileNotCreator()

    return profile

# src.profiles.router.py
@router.get("/profiles/{profile_id}", response_model=ProfileResponse)
async def get_user_profile_by_id(profile: Mapping = Depends(valid_profile_id)):
    """Get profile by id."""
    return profile

# src.creators.router.py
@router.get("/creators/{profile_id}", response_model=ProfileResponse)
async def get_user_profile_by_id(
     creator_profile: Mapping = Depends(valid_creator_id)
):
    """Get creator's profile by id."""
    return creator_profile

```
### FastAPI 응답 직렬화
일부 최적화를 위해 route의 `response_model`과 일치하는 Pydantic 객체를 반환할 수 있다고 생각한다면
이는 잘못된 생각입니다.

FastAPI는 먼저 해당 pydantic 객체를 `jsonable_encoder`를 사용하여 dict로 변환한 다음
`response_model`을 사용하여 데이터의 유효성을 검사한 다음 객체를 JSON으로 직렬화합니다.
```python
from fastapi import FastAPI
from pydantic import BaseModel, root_validator

app = FastAPI()


class ProfileResponse(BaseModel):
    @model_validator(mode="after")
    def debug_usage(self):
        print("created pydantic model")

        return self


@app.get("/", response_model=ProfileResponse)
async def root():
    return ProfileResponse()
```
**Logs Output:**
```
[INFO] [2022-08-28 12:00:00.000000] created pydantic model
[INFO] [2022-08-28 12:00:00.000020] created pydantic model
```

### 동기화 SDK를 사용해야 하는 경우 스레드 풀에서 실행합니다.
라이브러리를 사용하여 외부 서비스와 상호 작용해야 하고 `async`가 아닌 경우
외부 작업자 스레드에서 HTTP 호출을 수행하세요.

스타렛의 잘 알려진 `run_in_threadpool`을 사용할 수 있습니다.
```python
from fastapi import FastAPI
from fastapi.concurrency import run_in_threadpool
from my_sync_library import SyncAPIClient 

app = FastAPI()


@app.get("/")
async def call_my_sync_library():
    my_data = await service.get_my_data()

    client = SyncAPIClient()
    await run_in_threadpool(client.make_request, data=my_data)
```

### ValueErrors가 Pydantic ValidationError가 될 수 있습니다.
클라이언트가 직접 직면하는 Pydantic 스키마에서 `ValueError`를 발생시키면,
사용자에게 멋진 상세 응답을 반환합니다.
```python
# src.profiles.schemas
from pydantic import BaseModel, field_validator

class ProfileCreate(BaseModel):
    username: str
    
    @field_validator("password", mode="after")
    @classmethod
    def valid_password(cls, password: str) -> str:
        if not re.match(STRONG_PASSWORD_PATTERN, password):
            raise ValueError(
                "Password must contain at least "
                "one lower character, "
                "one upper character, "
                "digit or "
                "special symbol"
            )

        return password


# src.profiles.routes
from fastapi import APIRouter

router = APIRouter()


@router.post("/profiles")
async def get_creator_posts(profile_data: ProfileCreate):
   pass
```
**Response Example:**

<img src="images/value_error_response.png" width="400" height="auto">

### Docs
1. API가 공개되지 않는 한 기본적으로 문서를 숨깁니다. 지정된 `.envs`에서만 명시적으로 표시합니다.
```python
from fastapi import FastAPI
from starlette.config import Config

config = Config(".env")  # parse .env file for env variables

ENVIRONMENT = config("ENVIRONMENT")  # get current env name
SHOW_DOCS_ENVIRONMENT = ("local", "staging")  # explicit list of allowed envs

app_configs = {"title": "My Cool API"}
if ENVIRONMENT not in SHOW_DOCS_ENVIRONMENT:
   app_configs["openapi_url"] = None  # set url for docs as null

app = FastAPI(**app_configs)
```
2. 이해하기 쉬운 문서를 생성하는 FastAPI 도움말
   1. `response_model`, `status_code`, `description` 등을 설정합니다.
   2. 모델과 상태가 다른 경우 `responses` route 속성을 사용하여 다른 응답에 대한 문서를 추가합니다.
```python
from fastapi import APIRouter, status

router = APIRouter()

@router.post(
    "/endpoints",
    response_model=DefaultResponseModel,  # default response pydantic model 
    status_code=status.HTTP_201_CREATED,  # default status code
    description="Description of the well documented endpoint",
    tags=["Endpoint Category"],
    summary="Summary of the Endpoint",
    responses={
        status.HTTP_200_OK: {
            "model": OkResponse, # custom pydantic model for 200 response
            "description": "Ok Response",
        },
        status.HTTP_201_CREATED: {
            "model": CreatedResponse,  # custom pydantic model for 201 response
            "description": "Creates something from user request ",
        },
        status.HTTP_202_ACCEPTED: {
            "model": AcceptedResponse,  # custom pydantic model for 202 response
            "description": "Accepts request and handles it later",
        },
    },
)
async def documented_route():
    pass
```
이와 같은 문서가 생성됩니다:
![FastAPI Generated Custom Response Docs](images/custom_responses.png "Custom Response Docs")

### DB 키 명명 규칙 설정
데이터베이스의 규칙에 따라 인덱스 이름을 명시적으로 설정하는 것이 sqlalchemy보다 낫습니다.
```python
from sqlalchemy import MetaData

POSTGRES_INDEXES_NAMING_CONVENTION = {
    "ix": "%(column_0_label)s_idx",
    "uq": "%(table_name)s_%(column_0_name)s_key",
    "ck": "%(table_name)s_%(constraint_name)s_check",
    "fk": "%(table_name)s_%(column_0_name)s_fkey",
    "pk": "%(table_name)s_pkey",
}
metadata = MetaData(naming_convention=POSTGRES_INDEXES_NAMING_CONVENTION)
```
### Migrations. Alembic
1. 마이그레이션은 정적이고 되돌릴 수 있어야 합니다.
마이그레이션이 동적으로 생성된 데이터에 의존하는 경우,
동적인 것은 데이터 구조가 아니라 데이터 자체뿐인지 확인하세요.
2. 설명이 포함된 이름과 슬러그로 마이그레이션을 생성합니다. 슬러그는 필수이며 변경 사항을 설명해야 합니다.
3. 새 마이그레이션에 대해 사람이 읽을 수 있는 파일 템플릿을 설정합니다. `date*_*slug*.py` 패턴(예: `2022-08-24_post_content_idx.py`)을 사용합니다.
```
# alembic.ini
file_template = %%(year)d-%%(month).2d-%%(day).2d_%%(slug)s
```
### DB 명명 규칙 설정
이름에 일관성을 유지하는 것이 중요합니다. 우리는 몇 가지 규칙을 따랐습니다:
1. lower_case_snake
2. 단수형 표현(=singular form) (e.g. `post`, `post_like`, `user_playlist`)
3. 모듈 접두사를 사용하여 유사한 테이블을 그룹화합니다(예: `payment_account`, `payment_bill`, `post`, `post_like`)
4. 테이블 전체에 일관성을 유지하되 구체적인 이름을 지정해도 괜찮습니다, 예를 들자면
   1. 모든 테이블에 `profile_id`를 사용하되, 일부 테이블에 크리에이터인 프로필만 필요한 경우 `creator_id`를 사용합니다.
   2. `post_like`, `post_view`와 같은 모든 추상 테이블에는 `post_id`를 사용하되 `chapters.course_id`의 `course_id`와 같은 관련 모듈에서는 구체적인 이름을 사용합니다.
5. datetime에 대한 `_at` 접미사
6. date에 대한 `_date` 접미사
### SQL 우선. Pydantic 차선
- 일반적으로 데이터베이스는 CPython보다 훨씬 빠르고 깔끔하게 데이터 처리를 처리합니다.
- 복잡한 조인과 간단한 데이터 조작은 모두 SQL로 수행하는 것이 좋습니다.
- 중첩된 객체가 있는 응답의 경우 DB에서 JSON을 집계하는 것이 좋습니다.
  = It's preferable to aggregate JSONs in DB for responses with nested objects. ()
```python
# src.posts.service
from typing import Any

from pydantic import UUID4
from sqlalchemy import desc, func, select, text
from sqlalchemy.sql.functions import coalesce

from src.database import database, posts, profiles, post_review, products

async def get_posts(
    creator_id: UUID4, *, limit: int = 10, offset: int = 0
) -> list[dict[str, Any]]: 
    select_query = (
        select(
            (
                posts.c.id,
                posts.c.slug,
                posts.c.title,
                func.json_build_object(
                   text("'id', profiles.id"),
                   text("'first_name', profiles.first_name"),
                   text("'last_name', profiles.last_name"),
                   text("'username', profiles.username"),
                ).label("creator"),
            )
        )
        .select_from(posts.join(profiles, posts.c.owner_id == profiles.c.id))
        .where(posts.c.owner_id == creator_id)
        .limit(limit)
        .offset(offset)
        .group_by(
            posts.c.id,
            posts.c.type,
            posts.c.slug,
            posts.c.title,
            profiles.c.id,
            profiles.c.first_name,
            profiles.c.last_name,
            profiles.c.username,
            profiles.c.avatar,
        )
        .order_by(
            desc(coalesce(posts.c.updated_at, posts.c.published_at, posts.c.created_at))
        )
    )
    
    return await database.fetch_all(select_query)

# src.posts.schemas
from typing import Any

from pydantic import BaseModel, UUID4

   
class Creator(BaseModel):
    id: UUID4
    first_name: str
    last_name: str
    username: str


class Post(BaseModel):
    id: UUID4
    slug: str
    title: str
    creator: Creator

    
# src.posts.router
from fastapi import APIRouter, Depends

router = APIRouter()


@router.get("/creators/{creator_id}/posts", response_model=list[Post])
async def get_creator_posts(creator: dict[str, Any] = Depends(valid_creator_id)):
   posts = await service.get_posts(creator["id"])

   return posts
```
### 개발 0일차부터 테스트 클라이언트를 비동기화 합시다.
DB로 통합 테스트를 작성하면 나중에 이벤트 루프 오류가 발생할 가능성이 높습니다.
비동기 테스트 클라이언트를 즉시 설정하세요(예: [httpx](https://github.com/encode/starlette/issues/652))
```python
import pytest
from async_asgi_testclient import TestClient

from src.main import app  # inited FastAPI app


@pytest.fixture
async def client() -> AsyncGenerator[TestClient, None]:
    host, port = "127.0.0.1", "9000"

    async with AsyncClient(transport=ASGITransport(app=app, client=(host, port)), base_url="http://test") as client:
        yield client


@pytest.mark.asyncio
async def test_create_post(client: TestClient):
    resp = await client.post("/posts")

    assert resp.status_code == 201
```
동기화 DB 연결이 없거나(뭐라구요?) 통합 테스트를 작성할 계획이 없다면 말입니다.

### ruff를 사용합시다
린터를 사용하면 코드 서식 지정은 잊고 비즈니스 로직 작성에 집중할 수 있습니다.

[Ruff](https://github.com/astral-sh/ruff)는 black, autoflake, isort를 대체하고 600개 이상의 린트 규칙을 지원하는 "엄청나게 빠른" 새로운 린터입니다.

`pre-commit hooks`를 사용하는 것이 일반적인 모범 사례이지만 저희는 스크립트를 사용하는 것만으로도 충분했습니다.

```shell
#!/bin/sh -e
set -x

ruff --fix src
ruff format src
```

## 보너스 섹션
몇몇 친절한 분들이 자신의 경험과 모범 사례를 공유해 주셨으니 꼭 읽어보시기 바랍니다.
[프로젝트의 이슈 섹션](https://github.com/zhanymkanov/fastapi-best-practices/issues)에서 확인해 보세요.

예를 들어, [lowercase00](https://github.com/zhanymkanov/fastapi-best-practices/issues/4)님께서는
권한 및 인증, 클래스 기반 서비스 및 보기, 작업 대기열, 사용자 정의 응답 직렬화 기능자,
dynaconf를 사용한 구성 등을 사용한 모범 사례를 자세히 설명했습니다.

좋은 것이든 나쁜 것이든 FastAPI를 사용한 경험에 대해 공유할 것이 있으면
새 이슈를 만들어 주세요. 기꺼이 읽어드리겠습니다.
