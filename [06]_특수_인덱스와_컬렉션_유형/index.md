# #6 특수 인덱스와 컬렉션 유형

- 이 장에서 설명하는 것들
  - 큐 같은 데이터를 위한 제한 컬렉션
  - 캐시를 위한 TTL 인덱스
  - 단순 문자열 검색을 위한 전문 인덱스
  - 2d 구현 및 구현 기하학을 위한 공간 정보 인덱스
  - 대용량 파일 저장을 위한 GridFS



## 공간 정보 인덱스

- 몽고DB는 2dsphere와 2d라는 공간 정보 인덱스를 가진다.
- 2dsphere
  - WGS84 좌표계를 기반으로 지표면을 모델링하는 구면 기하학으로 동작
    - WGS84 좌표계는 지표면을 주상절벽으로 모델링하며, 이는 극지방에 약간의 평탄화가 존재함을 의미한다.
  - 2dsphere 인덱스를 사용하면 지구의 형태를 고려하므로 2d 인덱스를 사용할 때보다 더 정확한 거리를 계산할 수 있다. 
    - ex: 두 도시간의 거리
  - 만약 2차원 평면의 점 사이의 거리라면 2d 인덱스를 사용하면 된다.
  - GeoJSON 형식으로 점, 선, 다각형의 기하 구조를 지정할 수 있다. 
    - http://www.geojson.org

```
// 점은 경도 좌표와 위도 좌표를 요소로 갖는 배열로 표현된다.
{
  "name" : "New York City",
  "loc" : {
    "type" : "Point",
    "coordinates" : [50, 2]
  }
}


// 선은 점의 배열로 표현된다.
{
  "name" : "LineString",
  "loc" : {
    "type" : "Point",
    "coordinates" : [[0,1], [0, 2], [0, 3]]
  }
}

// 다각형은 선과 같은 방식으로 표현된다.
{
  "name" : "New England",
  "loc" : {
    "type" : "Polygon",
    "coordinates" : [[0,1], [0, 2], [0, 3]]
  }
}

// loc라는 필드명은 변경 가능하지만 type과 corordinates는 GeoJSON에서 지정되므로 변경이 불가하다.
```

- createIndex와 함께 2dsphere를 사용해 공간 정보 인덱스를 만들 수 있다.

```
db.openStreetMap.createIndex({"loc" : "2dsphere"})
```



### 공간 정보 쿼리 유형

- 공간정보 쿼리는 교차, 포함, 근접이라는 세가지 유형이 있다.
- 찾고싶은 항목은 {"$geometry" : geoJson} 와 같은 형태로 지정한다.
  - 예를 들어 "$geoIntersects" 연산자를 사용해 쿼리 위치와 교차하는 도큐먼트를 찾을 수 있다.

```
var eastVillage = { "type" : "Polygon", "coordinates" : [ [ [ -73.9732566, 40.7187272 ], [ -73.9724573, 40.7217745 ],
[ -73.9717144, 40.7250025 ], [ -73.9714435, 40.7266002 ], [ -73.975735, 40.7284702 ], [ -73.9803565, 40.7304255 ],
[ -73.9825505, 40.7313605 ], [ -73.9887732, 40.7339641 ], [ -73.9907554, 40.7348137 ], [ -73.9914581, 40.7317345 ],
[ -73.9919248, 40.7311674 ], [ -73.9904979, 40.7305556 ], [ -73.9907017, 40.7298849 ], [ -73.9908171, 40.7297751 ],
[ -73.9911416, 40.7286592 ], [ -73.9911943, 40.728492 ], [ -73.9914313, 40.7277405 ], [ -73.9914635, 40.7275759 ], 
[ -73.9916003, 40.7271124 ], [ -73.9915386, 40.727088 ], [ -73.991788, 40.7263908 ], [ -73.9920616, 40.7256489 ], 
[ -73.9923298, 40.7248907 ], [ -73.9925954, 40.7241427 ], [ -73.9863029, 40.7222237 ], [ -73.9787659, 40.719947 ],
[ -73.9772317, 40.7193229 ], [ -73.9750886, 40.7188838 ], [ -73.9732566, 40.7187272 ] ] ]}

// 뉴욕의 이스트빌리지 내에 있는 한 점을 갖는 점, 선, 다각형이 포함된 도큐먼트를 모두 찾는다.
db.openStreetMap.find({"loc" : {"$geoIntersects" : {"$geometry" : eastVillage}}})

// $geoWithin을 사용하면 지역에 완전히 포함된 항목을 쿼리할 수 있다.
db.openStreetMap.find({"loc" : {"$geoWithin" : {"$geometry" : eastVillage}}})

// $near를 사용해 주변 위치에 쿼리할 수 있다.
db.openStreetMap.find({"loc" : {"$near" : {"$geometry" : eastVillage}}})

```

- $near는 정렬을 포함하는 유일한 공간 정보 연산자다. 항상 가까운 곳부터 가장 먼 곳 순으로 반환된다.



### 공간 정보 인덱스 사용

- 몽고DB의 공간 정보 인덱스를 사용하면 특정 지역과 관련된 모양과 점이 포함된 컬렉션에서 공간 쿼리를 효율적으로 실행할 수 있다.
- 사용자가 뉴욕에서 레스토랑을 찾도록 돕는 모바일 애플리케이션의 요구사항
  - 사용자가 현재 위치한 지역을 찾는다.
  - 해당 지역 내 레스토랑 수를 보여준다.
  - 지정된 거리 내에 있는 레스토랑을 찾는다.

