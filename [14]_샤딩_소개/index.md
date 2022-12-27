# #14 샤딩 소개

- 이 장에서 배울 수 있는 것
  - 샤딩과 클러스터 구성 요소 소개
  - 샤딩 구성 방법
  - 샤딩과 애플리케이션의 상호 작용 기본





## 샤딩이란

- 샤딩은 여러 장비에 걸쳐 데이터를 분할하는 과정을 일컬으며 때때로 파티셔닝이라는 용어로도 불린다.
- 샤딩의 용도
  - 각 장비에 데이터의 서브셋을 넣음으로써 더 많은 수의 덜 강력한 장비로 더 많은 데이터를 저장하고 더 많은 부하를 처리한다.
  - 더 자주 접근하는 데이터를 성능이 더 좋은 하드웨어에 배치할 수 있다.
  - 지역에 따라 데이터셋을 분할해 주로 접근하는 애플리케이션 서버와 가까운 컬렉션에서 도큐먼트의 서브셋을 찾을 수 있다.
- 수동 샤딩
  - 수동 샤딩은 어떤 데이터베이스 소프트웨어를 사용하든 대부분 수행할 수 있다.
  - 수동 샤딩을 사용하면 애플리케이션은 여러 데이터베이스 서버와 연결을 유지하고 각기 다른 데이터를 다른 서버에 저장하고 데이터를 가져오기 위해 적절한 서버에 쿼리하는 과정을 관리한다.
  - 수동 샤딩은 잘 동작하지만 클러스터에서 노드를 추가하거나 삭제할 때 혹은 데이터 분산이나 부하 패턴이 변화할 때는 유지하기 어렵다.
- 자동 샤딩
  - 몽고DB는 애플리케이션에서 구조를 추상화하고 시스템 관리를 간단하게 하는 자동 샤딩을 지원한다.
  - 몽고DB가 샤드에 걸쳐 데이터 분산을 자동화하므로 용량을 추가하고 제거하기 쉽다.
- 개발 및 운영 측면에서 샤딩은 몽고DB를 구성하는 가장 어렵고 복잡한 방법이다.
  - 구성하고 모니터링할 구성 요소가 많고 클러스터에서 데이터가 자동으로 옮겨다니기 때문이다.
- 샤드 클러스터를 배포하기 전에 standalone 서버와 복제 셋을 다루는 데 충분히 익숙해져야 한다.
- 샤드 클러스터를 구성할때 좋은 도구들
  - 몽고DB 옵스 매니저: 컴퓨팅 자원 관리 툴
    - https://www.mongodb.com/products/ops-manager
  - 몽고DB 아틀라스: 인프라를 몽고DB에 맡길 수 있는 경우 권장 (AWS, Azure, GCP 에서 실행 가능)
    - https://www.mongodb.com/cloud/atlas/lp/try4





### 클러스터 구성 요소 이해하기

- 몽고DB 샤딩을 통해 많은 장비의 클러스터를 생성하고 각 샤드에 데이터 서브셋을 넣음으로써 컬렉션을 쪼갤 수 있다.
- 복제 vs 샤딩
  - 복제는 여러 서버에 데이터의 복사본을 생성하므로 모든 서버는 다른 서버의 미러 이미지다.
  - 샤딩은 각각 다른 데이터 서브셋이다.
- mongos
  - 샤딩의 한가지 목적은 n개의 샤드 클러스터가 하나의 장비처럼 보이게 하는 것이다.
  - 이러한 세부 사항을 애플리케이션으로부터 숨기기 위해, 샤드 앞단에 있는 mongos라는 라우팅 프로세스를 실행한다.
  - 애플리케이션은 라우터(mongos)에 연결해 정상적으로 요청을 발행할 수 있다.
  - mongos는 어떤 데이터가 어떤 샤드에 있는지 알기 때문에 요청을 적절한 샤드로 전달할 수 있다.
  - 요청에 대한 응답이 있으면 라우터는 응답을 수집하고 통합해 애플리케이션으로 되돌려 보낸다.

