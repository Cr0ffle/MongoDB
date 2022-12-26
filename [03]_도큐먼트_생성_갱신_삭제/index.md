# 도큐먼트 생성, 갱신, 삭제

# 도큐먼트 삽입

```jsx
db.movies.insertOne({"title":"Stand by Me"})
```

- id를 별도로 제공하지 않은 경우 _id 키가 추가되면서 몽고 DB에 도큐먼트가 저장된다

### insertMany

- 여러 도큐먼트를 컬렉션에 삽입한다
- 각 도큐먼트에 대해 데이터베이스로 왕복하지 않고 도큐먼트를 대량 삽입한다
- `mongoimport`
    - 데이터 피드나 MySQL등에서 원본 데이터를 임포트하는 경우에는 일괄 삽입 대신 쓸 수 있는 mongoimprt같은 명령행 도구가 있다
- 데이터 크기 제한
    - 한번에 일괄 삽입할 수 있는 데이터 크기에 제한이 있다
    - [maxWriteBatchSize](https://www.mongodb.com/docs/manual/reference/limits/#mongodb-limit-Write-Command-Batch-Limit-Size)보다 더 큰 operation을 시도하면 여러 그룹으로 나누어 삽입한다 (기본값은 100,000이다)
    - [https://www.mongodb.com/docs/manual/reference/method/db.collection.insertMany/#execution-of-operations](https://www.mongodb.com/docs/manual/reference/method/db.collection.insertMany/#execution-of-operations)
- 정렬 삽입과 비정렬 삽입
    - 정렬된 삽입의 경우 전달된 배열이 삽입 순서를 정의한다
        - 따라서 배열 중간에 있는 도큐먼트에서 특정 유형의 오류가 발생하는 경우, 삽입 오류가 발생되면 배열에서 해당 지점을 벗어난 도큐먼트는 삽입되지 않는다
    - 정렬되지 않는 삽입은 몽고DB는 일부 삽입이 오류를 발생시키는지 여부에 관계없이 모든 도큐먼트 삽입을 시도한다
    - `insertMany`에 두번째 매개변수로 옵션 도큐먼트를 지정할 수 있다
        - ordered ‘true’ : 정렬 삽입. 순서가 지정되지 않았다면 정렬된 삽입이 기본값이다
        - ordered ‘false’ : 비정렬 삽입. 몽고DB가 성능을 개선하려고 삽입을 재배열할 수 있다
- `insertMany`는 삽입 이외의 작업은 지원하지 않지만, 삽입을 제외한 다른작업도 대량 쓰기가 지원된다
    - [https://www.mongodb.com/docs/manual/core/bulk-write-operations/#bulkwrite---methods](https://www.mongodb.com/docs/manual/core/bulk-write-operations/#bulkwrite---methods)
    - `insertOne`, `updateOne`, `deleteOne`, `replaceOne`
        
        ```jsx
        db.pizzas.bulkWrite( [
              { insertOne: { document: { _id: 3, type: "beef", size: "medium", price: 6 } } },
              { insertOne: { document: { _id: 4, type: "sausage", size: "large", price: 10 } } },
              { updateOne: {
                 filter: { type: "cheese" },
                 update: { $set: { price: 8 } }
              } },
              { deleteOne: { filter: { type: "pepperoni"} } },
              { replaceOne: {
                 filter: { type: "vegan" },
                 replacement: { type: "tofu", size: "small", price: 4 }
              } }
           ] )
        ```
        

### 삽입 유효성 검사

- 몽고DB는 최소한의 검사만 수행한다
    - 도큐먼트의 기본구조를 검사하여 _id 필드가 존재하지 않으면 새로 추가한다
    - 모든 도큐먼트는 16메가바이트보다 작아야 하기 때문에 크기를 검사한다
    - 도큐먼트의 Binary JSON 크기를 보려면 object.bsonsize(doc)을 실행한다

## 도튜먼트 삭제

- CRUD API는 `deleteOne`과 `deleteMany`를 제공한다
- 두 메소드 모두 필터 도큐먼트를 첫 번째 매개변수로 사용한다

```jsx
db.movies.deleteOne({"_id":4})
```

- `deleteOne` : 필터와 일치하는 첫번째 도큐먼트를 삭제한다
- `deleteMany` : 필터와 일치하는 모든 도큐먼트를 삭제한다
- `deleteOne`는 어떤 도큐먼트가 먼저 발견되는지는 도큐먼트가 삽입된 순서, 도큐먼트에 어떤 갱신이 이뤄졌는지 어떤 인덱스를 지정하는지 등 몇가지 요인에 따라 달라진다
- capped collection이나 time series collection에 삭제를 사용할 경우 WriteError가 발생한다
    - capped collection : [https://www.mongodb.com/docs/manual/reference/glossary/#std-term-capped-collection](https://www.mongodb.com/docs/manual/reference/glossary/#std-term-capped-collection)
    - time series collection : [https://www.mongodb.com/docs/manual/reference/glossary/#std-term-time-series-collection](https://www.mongodb.com/docs/manual/reference/glossary/#std-term-time-series-collection) (5.0에서 새로 생김)

### drop

- `deleteMany`도 일반적으로 도큐먼트를 제거하는 작업이 꽤 빠르다
- 그러나 만약 전체 컬렉션을 삭제하려면 `drop`을 사용하는 편이 빠르다
- drop 후에 빈 컬렉션에 인덱스를 재생성한다
- v4.4까지 샤드 클러스터의 경우, '`drop`'을 사용한 다음에 같은 이름으로 새 컬렉션을 만드는 경우
    - drop 후에 flushRouterConfig를 사용하여 모든 몽고에서 캐시된 라우팅 테이블을 플러시한다
    - 캐시를 플러시하지 않으려면 ‘remove’를 사용한다

### 삽입과 삭제

- 몽고DB 3.0 이전 버전에서는 주로 insert,remove를 사용하였다
- 몽고 3.2부터는 `insertOne`, `insertMany`, `deleteOne`, `deleteMany`가 등장하였지만 호환성을 위해 `insert`,`delete`도 그대로 지원된다
    - `insert`,`remove`는 4.4까지도 지원이 되지만, shell에서 사용할 수 있을 뿐 mongoDB 드라이버에서는 지원되지 않는다
- 3.2부터는 `insertOne`, `insertMany`, `deleteOne`, `deleteMany`를 사용하자

## 도큐먼트 갱신

- `updateOne`, `updateMany`, `replaceOne`
- `updateOne`, `updateMany`은 필터 도큐먼트를 첫 번째 매개변수로, 변경사항을 두번째 매개변수로 사용한다
- 원자적 갱신
    - 갱신은 원자적으로 이뤄진다
    - 갱신 요청 두 개가 동시에 발생하면 서버에 먼저 도착한 요청이 적용된 후에 다음 요청이 적용된다
    - 여러 개의 갱신 요청이 빠르게 발생하더라도 결국 마지막 요청이 최후의 승리자가 된다

### 도큐먼트 치환

- `replaceOne`은 도큐먼트를 새로운 것으로 완전히 치환한다
- 도큐먼트 치환 시에, 고유한 도큐먼트를 갱신 대상으로 지정하는 것이 좋다
- _id값은 컬렉션 기본 인덱스의 기초를 형성하므로 필터에 _id를 사용해도 좋다

### 갱신 연산자

- 일반적으로 도큐먼트의 일부만 갱신하는 경우가 많다
- 부분 갱신에는 원자적 갱신 연산자를 사용한다
    - 예를 들어, 만약 페이지에 방문할 때마다 카운트를 증가한다고 해보자
        
        ```jsx
        {
        	"_id" : ObjectId("6315cc29f4a07448f070bbb1"),
          "url" : "www.example.com",
          "pageviews" : 52
        }
        ```
        
    - 누군가가 페이지를 방문할 때마다 url로 페이지를 찾고 pageviews 키의 값을 증가시키려면 $inc 제한자를 사용한다

**$set 제한자 사용하기**

- 스키마를 갱신하거나 사용자 정의 키를 추가할 때 사용한다
(필드가 존재하지 않으면 새 필드가 생성된다)
- 키의 데이터형도 변경할 수 있다
    
    ```jsx
    db.user.updateOne({"name", "joe"}, {"$set":{"facorite book" : "Green Eggs And Ham"}});
    db.user.updateOne({"name", "joe"}, {"$set":{"facorite book" : ["Green Eggs And Ham", "Cat's Cradle"]}});
    ```
    
- $unset은 키와 값을 모두 제거한다
    
    ```jsx
    db.users.updateOne({"name", "joe"}, {"$unset":{"facorite book":1}})
    ```
    
- 키를 추가, 변경, 삭제할때는 항상 $ 제한자를 사용해야 한다
    - 이 경우는 오류가 발생한다
    
    ```jsx
    db.blog.posts.updateOne({"author.name":"joe"}, {"author.name":"joe schemoe"})
    ```
    

**증가와 감소**

- $inc 연산자는 이미 존재하는 키의 값을 변경하거나 새 키를 생성한다
- 분석, 분위기, 투표 같은 자주 변하는 수치 값을 갱신하는데 유용하다
- $set과 비슷하지만 이는 숫자를 증감하기 위해 설계되었기에 int, long, double, decimal 타입에만 사용할 수 있다
- null, 불리언, 숫자로 나타낸 문자열라 할지라도 사용할 수 없다

**배열 연산자**

- 요소 추가하기
    - $push는 배열이 이미 존재하면 배열 끝에 요소를 추가하고 존재하지 않으면 새로운 배열을 생성한다
    
    ```jsx
    db.blog.posts.updateOne(
       {"title":"A blog post"},
       {"$push", {"comments":{"name":"joe", "email":"joe@example.com", "comment":"nice posts."}}}
    )
    ```
    
    - $push에 $each를 사용하면 작업 한번으로 여러 개를 추가할 수 있다
    
    ```jsx
    db.blog.posts.updateOne(
       {"title":"A blog post"},
       {"$push", {"comments":{"$each":[{"name":"joe", "email":"joe@example.com", "comment":"nice posts."}, {"name":"joe", "email":"wanda@example.com", "comment":"angry."}]}}}
    )
    ```
    
- 배열을 집합으로 사용하기
    - $ne
        - 특정 값이 배열에 존재하지 않을 때만 해당 값을 추가한다
        - 쿼리 도큐먼트에 $ne를 사용한다
            
            ```jsx
            db.papers.updateOne(
              {"authors cited":{"$ne":"Richie"}},
              {$push: {"authors cited":"Richie"}}
            )
            ```
            
    - $addToSet
        - 중복된 데이터가 없을 경우에만 추가한다
        
        ```jsx
        db.users.updateOne(
          {"id": ObjectId("6315cc29f4a07448f070bbb1")},
          {"$addToSet": {"emais":"joe@gmail.com"}}
        );
        {"acknowledged":true, "matchedCount":1, "modifiedCount":1}
        
        db.users.updateOne(
          {"id": ObjectId("6315cc29f4a07448f070bbb1")},
          {"$addToSet": {"emais":"joe@gmail.com"}}
        )
        {"acknowledged":true, "matchedCount":1, "modifiedCount":0}
        ```
        
        - 여러 개를 추가하려면 $each를 결합해 사용한다
- 요소 제거하기
    - $pop
        - 배열의 처음 또는 마지막부터 요소를 제거한다
        
        ```jsx
        {"$pop":{"key":1}} -> 마지막부터 하나씩 제거
        {"$pop":{"key":-1}} -> 처음부터 하나씩 제거
        ```
        
    - $pull
        - 주어진 조건에 맞는 배열 요소를 모두 제거한다
        
        ```jsx
        db.lists.updateOne({}, {"$pull":{"todo":"laundry"}})
        ```
        
- 배열의 위치 기반 변경
    - 값이 여러개인 배열에서 일부 변경을 조작하는 것은 꽤 어렵다
    - 배열의 위치를 이용하거나 위치 연산자를 사용한다
    - 배열의 위치를 이용
        
        ```jsx
        db.blog.updateOne({"post":post_id}, {"$inc":{"comments.**0**.votes" : 1}})
        // 0번째 요소의 값을 1 증가시킨다
        ```
        
    - 배열의 위치 연산자를 사용
        
        ```jsx
        db.blog.updateOne({"comments.author":"John"}, {"$set":{"comments.**$**.author" : "Jim"}})
        // john이라는 author가 jim으로 변경된다
        ```
        
- 배열 필터를 이용한 갱신
    - arrayFilters를 사용하여 특정 조건에 맞는 배열 요소를 갱신할 수 있다
    
    ```jsx
    db.blog.updateOne(
    	{"post":post_id},
    	{"$set":{"comments.$[elem].hidden":true}},
    	{arrayFilters:[{"elem.votes":{$lte:-5}}]}
    )
    // -5보다 작은 값을 지닌 elem에 hidden 을 true로 설정한다
    ```
    

### 갱신 입력

- 갱신 조건에 맞는 도큐먼트가 존재하지 않으면 쿼리 도큐먼트와 갱신 도큐먼트를 합쳐 새로운 도큐먼트를 생성한다
    
    ```jsx
    db.analyrics.updateOne(
    	{"url":"/blog"},
    	{"$inc":{"pageviews":1}},
    	**{"upsert":true}**
    )
    ```
    
- 조건에 맞는 도큐먼트가 발견되면 일반적인 갱신을 수행한다
- 즉, 같은 코드로 도큐먼트를 생성하거나 갱신한다
    
    ```jsx
    db.users.updateOne(
    	{"req":25},
    	{"$inc":{"req":3}},
    	{"upsert":true}
    )
    // req가 28인 문서가 생성된다
    ```
    
- 만약, 문서가 생성될 때에만 지정해야 할 필드가 있다면 $setOnInsert를 사용한다
    
    ```jsx
    > db.users.updateOne(
    	{},
    	{"$setOnInsert":{"createdAt":new Date()}},
    	{"upsert":true}
    )
    // createdAt이 현재 날짜로 문서가 생성된다
    ```
    

**셸 보조자 : save**

- `save`는 도큐먼트가 존재하지 않으면 도큐먼트를 삽입하고, 존재하면 도큐먼트를 갱신하는 셸 함수이다
- 도큐먼트가 _id 를 포함하면 save는 갱신 입력을 실행하고 포함하지 않으면 삽입을 실행한다
- save는 개발자가 셸에서 도큐먼트를 빠르게 수정하게 해준다

### 다중 도큐먼트 갱신

- `updateOne`은 필터 조건에 맞는 첫 번째 도큐먼트만 갱신한다
- 만약 조건에 맞는 도큐먼트를 모두 수정하려면 `updateMany`를 사용하자

### 도큐먼트 반환

- MongoDB에서는 여러 명령을 하나의 트랜잭션으로 묶어서 사용할 수 없다
    - (몽고디비는 단일 문서 단위의 트랜잭션만 지원됨)
- 이때문에 변경 직전이나 직후의 문서 데이터를 확인하기 쉽지 않다
- 이러한 기능을 제공하기 위해서 `FindAndModify`라는 명령을 제공한다
- 해당 명령은 검색 조건에 일치하는 문서를 검색하고, 그 문서를 변경하거나 삭제하는 후속 오퍼레이션을 설정할 수 있다
- `findAndModify`는 삭제, 대체, 갱신 이 세 가지 작업의 기능을 결합되어있는 메소드이다
- 따라서 사용자 오류가 발생하기 쉽기에 몽고DB 3.2에서는 `findAndModify`보다 더 쉬운 이름을 가진 `findOneAndDelete`, `findOneAndReplace`, `findOneAndUpdate`가 셸에 도입되었다
- 또 몽고DB 4.2에서는 갱신을 위한 집계 파이프라인을 수용하도록 `findOneAndUpdate`를 확장하였다
- `updateOne`과의 차이점 : 사용자가 수정된 도큐먼트의 값을 원자적으로 얻을 수 있다
    
    ```jsx
    var cursor = db.processes.find({"stauts":"READY"});
    ps = cursor.sort({"priority":-1}).limit(1).next();
    db.processes.updateOne({"_id":ps._id}, {"$set":{"status":"RUNNING"}});
    do_something(ps);
    ps.processes.updateOne({"_id":ps._id}, {"$set":{"status":"DONE"}});
    ```
    
    - 만약 여기서 스레드 두개가 실행중이고 스레드 A가 먼저 도큐먼트를 얻고 RUNNING으로 갱신하기 전에 스레드 B가 같은 도큐먼트를 받으면 두 스레드가 같은 프로세스를 실행하게 된다
    - 이 알고리즘은 경쟁 상태를 만들기 때문에 좋지 않다
    - 따라서 이런 상황에서는 `findOneAndUpdate`가 적합한데, 한 번의 연산으로 항목을 반환하고 갱신한다
        
        ```jsx
        db.processes.findOneAndUpdate(
        	{"status":"READY"},
        	{"$set":{"status":"RUNNING"}},
        	{"sort":{"priority":-1}}
        )
        ```
        
    - `findOneAndUpdate`는 기본적으로 도큐먼트의 상태를 수정하기 전에 반환하기 때문에, 반환된 도큐먼트에서 상태가 여전히 READY다
    - 옵션 도큐먼트의 `returnNewDocument` 필드를 true로 설정하면 갱신된 도큐먼트를 반환한다
        
        ```jsx
        db.processes.findOneAndUpdate(
        	{"status":"READY"},
        	{"$set":{"status":"RUNNING"}},
        	{"sort":{"priority":-1}, "returnNewDocument":true}
        )
        ```
        
    - `findOneAndReplace`는 동일한 매개변수를 사용하고,
    `findOneAndDelete`도 유사하지만 갱신 도큐먼트 매개변수는 사용하지 않으며 삭제된 도큐먼트를 반환한다