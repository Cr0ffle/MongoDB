# #2 몽고DB 기본

- 몽고DB 기본 개념
  - 몽고 DB 데이터의 기본 단위는 도큐먼트이고 이는 관계형 데이터베이스의 행과 유사하지만 더 강력하다.
  - 같은 맥락에서 몽고DB의 컬렉션은 동적 스키마가 있는 RDB 테이블과 같다.
  - 몽고DB단일 인스턴스는 자체적인 컬렉션을 갖는 여러개의 독립적인 데이터베이스를 호스팅한다.
  - 모든 도큐먼트는 컬렉션 내에서 고유한 특수키인 "_id"를 가진다.
  - 몽고DB는 몽고 셸이라는 간단하지만 강력한 도구와 함께 배포된다 몽고셸은 몽고DB인스턴스를 관리하고 몽고DB 쿼리 언어로 데이터를 조작하기 위한 내장 지원을 제공한다 또한 사용자가 다양한 목적으로 자신의 스크립트를 만들고 로드할 수 있는 완전한 기능의 자바스크립트 해석기다.

## 도큐먼트

- 몽고DB의 핵심
  - 정렬된 키와 연결된 값으로 이뤄진 도큐먼트

```json
// "greeting" 이라는 키에 "Hello, world!"라는 값을 가진 도큐먼트
{"greeting": "Hello, world!"}

// "greeting" 이라는 키에 "Hello, world!"라는 값을 갖고 "view"라는 키에 3이라는 값을 가진 도큐먼트
{"greeting": "Hello, world!", "view" : 3}
```

- 도뮤컨트의 키는 문자열이다. 다음 예외 몇가지를 제외하면 어떤 UTF-8 문자든 쓸 수 있다.
  - 키는 \\0(null)을 포함하지 않는다 \\0은 키의 끝을 나타내는 데 사용된다.
  - .과 $문자는 몇가지 특별한 속성을 가지며 특정 상황에서만 사용해야 한다. 이들은 예약어이기 떄문에 부적절하게 사용하면 드라이버에서 경고를 발생한다.
- 몽고 DB는 데이터형과 대소문자를 구별한다.
- 또한 몽고DB에서는 키가 중복될 수 없다.

```json
{"count": 5}
{"count": "5"}

{"count": 5}
{"Count": 5}
```

## 컬렉션

- 컬렉션은 도큐먼트의 모음이다.
- 몽고 DB의 도큐먼트가 관계형 데이터베이스의 행에 대응된다면 컬렉션은 테이블에 대응된다고 볼 수 있다.

### 동적 스키마

- 컬렉션은 동적 스키마를 가진다.

  - 하나의 컬렉션 내 도큐먼트들이 모두 다른 구조를 가질 수 있다.

  - ```json
    {"greeting": "Hello, world!", "view" : 3}
    {"signoff": "Good night, and good luck"}
    ```

- 그럼에도 하나의 컬렉션에 다양한 도큐먼트들을 넣지 않는 이유

  - 같은 컬렉션에 다른 종류의 도큐먼트를 저장하면 개발자와 관리자에게 번거로운 일이 생길수도 있다.
  - 컬렉션별로 목록을 뽑으면 한 컬렉션 내 특정 데이터형별로 쿼리해서 목록을 뽑을 때보다 훨씬 빠르다.
  - 같은 종류의 데이터를 하나의 컬렉션에 모아두면 데이터 지역성에 좋다.
  - 인덱스를 만드려면 도큐먼트는 특정 구조를 가져야 한다. 인덱스는 컬렉션별로 정의한다. 같은 유형의 도큐먼트를 하나의 컬렉션에 넣음으로써 컬렉션을 효율적으로 인덱싱할 수 있다.

### 네이밍