`쿼리에서의 2d vs. 구면 기하학`

- 공간 정보 쿼리는 쿼리와 사용 중인 인덱스 유형에 따라 구면 또는 2d 구조를 사용한다.
- 각 공간 정보 연산자가 사용하는 기하 구조 유형

| 쿼리 유형                                    | 기하 구조 유형 |
| -------------------------------------------- | -------------- |
| $near (GeoJSON point, 2dsphere 인덱스)       | 구면           |
| $near (레거시 좌표, 2d 인덱스)               | 평면           |
| $geoNear (GeoJSON point, 2dsphere 인덱스)    | 구면           |
| $geoNear (레거시 좌표, 2d 인덱스)            | 평면           |
| $nearSphere (GeoJSON point, 2dsphere 인덱스) | 구면           |
| $nearSphere (레거시 좌표, 2d 인덱스)         | 구면           |
| $geoWithin : { $geometry : ... }             | 구면           |
| $geoWithin : { $box : ... }                  | 평면           |
| $geoWithin : { $polygon : ... }              | 평면           |
| $geoWithin : { $center : ... }               | 평면           |
| $geoWithin : { $centerSphere : ... }         | 구면           |
| $geoIntersects                               | 구면           |

- 2d 인덱스는 구에서 평면 기하학과 거리 계산을 모두 지원한다는 점에서 유의하자. 하지만 구면 기하학을 사용하는 쿼리는 2dsphere 인덱스를 사용할 때 성능과 정확성이 향상된다.
- $geoNear 쿼리와 특수명령어 geoNear 
  - $geoNear 연산자는 집계 연산자라는 점에서 유의하자.
  - $near 쿼리 외에도 $geoNear나 특수 명령어인 geoNear를 사용해 주변 위치를 쿼리할 수 있다.
  - geoNear 명령어와 $geoNear 집계 연산자를 사용하려면 컬렉션에 2dsphere 인덱스와 2d 인덱스가 최대 1개만 있어야 한다. 반면에 $near나 $geoWithin과 같은 공간 정보 쿼리 연산자는 컬렉션이 여러 개의 공간 정보 인덱스를 갖도록 허용한다.
  - geoNear 명령과 $geoNear 구문은 둘 다 위치 필드를 포함하지 않으므로 공간 정보 인덱스 제한이 존재한다. 따라서 여러 2d인덱스나 2dsphere 인덱스 간의 선택이 모호해진다. 공간 정보 쿼리 연산자에는 이러한 제한이 없으며, 위치 필드를 사용해 모호성을 없앤다.
- $near 쿼리 연산자는 몽고DB의 샤딩을 사용해 배포된 컬렉션에서는 작동하지 않는다.



`왜곡`

- 구면 기하는 지도에 시각화하면 왜곡이 있는데, 지구와 같은 3차원 구를 평면에 투사하는 특성 때문이다.