![14-1](https://github.com/sup2is/dev-note/raw/master/db/images/mongodb/14-1.jpeg)

## 단일 장비 클러스터에서의 샤딩

- ShardingTest
  - 간편하게 클러스터를 만들려면 ShardingTest 클래스를 사용한다.
  - ShardingTest는 몽고DB 엔지니어링 내부용으로 설계된 클래스이므로 외부적으로 문서화되지 않았지만 몽고DB 서버와 함께 제공되므로 샤드 클러스터를 실험하는 가장 간단한 수단이다.
  - ShardingTest는 서버 테스트셋을 지원하도록 설계됐다.

```
st = ShardingTest({
 name:"one-min-shards",
 chunkSize:1,
 shards:2,
 rs:{
 nodes:3,
 oplogSize:10
 },
 other:{
 enableBalancer:true
 }
});
```

- ShardingTest 에 전달된 옵션
  - chunkSize: 데이터의 조각을 기본적으로 chunk라는 단위로 관리. chunk가 적을수록 데이터는 더 많은곳으로 분배되고 chunk가 클수록 데이터는 적게 분산됨
  - name: 샤드 클러스터에 대한 lable
  - shards: 클러스터가 두 개의 사드로 구성되도록 지정
  - rs: 각 샤드를 oplogSize가 10Mib인 3-노드 복제 셋으로 정의
  - other.enableBalancer: 클러스터가 스핀 업 되면서 밸런서를 활성화하도록 설정. 청크 수를 모니터링하고 있다가 각 샤드당 동일한 수의 청크 수를 갖게 하기 위해 밸런서가 동작함
- ShardingTest가 클러스터 설정을 마치면 연결 가능한 실행중인 프로세스는 7개이고 아래와 같다.
  - 노드 3개로 구성된 복제 셋 1개
  - 노드 3개로 구성된 서버 복제 셋 1개
  - mongos 1개
- mongos는 기본적으로 20009에서 실행되므로 다른 쉘에서 mongos에 연결할 수 있다.

```
db = (new Mongo("localhost:20009")).getDB("accounts")
```

- 이제 mongos는 요청을 샤드로 라우팅한다. 클라이언트(셸)는 샤드의 정보는 전혀 몰라도 된다.

```
for (var i=0; i<100000; i++) { db.users.insert({"username" : "user"+i, "created_at" : new Date()}); }
db.users.count()
//10만개
```

```
sh.status()
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("638332c6d00c858fb88f6020")
  }
  shards:
        {  "_id" : "one-min-shards-rs0",  "host" : "one-min-shards-rs0/e618035acdb9:20000,e618035acdb9:20001,e618035acdb9:20002",  "state" : 1,  "topologyTime" : Timestamp(1669542606, 1) }
        {  "_id" : "one-min-shards-rs1",  "host" : "one-min-shards-rs1/e618035acdb9:20003,e618035acdb9:20004,e618035acdb9:20005",  "state" : 1,  "topologyTime" : Timestamp(1669542609, 1) }
  active mongoses:
        "5.0.9" : 1
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled: yes
        Currently running: no
        Failed balancer rounds in last 5 attempts: 0
        Migration results for the last 24 hours: 
                No recent migrations
  databases:
        {  "_id" : "accounts",  "primary" : "one-min-shards-rs0",  "partitioned" : false,  "version" : {  "uuid" : UUID("c4e42e7d-b10f-4102-aaf0-f8e9283c3bc7"),  "timestamp" : Timestamp(1669543117, 2),  "lastMod" : 1 } }
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
```

- sh 변수 
  - 여러 샤딩 보조자 함수를 정의하는 전역 변수다.
  - sh.help()를 실행하면 도움말을 볼 수 있다.
  - sh.status()를 실행해서 클러스터의 상태를 볼 수 있다.
    - [https://www.mongodb.com/docs/manual/reference/method/sh.status/#output-fields](https://www.mongodb.com/docs/manual/reference/method/sh.status/#output-fields)
- 프라이머리 샤드
  - mongos는 클러스터에서 데이터의 양이 가장 적은 샤드를 선택해서 새 데이터베이스를 생성할 때 프라이머리 샤드로 지정한다.
  - 프라이머리 샤드는 클러스터에서 샤딩되지 않은 모든 컬렉션을 보유하고 있다.
  - 이전 장에서 배웠던 프라이머리 샤드와 복제 셋의 프라이머리는 다르다.
- 샤딩을 활성화해야 데이터베이스 내에서 컬렉션을 샤딩할 수 있다.

```
sh.enableSharding("accounts")
```

- 샤드 키
  - 컬렉션을 샤딩할 때 샤드 키를 선택하는데 이는 몽고DB가 데이터를 분할하는 데 사용하는 필드다.
  - 예를 들어 username 필드를 샤딩하도록 선택하면 몽고DB는 데이터를 사용자 이름 범위로 나눈다.
  - 샤드키는 인덱싱과 유사한 개념이고 컬렉션이 커질수록 샤드 키가 컬렉션에서 가장 중요한 인덱스가 된다. 따라서 샤드키는 신중하게 선택해야 한다.
  - 샤드키를 만들려면 먼저 필드에 인덱스를 생성해야 한다.

```
db.users.createIndex({"username" : 1})

sh.shardCollection("accounts.users", {"username": 1})
{
	"collectionsharded" : "accounts.users",
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1669594445, 26),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1669594445, 22)
}

--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("638332c6d00c858fb88f6020")
  }
  shards:
        {  "_id" : "one-min-shards-rs0",  "host" : "one-min-shards-rs0/e618035acdb9:20000,e618035acdb9:20001,e618035acdb9:20002",  "state" : 1,  "topologyTime" : Timestamp(1669542606, 1) }
        {  "_id" : "one-min-shards-rs1",  "host" : "one-min-shards-rs1/e618035acdb9:20003,e618035acdb9:20004,e618035acdb9:20005",  "state" : 1,  "topologyTime" : Timestamp(1669542609, 1) }
  active mongoses:
        "5.0.9" : 1
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled: yes
        Currently running: no
        Failed balancer rounds in last 5 attempts: 0
        Migration results for the last 24 hours: 
                3 : Success
  databases:
        {  "_id" : "account",  "primary" : "one-min-shards-rs1",  "partitioned" : true,  "version" : {  "uuid" : UUID("b6f1b863-8884-4c69-9676-6e6d392b1960"),  "timestamp" : Timestamp(1669594383, 1),  "lastMod" : 1 } }
        {  "_id" : "accounts",  "primary" : "one-min-shards-rs0",  "partitioned" : true,  "version" : {  "uuid" : UUID("c4e42e7d-b10f-4102-aaf0-f8e9283c3bc7"),  "timestamp" : Timestamp(1669543117, 2),  "lastMod" : 1 } }
                accounts.users
                        shard key: { "username" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                one-min-shards-rs0	3
                                one-min-shards-rs1	4
                        { "username" : { "$minKey" : 1 } } -->> { "username" : "user1" } on : one-min-shards-rs1 Timestamp(2, 0) 
                        { "username" : "user1" } -->> { "username" : "user2288" } on : one-min-shards-rs1 Timestamp(4, 0) 
                        { "username" : "user2288" } -->> { "username" : "user33609" } on : one-min-shards-rs1 Timestamp(4, 1) 
                        { "username" : "user33609" } -->> { "username" : "user44696" } on : one-min-shards-rs0 Timestamp(3, 6) 
                        { "username" : "user44696" } -->> { "username" : "user54548" } on : one-min-shards-rs0 Timestamp(4, 2) 
                        { "username" : "user54548" } -->> { "username" : "user9999" } on : one-min-shards-rs0 Timestamp(4, 3) 
                        { "username" : "user9999" } -->> { "username" : { "$maxKey" : 1 } } on : one-min-shards-rs1 Timestamp(3, 0) 
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
```

- 컬렉션은 n개의 청크로 분할되었고 각 청크는 데이터의 서브셋인것을 확인할 수 있다.
- 청크는 샤드 키 범위에 따라 나열된다 
  - ex: { "username" : "user1" } -->> { "username" : "user2288" }

![14-3](https://github.com/sup2is/dev-note/raw/master/db/images/mongodb/14-3.webp)

- 하나의 컬렉션에서 샤드키를 기반으로 컬렉션을 더 작은 청크로 구분한다. 분할된 청크는 여러 클러스터에 분산된다.
- 샤드 키 값
  - $minkey: 음의 무한대
  - $maxkey: 양의 무한대
  - 따라서 항상 샤드 키 값은 항상 $minkey와 $maxkey 사이에 있다.
  - 이러한 변수값은 BSON 유형이고 애플리케이션에서 사용해서는 안되고 내부에서만 사용해야한다.
- 실제로 샤딩된 컬렉션에 explain() 명령을 통해 쿼리가 어떻게 나가는지 확인할 수 있다.

```
db.users.find({username: "user12345"}).explain()

{
	"queryPlanner" : {
		"mongosPlannerVersion" : 1,
		"winningPlan" : {
			"stage" : "SINGLE_SHARD",
			"shards" : [
				{
					"shardName" : "one-min-shards-rs1",
					"connectionString" : "one-min-shards-rs1/e618035acdb9:20003,e618035acdb9:20004,e618035acdb9:20005",
					"serverInfo" : {
						"host" : "e618035acdb9",
						"port" : 20003,
						"version" : "5.0.9",
						"gitVersion" : "6f7dae919422dcd7f4892c10ff20cdc721ad00e6"
					},
					"namespace" : "accounts.users",
					"indexFilterSet" : false,
					"parsedQuery" : {
						"username" : {
							"$eq" : "user12345"
						}
					},
					"queryHash" : "379E82C5",
					"planCacheKey" : "F335C572",
					"maxIndexedOrSolutionsReached" : false,
					"maxIndexedAndSolutionsReached" : false,
					"maxScansToExplodeReached" : false,
					"winningPlan" : {
						"stage" : "FETCH",
						"inputStage" : {
							"stage" : "IXSCAN",
							"keyPattern" : {
								"username" : 1
							},
							"indexName" : "username_1",
							"isMultiKey" : false,
							"multiKeyPaths" : {
								"username" : [ ]
							},
							"isUnique" : false,
							"isSparse" : false,
							"isPartial" : false,
							"indexVersion" : 2,
							"direction" : "forward",
							"indexBounds" : {
								"username" : [
									"[\"user12345\", \"user12345\"]"
								]
							}
						}
					},
					"rejectedPlans" : [ ]
				}
			]
		}
	},
	"serverInfo" : {
		"host" : "e618035acdb9",
		"port" : 20009,
		"version" : "5.0.9",
		"gitVersion" : "6f7dae919422dcd7f4892c10ff20cdc721ad00e6"
	},
	"serverParameters" : {
		"internalQueryFacetBufferSizeBytes" : 104857600,
		"internalQueryFacetMaxOutputDocSizeBytes" : 104857600,
		"internalLookupStageIntermediateDocumentMaxSizeBytes" : 104857600,
		"internalDocumentSourceGroupMaxMemoryBytes" : 104857600,
		"internalQueryMaxBlockingSortMemoryUsageBytes" : 104857600,
		"internalQueryProhibitBlockingMergeOnMongoS" : 0,
		"internalQueryMaxAddToSetBytes" : 104857600,
		"internalDocumentSourceSetWindowFieldsMaxMemoryBytes" : 104857600
	},
	"command" : {
		"find" : "users",
		"filter" : {
			"username" : "user12345"
		},
		"lsid" : {
			"id" : UUID("3b1d29cf-0ec1-4f14-b351-14c8b3a3fa1b")
		},
		"$clusterTime" : {
			"clusterTime" : Timestamp(1669596202, 1),
			"signature" : {
				"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
				"keyId" : NumberLong(0)
			}
		},
		"$db" : "accounts"
	},
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1669596206, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1669594973, 14382)
}

```

- 쿼리플랜의 winningPlan.shards를 보면 어떤 노드를 사용해 사용자의 쿼리를 충족했음을 알 수 있다.
  - sh.status()로 샤드키범위와 실제 쿼리가 동작한 클러스터 멤버를 확인해보자.
- 만약 username (샤드키) 가 아니라 전체 사용자를 찾는 쿼리를 실행하면 mongos는 모든 클러스터에 쿼리한다.

```
{
	"queryPlanner" : {
		"mongosPlannerVersion" : 1,
		"winningPlan" : {
			"stage" : "SHARD_MERGE",
			"shards" : [
				{
					"shardName" : "one-min-shards-rs0",
					"connectionString" : "one-min-shards-rs0/e618035acdb9:20000,e618035acdb9:20001,e618035acdb9:20002",
					"serverInfo" : {
						"host" : "e618035acdb9",
						"port" : 20000,
						"version" : "5.0.9",
						"gitVersion" : "6f7dae919422dcd7f4892c10ff20cdc721ad00e6"
					},
					"namespace" : "accounts.users",
					"indexFilterSet" : false,
					"parsedQuery" : {
						
					},
					"queryHash" : "8B3D4AB8",
					"planCacheKey" : "D542626C",
					"maxIndexedOrSolutionsReached" : false,
					"maxIndexedAndSolutionsReached" : false,
					"maxScansToExplodeReached" : false,
					"winningPlan" : {
						"stage" : "SHARDING_FILTER",
						"inputStage" : {
							"stage" : "COLLSCAN",
							"direction" : "forward"
						}
					},
					"rejectedPlans" : [ ]
				},
				{
					"shardName" : "one-min-shards-rs1",
					"connectionString" : "one-min-shards-rs1/e618035acdb9:20003,e618035acdb9:20004,e618035acdb9:20005",
					"serverInfo" : {
						"host" : "e618035acdb9",
						"port" : 20003,
						"version" : "5.0.9",
						"gitVersion" : "6f7dae919422dcd7f4892c10ff20cdc721ad00e6"
					},
					"namespace" : "accounts.users",
					"indexFilterSet" : false,
					"parsedQuery" : {
						
					},
					"queryHash" : "8B3D4AB8",
					"planCacheKey" : "D542626C",
					"maxIndexedOrSolutionsReached" : false,
					"maxIndexedAndSolutionsReached" : false,
					"maxScansToExplodeReached" : false,
					"winningPlan" : {
						"stage" : "SHARDING_FILTER",
						"inputStage" : {
							"stage" : "COLLSCAN",
							"direction" : "forward"
						}
					},
					"rejectedPlans" : [ ]
				}
			]
		}
	},
	"serverInfo" : {
		"host" : "e618035acdb9",
		"port" : 20009,
		"version" : "5.0.9",
		"gitVersion" : "6f7dae919422dcd7f4892c10ff20cdc721ad00e6"
	},
	"serverParameters" : {
		"internalQueryFacetBufferSizeBytes" : 104857600,
		"internalQueryFacetMaxOutputDocSizeBytes" : 104857600,
		"internalLookupStageIntermediateDocumentMaxSizeBytes" : 104857600,
		"internalDocumentSourceGroupMaxMemoryBytes" : 104857600,
		"internalQueryMaxBlockingSortMemoryUsageBytes" : 104857600,
		"internalQueryProhibitBlockingMergeOnMongoS" : 0,
		"internalQueryMaxAddToSetBytes" : 104857600,
		"internalDocumentSourceSetWindowFieldsMaxMemoryBytes" : 104857600
	},
	"command" : {
		"find" : "users",
		"filter" : {
			
		},
		"lsid" : {
			"id" : UUID("3b1d29cf-0ec1-4f14-b351-14c8b3a3fa1b")
		},
		"$clusterTime" : {
			"clusterTime" : Timestamp(1669596206, 1),
			"signature" : {
				"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
				"keyId" : NumberLong(0)
			}
		},
		"$db" : "accounts"
	},
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1669596356, 3),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1669595396, 40)
}
```

- 타겟 쿼리
  - 샤드키를 포함하며 단일 샤드나 샤드 서브셋으로 보낼 수 있는 쿼리를 타겟 쿼리라고 한다.
- 분산-수집 쿼리
  - 모든 샤드로 보내야하는 쿼리는 분산-수집 쿼리라고 한다. mongos는 쿼리를 모든 샤드로 보내고 결과를 수집한다.