- 컬렉션은 이름으로 식별된다. 몇가지 제약조건
  - 빈 문자열은 유효한 컬렉션 명이 아니다.
  - \\0(null)문자는 컬렉션명의 끝을 나타내는 문자이므로 컬렉션명에 사용할 수 없다.
  - system.으로 시작하는 컬렉션명은 시스템 컬렉션에서 사용하는 예약어이므로 사용할 수 없다. 예를 들어 system.users 컬렉션에는 데이터베이스 사용자 정보가, system.namespace 컬렉션에는 데이터베이스 내 모든 컬렉션의 정보가 들어있다.
  - 사용자가 만든 컬렉션은 이름에 예약어인 $를 포함할 수 없다.

`서브 컬렉션`

- 서브컬렉션은 네임스페이스에 .(마침표) 문자를 사용해 컬렉션을 체계화할 수 있다.
  - ex: blog.post, blog.authors
  - 이는 단지 체계화하기 위함이며 blog 컬렉션이나 자식 컬렉션과는 아무런 관계가 없다.
- 서브컬렉션은 특별한 속성은 없지만 여러 몽고DB 툴에서 지원하므로 유용하다.
  - 큰 파일을 저장하는 프로토콜인 GridFS는 콘텐츠 데이터와 별도로 메타데이터를 저장하는 데 서브컬렉션을 사용한다. (GridFS는 6장에서 자세히 .. )
  - 대부분의 드라이버는 특정 컬렉션의 서브컬렉션에 접근하는 몇 가지 편리한 문법을 제공한다.
- 서브컬렉션은 몽고DB의 데이터를 체계화하는 훌륭한 방법이다.

## 데이터베이스

- 몽고 DB는 데이터베이스(스키마)에 컬렉션을 그룹지어 놓는다.
  - 데이터베이스 > 컬렉션 > 도큐먼트
- 몽고 DB의 단일 인스턴스와 데이터베이스
  - 몽고 DB의 단일 인스턴스는 여러 데이터베이스를 호스팅할 수 있다.
  - 각 데이터베이스를 완전히 독립적으로 취급할 수 있다.
- 데이터베이스 명에대한 몇가지 제약
  - 빈 문자열은은 유효한 데이터베이스 이름이 아니다.
  - 데이터베이스 이름은 다음 문자를 포함할 수 없다. `\, /, ., ' ', *, <, >, :, |, ?, $, \0`
  - 데이터베이스 이름은 대소문자를 구별한다.
  - 데이터베이스 이름은 최대 64바이트다.
- 예약된 데이터베이스 이름들
  - admin: 데이터베이스 인증과 권한 부여 역할을 한다.
  - local: 단일 서버에 대한 데이터를 저장한다.
  - config: 샤딩된 몽고DB 클러스터는 config 데이터베이스를 사용해 각 샤드의 정보를 저장한다.

## 몽고DB 시작

```
docker run --name some-mongo -d mongo:tag
```

- mongod는 인수 없이 실행하면 기본 데이터 디렉토리로 /data/db 를 사용한다. 데이터 디렉토리가 존재하지 않거나 쓰기 권한이 없을 때는 서버가 시작되지 않으므로 주의하자.



## 몽고DB 셸 소개

- 몽고 DB 명령행에서는 몽고DB 인스턴스와 상호작용하는 자바스크립트 셸을 제공한다.
- 셸은 관리 기능, 실행중인 인스턴스를 점검하거나 간단한 기능을 시험하는 데 매우 유용하다.

### 셸 실행

- 셸은 완전한 자바스크립트 해석기이고 임의의 자바스크립트 프로그램을 실행한다.

```
mongo
MongoDB shell version v5.0.9

> x = 200;
200
> x / 5;
40
```

- 표준 자바스크립트 라이브러리의 모든 기능을 활용할 수 있다.

```javascript
> Math.sin(Math.PI / 2)
1
> new Date("2019/1/1");
ISODate("2019-01-01T00:00:00Z")
> "Hello, World!".replace("World", "MongoDB");
Hello, MongoDB!
```

- 자바스크립트 함수를 정의하고 호출할 수도 있다.

```javascript
> function factorial (n) {
... if (n <= 1) return 1;
... return n * factorial(n - 1);
... }
> factorial(5);
120
```

### 몽고DB 클라이언트