![6-1](https://github.com/sup2is/dev-note/blob/master/db/images/mongodb/6-1.jpeg?raw=true)

`레스토랑 검색`

- 뉴욕에 위치한 지역 데이터 셋을 이용해서 레스토랑을 검색하는 예제

```
curl -L -o neighborhoods.json http://oreil.ly/rpGna
curl -L -o restaurants.json https://oreil.ly/JXYd-

docker cp ./neighborhoods.json mongodb:/
docker cp ./restaurants.json mongodb:/

mongoimport ./neighborhoods.json -c neighborhoods
mongoimport ./restaurants.json -c restaurants

// 2dsphere 인덱스 생성하기
db.neighborhoods.createIndex({location: "2dsphere"})
db.restaurants.createIndex({location: "2dsphere"})
```



`데이터 탐색`

```
db.neighborhoods.find({name: "Clinton"})

...

// 스키마 이해하기
> db.restaurants.find({name: "Little Pie Company"}).pretty()
{
	"_id" : ObjectId("55cba2476c522cafdb053dea"),
	"location" : {
		"coordinates" : [
			-73.99331699999999,
			40.7594404
		],
		"type" : "Point"
	},
	"name" : "Little Pie Company"
}

```



`현재 지역 찾기`

- 사용자의 휴대 기기가 정확한 위치 정보를 제공한다고 가정할 때, $geoIntersects를 사용해 사용자가 현재 위치한 지역을 쉽게 찾을 수 있다.

```
db.neighborgoods.findOne({geometry:{$geoIntersects:{$geometry:{type:"Point", coordinates:[-73.93414657, 40.82302903]}}}})
```





`지역 내 모든 레스토랑 찾기`

- 특정 지역 내 레스토랑을 모두 찾는 쿼리도 수행할 수 있다.

```
var neighborhood = db.neighborhoods.findOne({ geometry: 
    { $geoIntersects: { $geometry: { type: "Point", coordinates: [-73.93414657,40.82302903] } } } });

db.restaurants.find({ location: { $geoWithin: { $geometry: neighborhood.geometry } } }, {name: 1, _id: 0});
{ "name" : "Perfect Taste" }
{ "name" : "Event Productions Catering & Food Services" }
{ "name" : "Sylvia'S Restaurant" }
{ "name" : "Corner Social" }
{ "name" : "Cove Lounge" }
{ "name" : "Manna'S Restaurant" }
{ "name" : "Harlem Bar-B-Q" }
{ "name" : "Hong Cheong" }
{ "name" : "Lighthouse Fishmarket" }
{ "name" : "Mahalaxmi Food Inc" }
{ "name" : "Rose Seeds" }
{ "name" : "J. Restaurant" }
{ "name" : "Harlem Coral Llc" }
{ "name" : "Baraka Buffet" }
{ "name" : "Make My Cake" }
{ "name" : "Hyacinth Haven Harlem" }
{ "name" : "Mcdonald'S" }
{ "name" : "Ihop" }
{ "name" : "Island Spice And Southern Cuisine" }
{ "name" : "To Your Health & Happiness" }
Type "it" for more

// 127개 리턴 ..

```



`범위 내에서 레스토랑 찾기`

- 특정 지점으로부터 지정된 거리 내에 있는 레스토랑을 찾을 수 있다.
  - $centerSphere와 함께 $geoWithin을 사용하면 정렬되지 않은 순서로 결과를 반환한다.
  - $maxDistance와 함께 $nearSphere를 사용하면 거리 순으로 정렬된 결과를 반환한다.
  - 원형 지역 내 레스토랑을 찾으려면 $centerSphere와 함께 $geoWithin을 사용한다.
    - $centerShpere는  중심과 반경을 라디안으로 지정해 원형 영역을 나타내는 몽고DB 전용 구문이다.
    - $geoWithin은 도큐먼트를 특정 순서로 반환하지 않으므로 거리가 가장 먼 도큐먼트를 먼저 반환할 수 있다.

```
// $centerShpere와 $geoWithin을 사용해서 쿼리
> db.restaurants.find({ location: { $geoWithin: { $centerSphere: [ [-73.93414657,40.82302903], 5/3963.2 ] } } })
{ "_id" : ObjectId("55cba2476c522cafdb0552d5"), "location" : { "coordinates" : [ -73.9911006, 40.76503719999999 ], "type" : "Point" }, "name" : "Cakes 'N Shapes" }
{ "_id" : ObjectId("55cba2476c522cafdb05590c"), "location" : { "coordinates" : [ -73.9914056, 40.765347 ], "type" : "Point" }, "name" : "Il Melograno" }
{ "_id" : ObjectId("55cba2476c522cafdb057cf2"), "location" : { "coordinates" : [ -73.990922, 40.7653572 ], "type" : "Point" }, "name" : "Kare Thai" }
{ "_id" : ObjectId("55cba2486c522cafdb059980"), "location" : { "coordinates" : [ -73.9909028, 40.7654228 ], "type" : "Point" }, "name" : "City Slice" }
{ "_id" : ObjectId("55cba2476c522cafdb053e29"), "location" : { "coordinates" : [ -73.99076149999999, 40.7653444 ], "type" : "Point" }, "name" : "Azuri Cafe" }
{ "_id" : ObjectId("55cba2476c522cafdb058aae"), "location" : { "coordinates" : [ -73.9910182, 40.76501469999999 ], "type" : "Point" }, "name" : "Totto Ramen" }
{ "_id" : ObjectId("55cba2476c522cafdb058603"), "location" : { "coordinates" : [ -73.9906967, 40.7657361 ], "type" : "Point" }, "name" : "Crispin'S Hell'S Kitchen" }
{ "_id" : ObjectId("55cba2476c522cafdb057470"), "location" : { "coordinates" : [ -73.99078560000001, 40.765615 ], "type" : "Point" }, "name" : "Happy Joy" }
{ "_id" : ObjectId("55cba2476c522cafdb056129"), "location" : { "coordinates" : [ -73.9910501, 40.7659938 ], "type" : "Point" }, "name" : "Ardesia" }
{ "_id" : ObjectId("55cba2476c522cafdb05845b"), "location" : { "coordinates" : [ -73.990179, 40.765078 ], "type" : "Point" }, "name" : "Zoralie Restaurant Inc." }
{ "_id" : ObjectId("55cba2476c522cafdb054746"), "location" : { "coordinates" : [ -73.9889215, 40.7645101 ], "type" : "Point" }, "name" : "Posh" }
{ "_id" : ObjectId("55cba2476c522cafdb0587d2"), "location" : { "coordinates" : [ -73.988923, 40.764083 ], "type" : "Point" }, "name" : "Atlas Social Club" }
{ "_id" : ObjectId("55cba2476c522cafdb053f68"), "location" : { "coordinates" : [ -73.9890549, 40.763902 ], "type" : "Point" }, "name" : "Uncle Nicks" }
{ "_id" : ObjectId("55cba2476c522cafdb054c5a"), "location" : { "coordinates" : [ -73.98357779999999, 40.7613731 ], "type" : "Point" }, "name" : "Applebee'S Neighborhood Grill & Bar" }
{ "_id" : ObjectId("55cba2476c522cafdb054077"), "location" : { "coordinates" : [ -73.983583, 40.76174839999999 ], "type" : "Point" }, "name" : "Winter Garden Theater" }
{ "_id" : ObjectId("55cba2476c522cafdb055bc5"), "location" : { "coordinates" : [ -73.983583, 40.76174839999999 ], "type" : "Point" }, "name" : "Nanking" }
{ "_id" : ObjectId("55cba2476c522cafdb053fa2"), "location" : { "coordinates" : [ -73.98333319999999, 40.7618628 ], "type" : "Point" }, "name" : "Ellen'S Stardust Diner" }
{ "_id" : ObjectId("55cba2476c522cafdb054101"), "location" : { "coordinates" : [ -73.9835976, 40.7621614 ], "type" : "Point" }, "name" : "Starbucks Coffee" }
{ "_id" : ObjectId("55cba2476c522cafdb057ee5"), "location" : { "coordinates" : [ -73.9837508, 40.7622594 ], "type" : "Point" }, "name" : "Mcdonald'S" }
{ "_id" : ObjectId("55cba2476c522cafdb0546c5"), "location" : { "coordinates" : [ -73.9845362, 40.7620434 ], "type" : "Point" }, "name" : "Circle In The Square Theatre" }

```

- 애플리케이션은 공간 정보 인덱스 없이도 $centerSphere를 사용할 수 있지만 공간정보 인덱스를 사용하면 훨씬 빨리 쿼리를 실행할 수 있다.

```
// 사용자로부터 5마일 이내에 있는 모든 레스토랑을 가장 가까운 곳에서 가장 먼 곳 순으로 반환하는 쿼리

// $nearSphere와 $maxDistance를 사용해서 쿼리
> var METERS_PER_MILE = 1609.34;
> db.restaurants.find({ location: { $nearSphere: { $geometry: { type: "Point",coordinates: [-73.93414657,40.82302903] }, $maxDistance: 5*METERS_PER_MILE } } });

{ "_id" : ObjectId("55cba2476c522cafdb058c83"), "location" : { "coordinates" : [ -73.9316894, 40.8231974 ], "type" : "Point" }, "name" : "Gotham Stadium Tennis Center Cafe" }
{ "_id" : ObjectId("55cba2476c522cafdb05864b"), "location" : { "coordinates" : [ -73.9378967, 40.823448 ], "type" : "Point" }, "name" : "Tia Melli'S Latin Kitchen" }
{ "_id" : ObjectId("55cba2476c522cafdb058c63"), "location" : { "coordinates" : [ -73.9303724, 40.8234978 ], "type" : "Point" }, "name" : "Chuck E. Cheese'S" }
{ "_id" : ObjectId("55cba2476c522cafdb0550aa"), "location" : { "coordinates" : [ -73.93795159999999, 40.823376 ], "type" : "Point" }, "name" : "Domino'S Pizza" }
{ "_id" : ObjectId("55cba2476c522cafdb0548e0"), "location" : { "coordinates" : [ -73.9381738, 40.8224212 ], "type" : "Point" }, "name" : "Red Star Chinese Restaurant" }
{ "_id" : ObjectId("55cba2476c522cafdb056b6a"), "location" : { "coordinates" : [ -73.93011659999999, 40.8219403 ], "type" : "Point" }, "name" : "Applebee'S Neighborhood Grill & Bar" }
{ "_id" : ObjectId("55cba2476c522cafdb0578b3"), "location" : { "coordinates" : [ -73.93011659999999, 40.8219403 ], "type" : "Point" }, "name" : "Marisco Centro Seafood Restaurant  & Bar" }
{ "_id" : ObjectId("55cba2476c522cafdb058dfc"), "location" : { "coordinates" : [ -73.9370572, 40.8206095 ], "type" : "Point" }, "name" : "108 Fast Food Corp" }
{ "_id" : ObjectId("55cba2476c522cafdb0574cd"), "location" : { "coordinates" : [ -73.9365102, 40.8202205 ], "type" : "Point" }, "name" : "Kentucky Fried Chicken" }
{ "_id" : ObjectId("55cba2476c522cafdb057d52"), "location" : { "coordinates" : [ -73.9385009, 40.8222455 ], "type" : "Point" }, "name" : "United Fried Chicken" }
{ "_id" : ObjectId("55cba2476c522cafdb054e83"), "location" : { "coordinates" : [ -73.9373291, 40.8206458 ], "type" : "Point" }, "name" : "Dunkin Donuts" }
{ "_id" : ObjectId("55cba2476c522cafdb05615f"), "location" : { "coordinates" : [ -73.9373291, 40.8206458 ], "type" : "Point" }, "name" : "King'S Pizza" }
{ "_id" : ObjectId("55cba2476c522cafdb05476a"), "location" : { "coordinates" : [ -73.9365637, 40.8201488 ], "type" : "Point" }, "name" : "Papa John'S" }
{ "_id" : ObjectId("55cba2486c522cafdb059a11"), "location" : { "coordinates" : [ -73.9365637, 40.8201488 ], "type" : "Point" }, "name" : "Jimbo'S Hamburgers" }
{ "_id" : ObjectId("55cba2476c522cafdb0580a7"), "location" : { "coordinates" : [ -73.938599, 40.82211110000001 ], "type" : "Point" }, "name" : "Home Garden Chinese Restaurant" }
{ "_id" : ObjectId("55cba2476c522cafdb05814c"), "location" : { "coordinates" : [ -73.9367511, 40.8198978 ], "type" : "Point" }, "name" : "Sweet Mama'S Soul Food" }
{ "_id" : ObjectId("55cba2476c522cafdb056b96"), "location" : { "coordinates" : [ -73.9308109, 40.82594580000001 ], "type" : "Point" }, "name" : "Dunkin Donuts (Inside Gulf Gas Station On North Side Of Maj. Deegan Exwy- After Exit 13 - 233 St.)" }
{ "_id" : ObjectId("55cba2476c522cafdb056ffd"), "location" : { "coordinates" : [ -73.939159, 40.8216897 ], "type" : "Point" }, "name" : "Reggae Sun Delights Natural Juice Bar" }
{ "_id" : ObjectId("55cba2476c522cafdb056b0c"), "location" : { "coordinates" : [ -73.939145, 40.8213757 ], "type" : "Point" }, "name" : "Ho Lee Chinese Restaurant" }
{ "_id" : ObjectId("55cba2486c522cafdb059617"), "location" : { "coordinates" : [ -73.9396354, 40.8220958 ], "type" : "Point" }, "name" : "Ivory D O S  Inc" }
Type "it" for more


```



### 복합 공간 정보 인덱스

- 공간 정보 인덱스도 다른 인덱스와 마찬가지로 다른 필드와 묶어서 더 복잡한 쿼리를 최적화할 수 있다.

```
// tags, location 복합 인덱스 생성
db.openStreetMap.createIndex({"tags" : 1, "location" : "2dsphere"})

db.openStreetMap.find({"loc" : {"$geoWithin" : {"$geometry" : hellsKitchen.geometry}}, "tags" : "pizza"})
```

- 복합 공간 정보 인덱스도 마찬가지로 인덱스를 생성할때 필드의 순서를 카디널리티가 높은 필드 순서대로 둬서 결과를 더 많이 필터링하도록 하는게 좋다.



### 2d 인덱스

- 비구체 지도 (비디오 게임 지도, 시계열 데이터 등등 ..) 에는 2dsphere 대신 2d 인덱스를 사용한다.
- 2d 인덱스
  - 2d 인덱스는 지형이 구체가 아니라 완전히 평평한 표면이라고 가정한다.
  - 구체에 2d 인덱스를 사용하면 왜곡이 매우 심하므로 사용해서는 안된다.
  - 2d 인덱스는 점만 인덱싱할 수 있음로 GeoJSON 형태의 데이터는 저장하지 말아야 한다.
  - 2d 인덱스는 점의 배열을 저장할 수 있지만 이는 선과 다르다. $geoWithin 쿼리에서 점 하나가 주어진 도형 안에 있으면 해당 도큐먼트가 $geoWithin과 일치한다.

```
db.hyrule.createIndex({"tile" : "2d"})
```

- 기본적으로 2d 인덱스는 값이 -180과 180 사이에 있다고 가정하기 때문에 범위를 넓히거나 좁히려면 createIndex를 사용해 최솟값과 최댓값을 지정한다.

```
db.hyrule.createIndex({"light-years" : "2d"}, {"min" : -1000, "max" : 1000})
```

- 2d인덱스가 지원하는 쿼리 셀렉터
  - $geoWithin
  - $nearSphere
  - $near

```
// 왼쪽하단 모서리 10,10과 오른쪽 상단 모서리 100,100 으로 정의된 사각형 내 도큐먼트에 대한 쿼리
db.hyrule.find({ tile: { $geoWithin: { $box: [[10, 10], [100, 100]] } } })
```

- $box의 파라미터
  - 첫 번째 요소는 왼쪽 하단 모서리 좌표
  - 두 번째 요소는 오른쪽 상단 모서리 좌표

```
// 중심이 17, 20.5 이고 반지름이 25인 원 안에 있는 도큐먼트에 대한 쿼리
db.hyrule.find({ tile: { $geoWithin: { $center: [[-17, 20.5] , 25] } } })
```

```
// 다각형 쿼리
db.hyrule.find({ tile: { $geoWithin: { $polygon: [[0, 0], [3, 6], [6, 0]] } } )
```

- 몽고DB는 레거시 지원을 위해 2d인덱스에 대한 기초적인 구형 쿼리도 지원한다.
- 구형쿼리에서는 2dsphere 인덱스를 사용해야하지만 구 안에 있는 레거시 좌표 쌍을 쿼리하려면 $centerSphere 연산자와 함께 $geoWithin을 사용할 수 있다.

```
db.hyrule.find({ loc: { $geoWithin: { $centerSphere: [[88, 30], 10/3963.2] } } })
```

- 주변에 있는 점을 쿼리하려면 $near를 사용한다. 근접 쿼리는 특정 지점으로부터 가장 가까운 좌표 쌍을 포함하는 도큐먼트를 반환하고 결과를 거리 순으로 정렬한다.

```
db.hyrule.find({"tile" : {"$near" : [20, 21]}})
```



## 전문 검색을 위한 인덱스

- text 인덱스
  - 몽고DB의 text 인덱스는 전문 검색의 요구사항을 지원한다.
  - 애플리케이션 사용자가 제목, 설명 등 컬렉션 내에 있는 필드의 텍스트와 일치시키는 키워드 쿼리를 하게 하려면 text 인덱스를 사용하자.
  - 정규 표현식을 이용해 문자열을 쿼리할 수도 있지만 속도가 느리고 언어 특성을 반영하기도 쉽지 않다 ex :entry와 entries는 정규표현식으로 검색하기 쉽지 않다.
  - 몽고DB의 text 인덱스는 텍스트를 빠르게 검색하는 기능을 제공하며 언어에 적합한 토큰화, stop word, 형태소 분석 등 일반적인 검색 엔진 요구사항을 지원한다.
- text 인덱스 생성
  - text 인덱스에서 필요한 키의 개수는 인덱싱 되는 필드의 단어 개수에 비례한다. 따라서 text 인덱스를 만들면 시스템 리소스가 많이 소비될 수 있다.
  - text 인덱스 생성은 애플리케이션 성능에 부정적인 영향을 미치지 않을 때 생성해야 하며, 가능하면 백그라운드에서 인덱스를 구축해야 한다.
  - 우수한 성능을 보장하려면 생성하는 모든 text 인덱스가 램에 맞는지 확인해야 한다.
- text 인덱스와 쓰기 성능
  - text 인덱스에서 쓰기가 발생하면 문자열이 토큰화되고, 형태소화되며, 인덱스는 잠재적으로 여러 위치에서 갱신된다.
  - text 인덱스에 대한 쓰기는 일반적으로 단일, 복합 인덱스에 대한 쓰기보다 더 많은 비용이 발생한다. 따라서 쓰기 성능이 떨어지는 경향이 있다.
  - 샤딩된 상태에선 데이터 이동 속도가 느려지며, 모든 텍스트는 새 샤드로 마이그레이션 될 때 다시 인덱싱 되어야 한다.



### 텍스트 인덱스 생성

```
db.articles.createIndex({"title": "text", "body" : "text"})
```

- 키에 순서가 있는 일반적인 복합 인덱스와 달리 기본적으로 각 필드는 text 인덱스에서 동등하게 고려된다.
- 가중치를 지정하면 몽고DB가 각 필드에 지정하는 상대적 중요도를 제어할 수 있다.

```
db.articles.createIndex({"title": "text", "body": "text"}, {"weights" : { "title" : 3, "body" : 2}})
```

- 인덱스를 생성한 후에는 가중치를 변경할 수 없다. 삭제 후 다시 생성해야 한다. 따라서 상용 데이터에 인덱스를 생성하기 전에 샘플 데이터셋에 가중치를 적용해보는게 좋다.
- 컬렉션에 따라 도큐먼트에 포함될 필드를 모들 수도 있는데 이 경우 아래와 같이 모든 문자열 필드에 전문 인덱스를 생성할 수 있다.
- 모든 최상위 문자열 필드를 인덱싱할 뿐 아니라 내장 도큐먼트와 배열의 문자열 필드를 인덱싱한다.

```
db.articles.createIndex({"$**" : "text"})
```



### 텍스트 검색

- $text 쿼리 연산자
  - $text 쿼리 연산자를 사용해 text 인덱스가 있는 컬렉션에 텍스트 검색을 수행할 수 있다.
  - $text는 공백과 구두점을 구분 기호로 사용해 검색 문자열을 토큰화하며, 검색문자열에서 모든 토큰의 논리적 OR를 수행한다.

```
// impact, crater, lunar 라는 용어가 포함된 기사를 모두 찾는 쿼리
db.articles.find({"$text": {"$search": "impact crater lunar"}}, {title: 1} ).limit(10)
```

- 위 쿼리를 실행하면 예상과는 다르게 그다지 관련성 없는 도큐먼트들이 반환된다.
  - 몽고DB가 OR를 사용해서 실행한다는 점을 고려하면 쿼리가 매우 광범위하다.
  - 텍스트 검색은 기본적으로 결과를 관련성에 따라 정렬하지 않는다.
- 구문을 사용해 쿼리 자체의 문제를 해결할 수 있다. 텍스트를 큰따옴표로 묶어서 정확히 일치하는 구문을 검색할 수 있다.

```
// "impact crater" AND "lunar" 로 처리하는 쿼리
db.articles.find({$text: {$search: "\"impact crater\" lunar"}}, {title: 1}).limit(10)

// "impact crater" AND  ("lunar" OR "meter") 로 처리하는 쿼리
db.articles.find({$text: {$search: "\"impact crater\" lunar meteor"}}, {title: 1}).limit(10)

// "impact crater" AND "lunar" AND "meter" 로 처리하는 쿼리
db.articles.find({$text: {$search: "\"impact crater\" \"lunar\" \"meteor\""}}, {title: 1}).limit(10)
```

- 텍스트 쿼리를 사용하면 각 쿼리 결과에 메타데이터가 연결된다. 메타데이터는 $meta 연산자를 사용해 명시적으로 투영하지 않는 한 쿼리 결과에 표시되지 않는다.
- 관련성 스코어는 textScore라는 메타 데이터 필드에 저장된다.

```
db.articles.find({$text: {$search: "\"impact crater\" lunar"}}, {title: 1, score: {$meta: "textScore"}}).limit(10)
```

- textScore를 정렬하면 관련성에 따라 정렬된 결과를 확인할 수 있다.

```
db.articles.find({$text: {$search: "\"impact crater\" lunar"}}, {title: 1, score: {$meta: "textScore"}} ).sort({score: {$meta: "textScore"}}).limit(10)
```





### 전문 검색 최적화

- 다른 기준으로 검색 결과를 좁힐 수 있다면 복합 인덱스를 생성할 때 다른 기준을 첫 번째로 두고 전문 필드를 그 다음으로 둔다.

```
db.blog.createIndex({"date" : 1, "post" : "text"})
```

- 전문 인덱스를 date에 따라 몇 개의 작은 트리 구조로 쪼개며 이를 파티셔닝이라고 한다. 전문 인덱스를 분할해 특정 날짜에 대한 전문 검색을 훨씬 빨리할 수 있다.
- 또한 다른 기준을 뒤쪽에 두어 사용할 수도 있다. "author"와 "post" 필드만 반환한다면 두 필드에 대해 복합 인덱스를 생성할 수도 있다.

```
db.blog.createIndex({"post" : "text", "author" : 1})
```



### 다른 언어로 검색하기

- 언어에 따라 형태소 분석 방법이 다르므로 인덱스나 도큐먼트가 어떤 언어로 쓰였는지 명시해야 한다 
- text 인덱스에는 default_language 옵션을 지정할 수 있고 기본값은 english다.
  - https://docs.mongodb.com/manual/reference/text-search-languages/#text-search-languages
  - no korean ... ㅎㅎ

```
// 프랑스어 인덱스
db.users.createIndex({"profil" : "text", "intérêts" : "text"}, {"default_language" : "french"})
```

- 도큐먼트의 언어를 language 필드에 명시해 도큐먼트별로 형태소 분석 언어를 다르게 지정할 수 있다.

```
db.users.insert({"username" : "swedishChef", "profile" : "Bork de bork", language : "swedish"})
```



## 제한 컬렉션

- 몽고DB의 일반적인 컬렉션은 동적으로 생성되고 추가적인 데이터에 맞춰 크기가 자동으로 늘어난다.
- 몽고DB는 제한 컬렉션이라는 다른 형태의 컬렉션을 지원하는데 이는 미리 생성돼 크기가 고정된다.
- 제한 컬렉션은 빈 공간이 없으면 오래된 도큐먼트가 지워지고 새로운 도큐먼트가 그 자리를 차지한다.
- 제한 컬렉션의 제약사항
  - 제한 컬렉션에서 도큐먼트를 임의로 삭제할 수 없다.
  - 도큐먼트 크기가 커지도록 하는 갱신은 허용되지 않는다. 
  - 제한 컬렉션은 샤딩될 수 없다.
- 제한 컬렉션의 도큐먼트는 입력 순서대로 저장되며, 삭제된 도큐먼트로 인해 생긴 가용 저장 공간 목록을 유지할 필요가 없다.
- 제한 컬렉션은 디스크의 고정된 영역에 순서대로 저장되기 때문에 회전 디스크에서 쓰기를 다소 빠르게 수행할 수 있다. 
- 제한 컬렉션은 유연성이 부족하지만 로깅에서는 나름 유용하게 사용된다.



### 제한 컬렉션 생성

- 제한 컬렉션은 사용되기 전에 명시적으로 생성돼야 한다.

```
// 10만 바이트 고정 크기로 제한 컬렉션 생성하기
db.createCollection("my_collection", {"capped" : true, "size" : 100000});

// 도큐먼트 수를 제한하기
db.createCollection("my_collection2", {"capped" : true, "size" : 100000, "max" : 100});

```

- 제한 컬렉션은 생성 후 변경이 불가능하기 때문에 큰 컬렉션을 생성하기 전에는 신중히 검토해야 한다.
- 제한 컬렉션 동작 방식
  - max 만큼의 도큐먼트수가 도달하거나
  - size 만큼의 저장공간에 도달하거나
  - 둘 중 하나에 도달하면 오래된 도큐먼트를 삭제한다
- 제한 컬렉션을 일반 컬렉션으로 변환하는 방법은 없다. 하지만 일반 컬렉션을 제한 컬렉션으로 변환할 수 있다.

```
db.runCommand({"convertToCapped" : "test", "size" : 10000});
```



### 꼬리를 무는 커서

- 꼬리를 무는 커서는 결과를 모두 꺼낸 후에도 종료되지 않는 특수한 형태의 커서다. tail -f 명령어에서 영감을 받아 만들어졌다.
- tail -f 와 같이 커서는 결과를 다 꺼내도 종료되지 않으므로 컬렉션에 데이터가 추가되면 새로운 결과를 바로 꺼낼 수 있다.
- 일반 컬렉션에서는 입력 순서가 추적되지 않기 때문에 꼬리를 무는 커서는 제한 컬렉션에서만 사용가능하다.
- 꼬리를 무는 커서는 아무런 결과가 없으면 10분 후 종료된다.
- 커서는 시간이 초과되거나 누군가가 쿼리 작업을 중지할 때까지는 결과를 처리하거나 다른 결과가 더 도착하기를 기다린다.

<br>

- 일반적으로 몽고DB TTL 인덱스가 제한 컬렉션보다 더 나은 성능을 발휘하므로 제한 컬렉션보다 TTL 인덱스가 더 권장된다.



## TTL 인덱스

- TTL 인덱스에서 도큐먼트는 미리 설정한 시간에 도달하면 지워진다.
- 이런 인덱스는 세션 스토리지와 같은 문제를 캐싱하는 데 유용하다.

```
// 도큐먼트에 lastUpdated가 날짜형필드라면 서버시간이 expireAfterSeconds만큼 지났을때 도큐먼트가 삭제된다.
db.sessions.createIndex({"lastUpdated" : 1}, {"expireAfterSeconds" : 60*60*24})
```

- 도큐먼트의 lastUpdated필드가 갱신된다면 다시 expireAfterSeconds만큼 기다리고 삭제한다.
- 몽고DB는 TTL 인덱스를 매분마다 청소하므로 초단위로 신경쓸 필요가 없다 ..? 초단위는 불가능?? 뭔말 ..?
- collMod 명령어를 이용해 expireAfterSeconds를 변경할 수 있다.

```
db.runCommand( {"collMod" : "someapp.cache" , "index" : { "keyPattern" : {"lastUpdated" : 1} , "expireAfterSeconds" : 3600 } } );
```

- 하나의 컬레션에 TTL 인덱스를 여러 개 가질 수 있다. 복합 인덱스는 될 수 없지만 정렬 및 쿼리 최적화가 목적이라면 일반 인덱스처럼 사용될 수 있다.



## GridFS로 파일 저장하기

- GridFS는 몽고DB에 대용량 이진 파일을 저장하는 메커니즘이다.
- GridFS를 고려하는 몇가지 이유
  - GridFS를 사용하면 아키텍처 스택을 단순화할 수 있다. 이미 몽고DB를 사용중이라면 파일스토리지를 위한 별도의 도구 대신 GridFS를 사용하면 된다.
  - GridFS 몽고DB를 위해 설정한 기존의 복제나 자동 샤딩을 이용할 수 있어, 파일 스토리지를 위한 장애 조치와 분산 확장이 더욱 쉽다.
  - GridFS는 사용자가 올린 파일을 저장할 때 특정 파일시스템이 갖는 문제를 피할 수 있다. ex: GridFS는 같은 디렉토리에 대량의 파일을 저장해도 문제가 없다.
- GridFS의 단점
  - 성능이 느리다. 
  - 도큐먼트를 수정하려면 도큐먼트 전체를 삭제하고 다시 저장해야 한다.
- GridFS는 큰 변화가 없고 순차적인 방식으로 접근하려는 대용량 파일을 저장할 때 일반적으로 최선의 메커니즘이다.



### GridFS 시작하기: mongofiles

- mongofiles 유틸리티로 쉽게 GridFS를 설치하고 운영할 수 있다.
- mongofiles는 모든 몽고DB 배포판에 포함되며 GridFS에서 파일을 올리고, 받고, 목록을 출력하고, 검색하고, 삭제할 때 등에 사용한다.

```
// 파일을 받아 GridFS에 저장
# mongofiles put ./restaurants.json
2022-10-30T23:46:20.415+0000	connected to: mongodb://localhost/
2022-10-30T23:46:20.416+0000	adding gridFile: ./restaurants.json
2022-10-30T23:46:20.624+0000	added gridFile: ./restaurants.json

// GridFS의 목록 보기
# mongofiles list
2022-10-30T23:46:44.196+0000	connected to: mongodb://localhost/
./restaurants.json	3544930

// GridFS -> 파일 시스템에 저장하기
# mongofiles get ./restaurants.json
2022-10-30T23:47:32.638+0000	connected to: mongodb://localhost/
2022-10-30T23:47:32.961+0000	finished writing to ./restaurants.json
```



### 몽고DB 드라이버로 GridFS 작업하기

- 모든 클라이언트 라이브러리는 GridFS API를 가진다.
- 자세한 내용은 자신이 사용하는 라이브러리의 문서를 참조할 것 ...



### 내부 살펴보기

- GridFS는 파일 저장을 위한 간단한 명세이며 일반 몽고DB 도큐먼트를 기반으로 만들어졌다.
- 몽고DB 서버는 GridFS 요청을 처리하면서 특별한 작업은 거의하지 않고 모든 작업은 클라이언트쪽의 드라이버 도구가 처리한다.
- GridFS의 기본 개념은 대용량 파일을 청크로 나눈 후 각 청크를 도큐먼트로 저장할 수 있다는 것이다.
- 파일의 청크를 저장하는 작업 외에도 여러 청크를 묶고 파일의 메타데이터를 포함하는 단일 도큐먼트를 만든다.
- GridFS의 청크는 기본적으로 fs.chunks 컬렉션에 저장되지만 변경할 수 있다.

```
// 청크 내 컬렉션 도큐먼트 구조 
{
  "_id" : ...,
  "n" : 0,
  "data" : BinData("..."),
  "files_id" : ObjectId("...")
}
```

- files_id: 청크에 대한 메타데이터를 포함하는 파일 도큐먼트의 id
- n: 다른 청크를 기준으로하는 파일 내 청크의 위치
- data: 파일 내 청크의 크기
- 각 파일의 메타데이터는 기본적으로 별도의 컬렉션인 fs.files에 저장된다.
- fs.files의 구조
  - \_id : 파일의 고유id
  - length : 파일 내용의 총 바이트 수
  - chunkSize : 파일을 구성하는 각 청크의 크기, 단위는 바이트 기본적으로 256킬로바이트지만 필요시 조정할 수 있다.
  - uploadDate : GridFS에 파일이 저장된 시간
  - md5 : 체크썸
- md5 값은 몽고DB에서 filemd5 명령어를 사용해 생성하는데 사용자가 파일이 제대로 올라갔는지 확인하려면 md5 키의 값을 확인해볼 수 있다.



# 