- 셸은 시작할 때 test 데이터베이스에 연결하고 데이터베이스 연결을 전역 변수 db에 할당한다.

```
> db
test

// video 데이터베이스 선택
> use video
switched to db video

> db
video

// db변수에서 컬렉션 접근하기
> db.movies
video.movies
```

### 셸 기본 작업

- 셸에서 데이터를 조작하거나 보려면 생성, 읽기, 갱신, 삭제의 네가지 기본적인 작업을 한다.

`생성`

- insertOne 함수는 컬렉션에 도큐먼트를 추가한다.

```json
> movie = {"title" : "Star Wars: Episode IV - A New Hope", "director" : "George Lucas", "year" : 1977}
{
    "title" : "Star Wars: Episode IV - A New Hope",
    "director" : "George Lucas",
    "year" : 1977
}
> db.movies.insertOne(movie)
{
    "acknowledged" : true,
    "insertedId" : ObjectId("62e3e6439bac1dcffb872981")
}
```

- find() 로 찾기

```json
> db.movies.find().pretty()
{
    "_id" : ObjectId("62e3e6439bac1dcffb872981"), // 갑툭튀한 _id 필드는 아래에서 설명
    "title" : "Star Wars: Episode IV - A New Hope",
    "director" : "George Lucas",
    "year" : 1977
}
```

`읽기`

- find와 findOne은 컬렉션을 쿼리하는데 사용한다
- 컬렉션에서 단일 도큐먼트를 읽으려면 findOne을 사용한다.

```json
> db.movies.findOne()
{
    "_id" : ObjectId("62e3e6439bac1dcffb872981"),
    "title" : "Star Wars: Episode IV - A New Hope",
    "director" : "George Lucas",
    "year" : 1977
}
```

- find와 findOne은 쿼리 도큐먼트 형태로 조건 전달도 가능하다.
- 셸은 기본적으로 find와 일치하는 도큐먼트를 20개까지 자동으로 출력한다. (쿼리는 4장에서 ...)

`갱신`

- 게시물을 갱신하려면 updateOne을 사용한다.
- updateOne의 매개변수
  - 첫번째는 수정할 도큐먼트를 찾는 기준이다.
  - 두 번째는 갱신 작업을 설명하는 도큐먼트다.
- 갱신을위해 $set연산자를 사용한다.

```json
> db.movies.updateOne({title : "Star Wars: Episode IV - A New Hope"}, {$set : {reviews: []}})
{ "acknowledged" : true, "matchedCount" : 1, "modifiedCount" : 1 }

> db.movies.findOne()
{
    "_id" : ObjectId("62e3e6439bac1dcffb872981"),
    "title" : "Star Wars: Episode IV - A New Hope",
    "director" : "George Lucas",
    "year" : 1977,
    "reviews" : [ ]
}
```

`삭제`

- deleteOne과 deleteMany는 도큐먼트를 데이터베이스에서 영구적으로 삭제한다.

```json
> db.movies.deleteOne({title : "Star Wars: Episode IV - A New Hope"})
{ "acknowledged" : true, "deletedCount" : 1 }

> db.movies.findOne()
null
```

## 데이터형

### 기본 데이터형

- 몽고DB 도큐먼트는 여섯 가지 데이터형(null, 불리언, 숫자, 문자열, 배열, 객체)만 갖고 있다. 
- 몽고DB는 모든 데이터를 JSON으로 표현하기 때문에 간결하지만 단점도 있다. 날짜형이 없어서 날짜 다루기가 일반적으로 더 성가시고 부동소수점형과 정수형을 표현하는 방법은 없으며 32비트와 64비트도 구별되지 않는다. 함수나 정규표현식과 같은 흔히 쓰는 데이터형을 표현하는 방법도 없다.

`null`

- null 값과 존재하지 않는 필드를 사용한다.

```
{"x": nul}
```

`불리언`

- 참과 거짓 값에 사용한다.

```
{"x": true}
```

`숫자`

- 셸은 64비트 부동소수점 수를 기본으로 사용한다.

```
{"x": 3.14}
{"x": 3}
```

`문자열`

- 어떤 UTF-8 문자열이든 문자열형으로 표현할 수 있다.

```
{"x": "foobar"}
```

`날짜`

- 몽고DB는 1970년 1월 1일부터의 시간을 1/1000초 단위로 나타내는 64비트 정수로 날짜를 저장한다. 표준 시간대는 저장하지 않는다.

```
{"x": new Date()}
```

`정규 표현식`

- 쿼리는 자바스크립트의 정규 표현식 문법을 사용할 수 있다.

```
{"x": /foobar/i}
```

`배열`

- 값의 셋이나 리스트를 배열로 표현할 수 있다.

```
{"x": ["a", "b", "c"]}
```

`내장 도큐먼트`

- 도큐먼트는 부모 도큐먼트의 값으로 내장된 도큐먼트 전체를 포함할 수 있다.

```
{"x": {"foo": "bar"}}
```

`객체 ID`

- 객체 ID는 도큐먼트용 12바이트 ID다.

```
{"x": ObjectId()}
```

- 상대적으로 덜 사용되는 데이터형 목록들

  - 이진데이터

  - 코드

    - 쿼리와 도큐먼트는 임으의 자바스크립트 코드를 포함할 수 있다.

    - ```
      {"x" : function() { /*...*/ }}
      ```

      

### 날짜

- 자바스크립트에서 Date클래스는 몽고DB의 날짜를 표현하는 데 사용한다.
- 새로운 Date 객체를 생성할 때는 반드시 new Date()를 사용해야 한다.
- 셸에서는 날짜가 현지 시간대 설정을 이용해 표시된다. 하지만 데이터베이스의 날짜는 1970년 1월1일부터의 시간을 1/10000초 단위로 저장하며 표준 시간대는 없다. (표준 시간대 정보를 또 다른 키 값으로 저장하면 되긴 한다.)

### 배열

```
{"things": ["pie", 3.14]}
```

- 배열은 서로 다른 데이터형을 값으로 포함할 수 있다.
- 배열값은 일반적인 키/값 쌍을 지원하는 어떤 데이터형 값이든 될 수 있다.
- 몽고DB에서 배열의 장점
  - 몽고DB는 배열의 구조를 이해해서 배열의 내용에 작업을 수행하기 위해 내부에 도달하는 방법을 알고 있다. 
  - 따라서 인덱스를 생성해 쿼리의 속도를 향상시킬 수 있다.
  - 배열 내부에 도달해서 원자적으로 배열의 내용을 수정하게 할 수 있다.

### 내장 도큐먼트

- 내장 도큐먼트를 사용해 데이터를 키/값 쌍의 평면적인 구조보다는 좀 더 자연스러운 방법으로 구성할 수 있다.

```json
{
  "name": "John Doe",
  "address": {
    "street": "123 Park Street",
    "city": "Anytown",
    "state": "NY"
  }
}
```

- 배열과 마찬가지로 몽고DB는 내장 도큐먼트의 구조를 이해하고 인덱스를 구성하고 쿼리하며 갱신하기 위해 내장 도큐먼트 내부에 접근한다.
- 이런 내장 도큐먼트의 특징은 데이터를 다 때려박을 수 있다는건데 정규화가 있는 관계형 데이터베이스보다 유연하다는 장점이 있지만 데이터 중복과 조인에서 얻을 수 있는 간결함을 얻을 수 없다.

### _id와 ObjectId

- 몽고DB에 저장된 모든 도큐먼트는 _id 키를 갖는다.
- _id는 어떤 데이터형도 상관없지만 기본적으로 ObjectId 가 기본이다.
- 모든 도큐먼트는 고유한 _id 값을 가지며 이 값은 컬렉션 내 모든 도큐먼트가 고유하게 식별되게 한다.

`ObjectIds`

- ObjectId를 사용하는 이유
  - 자동증가 아이디가 아니라 ObjectId를 사용하는 주요 이유는 몽고DB의 분산 특성때문이다.
  - 여러 서버에 걸쳐서 자동으로 증가하는 기본 키를 동기화하는 작업은 어렵고 시간이 걸린다.
  - 몽고DB는 분산 데이터베이스로 설계됐기 때문에 샤딩된 환경에서 고유 식별자를 생성하는 것이 매우 중요했다.
- ObjectId
  - ObjectId는 12바이트 스토리지를 사용하여 24자리 16진수 문자열 표현이 가능하다.
  - 바이트당 2자리를 사용한다. 실제로 문자열은 저장된 데이터의 두 배만큼 길다는 점을 알아두자.
- ObjectId의 타임스탬프
  - ObjectId의 첫 4바이트는 1970년 1월 1일부터의 시간을 1/1000초 단위로 저장하는 타임스탬프다.
  - 타임스탬프의 몇가지 유용한 속성
    - 타임스탬프는 그 다음 5바이트와 묶일 때 초 단위의 유일성을 제공한다.
    - 타임스탬프가 맨 처음에 온다는 것은 ObjectId가 대략 입력 순서대로 정렬된다는 의미다. 이는 확실히 보장되진 않지만 ObjectId를 인덱싱하는 장점을 얻을 수 있다.
    - 이 4바이트에는 각 도큐먼트가 생성된 때의 잠재적인 타임스탬프가 존재한다. 대부분의 드라이버는 ObjectId로부터 이런 정보를 추출하는 방법을 제공한다.
  - 타임스탬프를 사용한다고해서 서버의 시간을 동기화해야할 필요는 없다. 타임스탬프의 실제 값은 ObjectId와 상관없다.
- ObjectId의 앞9바이트는 1초동안 여러 장비와 프로세스에 걸쳐 유일성을 보장한다. 마지막 3바이트는 단순히 증분하는 숫자로 1초 내 단일 프로세스의 유일성을 보장한다.
- 고유한 ObjectId는 프로세스당 1초에 1677만개까지 생성된다.

`_id 자동생성`

- 도큐먼트를 입력할 때 "_id" 키를 명시하지 않으면 입력된 도큐먼트에 키가 자동으로 추가된다.
- 이는 몽고DB 서버에서 관리할 수 있지만 일반적으로 클라이언트 쪽 드라이버에서 관리한다.

## 몽고DB 셸 사용

- 셸은 사용자 시스템에서 연결할 수 있는 어떤 몽고DB 인스턴스든 연결할 수 있다.
- 셸을 시작할 때 호스트명, 포트, 데이터베이스를 명시해야 한다.

```
mongo some-host:30000/myDB
```

### 셸 활용 팁

- mongo는 자바스크립트 셸이므로 온라인 자바스크립트 문서를 참고하면 유용하다
- help로 셸에 내장된 도움말을 볼 수 있다.
  - 데이터베이스 수준의 도움말은 db.help()
  - 컬렉션 수준의 도움말은 db.foo.help()
- 함수의 기능을 알고 싶으면 함수명을 괄호 없이 입력하면 된다.

```javascript
> db.movies.findOne
function(query, fields, options, readConcern, collation) {
    var cursor = this.find(query, fields, -1 /* limit */, 0 /* skip*/, 0 /* batchSize */, options);

    if (readConcern) {
        cursor = cursor.readConcern(readConcern);
    }

    if (collation) {
        cursor = cursor.collation(collation);
    }

    if (!cursor.hasNext())
        return null;
    var ret = cursor.next();
    if (cursor.hasNext())
        throw Error("findOne has more than 1 result!");
    if (ret.$err)
        throw _getErrorWithCode(ret, "error " + tojson(ret));
    return ret;
}
```

### 셸에서 스크립트 실행하기

- 다음과 같이 자바스크립트 파일을 셸로 넘길 수도 있다.

```
mongo script1.js script2.js ...
```

- load 함수를 사용할 수도 있다.

```
load("script1.js")
```

- 스크립트는 db 변수에 대한 접근 권한을 가진다. 하지만 use db나 show collections와 같은 셸 보조자는 파일에서 작동하지 않는다.
- 셸 보조자와 대응하는 자바스크립트 함수

| 셸 보조자        | 같은 의미의 자바스크립트 |
| ---------------- | ------------------------ |
| use video        | db.getSisterDB("video")  |
| show dbs         | db.getMongo().getDBs()   |
| show collections | db.getCollectionNames()  |

- 스크립트를 사용해 셸에 변수를 입력할 수도 있다.

```javascript
//defineConnectTo.js

var connectTo = function(port, dbname) {
 if (!port) {
 port = 27017;
 }
 if (!dbname) {
 dbname = "test";
 } 
 db = connect("localhost:"+port+"/"+dbname);
 return db;
};
```

- 이 스크립트를 셸에 로드하면 connectTo가 비로소 정의된다.

```
> typeof connectTo
undefined
> load('defineConnectTo.js')
> typeof connectTo
function
```

- 기본적으로 셸은 셸을 시작한 디렉터리에서 스크립트를 찾는다. 셸을 시작한 디렉터리를 확인하려면 run("pwd") 명령어를 사용한다.
- 스크립트가 현재 디렉터리에 없다면 상대 또는 절대경로를 제공하면 된다.

### .mongorc.js 만들기

- 자주 로드되는 스크립트 (셸이 시작할 때마다 실행되는)를 .mongorc.js 파일에 넣을 수 있다.
- .mongorc.js의 역할
  - 사용하고 싶은 전역변수 설정
  - alias 설정
  - 내장 함수 재정의
  - 위험한 셸 보조자 제거 (dropDatabase, deleteIndexes ... )
- 셸을 시작할때 --norc 옵션을 사용해 .mongorc.js의 로딩을 비활성화할 수도 있다.

### 프롬프트 커스터마이징하기

- 기본 셸 프롬프트는 문자열이나 함수에 prompt 변수를 설정해 재정의한다.

```
// 마지막 작업이 완료된 시각을 얻는 프롬프트
> prompt = function() {
...   return (new Date()) + "> ";
... }
function() {
  return (new Date()) + "> ";
}
Tue Oct 18 2022 13:27:25 GMT+0000 (UTC)> 
Tue Oct 18 2022 13:27:27 GMT+0000 (UTC)> 
Tue Oct 18 2022 13:27:27 GMT+0000 (UTC)> 
Tue Oct 18 2022 13:27:28 GMT+0000 (UTC)> 
Tue Oct 18 2022 13:27:28 GMT+0000 (UTC)> 
```

- 프롬프트 함수는 문자열을 반환하고 예외를 잡는 데 주의를 기울여야 한다.
- 항상 사용자 정의 프롬프트를 사용하려면 .mongorc.js를 사용하자.

### 복잡한 변수 수정하기

- 셸에서 다중행 지원은 다소 제한적이며 이전 행들을 편집할 수 없다. 따라서 코드나 객체 블록이 크면 에디터에서 편집하는게 좋다.

```
EDITOR="/usr/bin/emacs"

// edit {변수명} 형식으로 변수를 편집할 수 있다.
var wap = db.books.findOne({title: "War and Peace"});
edit wap
```



### 불편한 컬렉션명

- 컬렉션명이 예약어가 아니거나 유효하지 않은 자바스크립트 속셩명이 아니라면 db.collectionName으로 컬렉션을 항상 가져올 수 있다.
- 그럼에도 불구하고 특정 collectionName을 사용해야하는 경우 아래와 같이 사용하면 된다.

```
db.getCollection("version")
```

- 자바스크립트의 배열 접근 구문을 사용해서 컬렉션에 접근할 수도 있다
  - 자바스크립트에서 x.y는 x['y']와 동일하다.

```
var collections = ["posts", "comments", "authors"];

for (var i in collections) {
  print(db.blog[collections[i]]);
}
```

- 접근하려는 컬렉션의 이름이 다루기 힘든 경우에는 다음과 같은 방법을 사용한다.

```
var name = "@#$%"
db[name].find()
```

