# #10 복제 셋 설정

- 이 장에서 배울 수 있는 것
  - 복제 셋의 정의
  - 복제 셋을 설정하는 방법
  - 복제 셋 멤버 구성 옵션



## 복제 소개

![10-1](https://github.com/sup2is/dev-note/raw/master/db/images/mongodb/10-1.svg)

- 복제를 사용하는 이유
  - standalone server 형태의 단일 mongodb 서버를 실제 real 환경에서 사용하는 것은 매우 위험하기 때문에 복제를 사용해야 한다.
  - 복제는 데이터의 동일한 복사본을 여러 서버상에서 보관하는 방법이며 실제 서비스를 배포할 때 권장된다.
  - 복제는 한 대 또는 그 이상의 서버에 이상이 발생하더라도 애플리케이션이 정상적으로 동작하게 하고 데이터를 안전하게 보존한다.
- 몽고DB의 복제 
  - 몽고DB의 복제 셋은 클라이언트 요청을 처리하는 프라이머리 서버, 프라이머리 데이터의 복사본을 갖는 세컨더리 서버 여러대로 이뤄진다.
  - 프라이머리 서버에 장애가 발생하면 세컨더리 서버는 자신들 중에서 새로운 프라이머리 서버를 선출할 수 있다.
  - 복제를 사용하는 상태에서 서버가 다운되면, 복제 셋에 있는 다른 서버를 통해 데이터에 접근할 수 있다.
  - 서버상의 데이터가 손상되거나 접근할 수 없는 상태라면 복제 셋의 다른 멤버로부터 새로운  복제 데이터를 만들 수 있다.



![10-2](https://github.com/sup2is/dev-note/raw/master/db/images/mongodb/10-2.svg)



- 매커니즘엔 관심이 없고 단순히 테스트/개발/운영만 하고 싶다면 아래 솔루션을 사용하면 된다.
  - 몽고DB 옵스 매니저: 컴퓨팅 자원 관리 툴
  - https://www.mongodb.com/products/ops-manager
  - 몽고DB 아틀라스: 인프라를 몽고DB에 맡길 수 있는 경우 권장 (AWS, Azure, GCP 에서 실행 가능)
    - https://www.mongodb.com/cloud/atlas/lp/try4



## 복제 셋 설정 - 1장

- 운영 환경에서는 항상 복제 셋을 사용하며, 각 멤버에 전용 호스트를 할당해 리소스 경합을 방지하고 서버 오류에 대한 격리를 제공해야 한다. 추가적인 복원력을 제공하려면 DNS Seedlist 연결 형식 (https://www.mongodb.com/docs/manual/reference/connection-string/#dns-seed-list-connection-format) 을 사용해 애플리케이션이 복제 셋에 연결하는 방법을 지정해야 한다.
  - DNS를 사용하면 몽고DB 복제 셋 멤버를 호스팅하는 서버를 클라이언트에서 재구성할 필요 없이 돌아가면서 변경할 수 있다는 장점이 있다.

```
// docker-compose.yaml

version: '3.8'

services:
  mongo1:
    container_name: mongo1
    image: mongo:4.4
    volumes:
    networks:
      - mongo-network
    ports:
      - 27017:27017
    depends_on:
      - mongo2
      - mongo3
    links:
      - mongo2
      - mongo3
    restart: always
    entrypoint: [ "/usr/bin/mongod", "--bind_ip_all", "--replSet", "dbrs" ]

  mongo2:
    container_name: mongo2
    image: mongo:4.4
    networks:
      - mongo-network
    ports:
      - 27018:27017
    restart: always
    entrypoint: [ "/usr/bin/mongod", "--bind_ip_all", "--replSet", "dbrs" ]
  mongo3:
    container_name: mongo3
    image: mongo:4.4
    networks:
      - mongo-network
    ports:
      - 27019:27017
    restart: always
    entrypoint: [ "/usr/bin/mongod", "--bind_ip_all", "--replSet", "dbrs" ]

networks:
  mongo-network:
    driver: bridge
```



## 네트워크 고려 사항

- 복제 셋의 모든 멤버는 같은 셋 내 다른 멤버와 연결할 수 있어야 한다.

- 시작한 프로세스는 별도의 서버에서 쉽게 실행할 수 있지만 3.6 버전부터 mongod는 기본적으로 로컬호스트에만 바인딩 된다. 복제 셋의 각 멤버가 다른 멤버와 통신하려면 다른 멤버가 연결할 수 있는 ip 주소를 바인딩해야 한다.

  - 예를 들어 IP 주소가 198.51.100.1인 서버에서 mongod 인스턴스를 실행중이고 인스턴스를 각기 다른 서버의 멤버와 함께 복제 셋의 멤버로 실행하려면  --bind_ip를 지정하면 된다.

  - ```
    mongod --bind_ip localhost,192.51.100.1 --replSet mdbDefGuid  ...
    ```

  - 모든주소를 허용하려면 0.0.0.0 이나 --bind_ip_all 옵션을 사용하면 된다.



## 보안 고려 사항

- localhost 이외의 ip 주소에 바인딩하기 전 복제 셋을 구성할 때 권한 제어를 활성화하고 인증 메커니즘을 지정해야 한다.
- 디스크의 데이터를 암호화하고 복제 셋 멤버간 통신 및 셋과 클라이언트간 통신을 암호화하면 좋다.
- 복제 셋 보안은 19장에서 자세히..!



## 복제 셋 설정 - 2장

- 위 docker-compose.yaml은 아직 각 mongod가 다른 mongod의 존재를 알지 못한다.
- 각 멤버를 나열하는 구성을 만들어 mongod 프로세스 중 하나로 보내면 멤버들은 서로의 존재를 알게 된다.

```
// 연결 설정
rsconf = { _id: "dbrs", members: [ {_id: 0, host: "mongo1:27017"}, {_id: 1, host: "mongo2:27017"}, {_id: 2, host: "mongo3:27017"} ] }

rs.initiate(rsconf)
```

- 복제 셋 구성 도큐먼트

  - _id는 몽고DB를 실행할 때 선언한 이름이어야 한다.
  - members 배열
    -  _id는 정수이며 복제 셋 멤버 간에 고유해야 한다.
    -  host에는 mongod 인스턴스 서버를 나열한다.

- rs 보조자 함수

  - rs는 복제 보조자 함수를 포함하는 전역 변수다

  - rs.help()로 도움말을 확인할 수 있다.

  - 이 함수들은 거의 항상 데이터베이스 명령을 감싸는 래퍼이고 아래 명령어는 rs.initiate(config)와 동일한 역할을 수행한다.

    - ```
      db.adminCommand({"replSetInitiate" : config})
      ```

    - 보조자 대신 명령 양식을 사용하는 것이 더 쉬울 떄도 있으므로 보조자와 기본 명령 둘다 익히면 좋다.

```
// 복제 셋 설정 후 rs.status() 로 상태 확인하기
rs.status()

{
	"set" : "dbrs",
	"date" : ISODate("2022-11-14T14:13:12.569Z"),
	"myState" : 1,
	"term" : NumberLong(1),
	"syncSourceHost" : "",
	"syncSourceId" : -1,
	"heartbeatIntervalMillis" : NumberLong(2000),
	"majorityVoteCount" : 2,
	"writeMajorityCount" : 2,
	"votingMembersCount" : 3,
	"writableVotingMembersCount" : 3,
	"optimes" : {
		"lastCommittedOpTime" : {
			"ts" : Timestamp(1668435183, 1),
			"t" : NumberLong(1)
		},
		"lastCommittedWallTime" : ISODate("2022-11-14T14:13:03.007Z"),
		"readConcernMajorityOpTime" : {
			"ts" : Timestamp(1668435183, 1),
			"t" : NumberLong(1)
		},
		"readConcernMajorityWallTime" : ISODate("2022-11-14T14:13:03.007Z"),
		"appliedOpTime" : {
			"ts" : Timestamp(1668435183, 1),
			"t" : NumberLong(1)
		},
		"durableOpTime" : {
			"ts" : Timestamp(1668435183, 1),
			"t" : NumberLong(1)
		},
		"lastAppliedWallTime" : ISODate("2022-11-14T14:13:03.007Z"),
		"lastDurableWallTime" : ISODate("2022-11-14T14:13:03.007Z")
	},
	"lastStableRecoveryTimestamp" : Timestamp(1668435160, 1),
	"electionCandidateMetrics" : {
		"lastElectionReason" : "electionTimeout",
		"lastElectionDate" : ISODate("2022-11-14T00:32:28.212Z"),
		"electionTerm" : NumberLong(1),
		"lastCommittedOpTimeAtElection" : {
			"ts" : Timestamp(0, 0),
			"t" : NumberLong(-1)
		},
		"lastSeenOpTimeAtElection" : {
			"ts" : Timestamp(1668385937, 1),
			"t" : NumberLong(-1)
		},
		"numVotesNeeded" : 2,
		"priorityAtElection" : 1,
		"electionTimeoutMillis" : NumberLong(10000),
		"numCatchUpOps" : NumberLong(0),
		"newTermStartDate" : ISODate("2022-11-14T00:32:28.278Z"),
		"wMajorityWriteAvailabilityDate" : ISODate("2022-11-14T00:32:29.786Z")
	},
	"members" : [
		{
			"_id" : 0,
			"name" : "mongo1:27017",
			"health" : 1,
			"state" : 1,
			"stateStr" : "PRIMARY",
			"uptime" : 49533,
			"optime" : {
				"ts" : Timestamp(1668435183, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2022-11-14T14:13:03Z"),
			"lastAppliedWallTime" : ISODate("2022-11-14T14:13:03.007Z"),
			"lastDurableWallTime" : ISODate("2022-11-14T14:13:03.007Z"),
			"syncSourceHost" : "",
			"syncSourceId" : -1,
			"infoMessage" : "",
			"electionTime" : Timestamp(1668385948, 1),
			"electionDate" : ISODate("2022-11-14T00:32:28Z"),
			"configVersion" : 1,
			"configTerm" : 1,
			"self" : true,
			"lastHeartbeatMessage" : ""
		},
		{
			"_id" : 1,
			"name" : "mongo2:27017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 49255,
			"optime" : {
				"ts" : Timestamp(1668435183, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1668435183, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2022-11-14T14:13:03Z"),
			"optimeDurableDate" : ISODate("2022-11-14T14:13:03Z"),
			"lastAppliedWallTime" : ISODate("2022-11-14T14:13:03.007Z"),
			"lastDurableWallTime" : ISODate("2022-11-14T14:13:03.007Z"),
			"lastHeartbeat" : ISODate("2022-11-14T14:13:10.990Z"),
			"lastHeartbeatRecv" : ISODate("2022-11-14T14:13:10.990Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncSourceHost" : "mongo1:27017",
			"syncSourceId" : 0,
			"infoMessage" : "",
			"configVersion" : 1,
			"configTerm" : 1
		},
		{
			"_id" : 2,
			"name" : "mongo3:27017",
			"health" : 1,
			"state" : 2,
			"stateStr" : "SECONDARY",
			"uptime" : 49255,
			"optime" : {
				"ts" : Timestamp(1668435183, 1),
				"t" : NumberLong(1)
			},
			"optimeDurable" : {
				"ts" : Timestamp(1668435183, 1),
				"t" : NumberLong(1)
			},
			"optimeDate" : ISODate("2022-11-14T14:13:03Z"),
			"optimeDurableDate" : ISODate("2022-11-14T14:13:03Z"),
			"lastAppliedWallTime" : ISODate("2022-11-14T14:13:03.007Z"),
			"lastDurableWallTime" : ISODate("2022-11-14T14:13:03.007Z"),
			"lastHeartbeat" : ISODate("2022-11-14T14:13:10.990Z"),
			"lastHeartbeatRecv" : ISODate("2022-11-14T14:13:10.988Z"),
			"pingMs" : NumberLong(0),
			"lastHeartbeatMessage" : "",
			"syncSourceHost" : "mongo1:27017",
			"syncSourceId" : 0,
			"infoMessage" : "",
			"configVersion" : 1,
			"configTerm" : 1
		}
	],
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1668435183, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1668435183, 1)
}
```

- rs.status()을 사용하면 복제 셋에 대한 많은 정보들을 알려준다.
  - members 배열을 확인해보면 누가 프라이머리인지, 세컨더리인지 확인할 수 있다.
  - 그 외에 기타 자세한 정보는 https://www.mongodb.com/docs/manual/reference/command/replSetGetStatus/#output](https://www.mongodb.com/docs/manual/reference/command/replSetGetStatus/#output) 에서 확인할 수 있다.
- 복제셋은 서로 다른 데이터를 갖는 둘 이상의 멤버로 복제셋을 시작할 수 없다. 한 멤버에 있는 데이터로 셋을 시작하려면 그 데이터를 갖는 멤버로 복제셋을 구성해야 한다.



## 복제 관찰

- 복제 셋이 시작되고 mongo 셸이 현재 프라이머리에 연결되어 있다면 아래와 같이 프롬프트가 변경된다.

```
dbrs:PRIMARY>
```

- 이는 _id가 dbrs인 복제 셋의 프라이머리에 연결됐음을 의미한다.
- 프라이머리에서 쓰기와 읽기가 잘 되고 있는지 테스트해보자.

```
dbrs:PRIMARY> use test
switched to db test
dbrs:PRIMARY> for (i=0; i<1000; i++) {db.coll.insert({count: i})}
WriteResult({ "nInserted" : 1 })
dbrs:PRIMARY> db.coll.count()
//1000개 리턴
```

- 아래와 같이 세컨더리 커넥션을 만들고 복제가 잘 되고 있는지 확인할 수 있다.

```
dbrs:PRIMARY> secondaryConn = new Mongo("mongo2:27017")
connection to mongo2:27017

dbrs:PRIMARY> secondaryDB = secondaryConn.getDB("test")
test

dbrs:PRIMARY> secondaryDB.coll.find()
Error: error: {
	"topologyVersion" : {
		"processId" : ObjectId("63718b7a0b78f691477a7e59"),
		"counter" : NumberLong(4)
	},
	"operationTime" : Timestamp(1668467093, 1),
	"ok" : 0,
	"errmsg" : "not master and slaveOk=false",
	"code" : 13435,
	"codeName" : "NotPrimaryNoSecondaryOk",
	"$clusterTime" : {
		"clusterTime" : Timestamp(1668467093, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	}
}
```

- 세컨더리에서 오류가 발생한 이유
  - 세컨더리는 프라이머리보다 뒤처지며 최신 데이터가 아닐 수 있기 때문에 기본적으로 세컨더리는 애플리케이션이 실수로 실효 데이터를 읽지 않도록 요청을 거부한다.
  - errmsg에 보면 마스터가 아니고 slaveOk=false 인 것을 확인할 수 있다.
  - 세컨더리에 대한 쿼리를 허용하려면 아래처럼 세컨더리 설정을 해주어야한다.

```
// '세컨더리에서 읽어도 괜찮다' 라는 의미
dbrs:PRIMARY> secondaryConn.setSecondaryOk()

dbrs:PRIMARY> secondaryDB.coll.find()
{ "_id" : ObjectId("63724e8320ef5fd7afde6b7c"), "count" : 0 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b7d"), "count" : 1 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b7f"), "count" : 3 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b84"), "count" : 8 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b80"), "count" : 4 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b81"), "count" : 5 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b82"), "count" : 6 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b7e"), "count" : 2 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b83"), "count" : 7 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b85"), "count" : 9 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b86"), "count" : 10 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b87"), "count" : 11 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b88"), "count" : 12 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b89"), "count" : 13 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b8a"), "count" : 14 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b8b"), "count" : 15 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b8c"), "count" : 16 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b8d"), "count" : 17 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b8e"), "count" : 18 }
{ "_id" : ObjectId("63724e8320ef5fd7afde6b90"), "count" : 20 }
```

- setSecondaryOk() 는 데이터베이스가 아니라 connection에 설정됨을 주의하자.
- 세컨더리는 복제를 통한 쓰기만 수행하고 클라이언트에 대한 직접쓰기를 할 수 없으므로 쓰기를 시도하면 아래와 같은 에러가 발생한다.

```
dbrs:PRIMARY> secondaryDB.coll.insert({"count" : 1001})
WriteCommandError({
	"topologyVersion" : {
		"processId" : ObjectId("63718b7a0b78f691477a7e59"),
		"counter" : NumberLong(4)
	},
	"operationTime" : Timestamp(1668469314, 1),
	"ok" : 0,
	"errmsg" : "not master",
	"code" : 10107,
	"codeName" : "NotWritablePrimary",
	"$clusterTime" : {
		"clusterTime" : Timestamp(1668469314, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	}
})
dbrs:PRIMARY> secondaryDB.coll.count()
1000

```

- 자동 장애조치
  - 프라이머리가 종료되면 프라이머리의 중단을 가장 먼저 발견한 세컨더리가 새로운 프라이머리로 선출된다.

```
// 프라이머리 중지 (mongo1서버)
db.adminCommand({"shutdown" : 1})

// mongo2 서버로 재접속

dbrs:PRIMARY> db.isMaster()
{
	"topologyVersion" : {
		"processId" : ObjectId("63718b7a0b78f691477a7e59"),
		"counter" : NumberLong(7)
	},
	"hosts" : [
		"mongo1:27017",
		"mongo2:27017",
		"mongo3:27017"
	],
	"setName" : "dbrs",
	"setVersion" : 1,
	"ismaster" : true,
	"secondary" : false,
	"primary" : "mongo2:27017", // mongo2로 프라이머리가 변경된 것을 확인할 수 있다.
	"me" : "mongo2:27017",
	"electionId" : ObjectId("7fffffff0000000000000002"),
	"lastWrite" : {
		"opTime" : {
			"ts" : Timestamp(1668469524, 1),
			"t" : NumberLong(2)
		},
		"lastWriteDate" : ISODate("2022-11-14T23:45:24Z"),
		"majorityOpTime" : {
			"ts" : Timestamp(1668469524, 1),
			"t" : NumberLong(2)
		},
		"majorityWriteDate" : ISODate("2022-11-14T23:45:24Z")
	},
	"maxBsonObjectSize" : 16777216,
	"maxMessageSizeBytes" : 48000000,
	"maxWriteBatchSize" : 100000,
	"localTime" : ISODate("2022-11-14T23:45:30.391Z"),
	"logicalSessionTimeoutMinutes" : 30,
	"connectionId" : 45,
	"minWireVersion" : 0,
	"maxWireVersion" : 9,
	"readOnly" : false,
	"ok" : 1,
	"$clusterTime" : {
		"clusterTime" : Timestamp(1668469524, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	},
	"operationTime" : Timestamp(1668469524, 1)
}

```

- 복제 셋의 핵심 개념
  - 클라이언트는 독립 실행형 서버에 보낼 수 있는 모든 작업을 프라이머리 서버에 보낼 수 있다.
    - 읽기, 쓰기, 명령, 인덱스 구축 등등 ..
  - 클라이언트는 세컨더리에 쓰기를 할 수 없다.
  - 기본적으로 클라이언트는 세컨더리로부터 읽을 수 없다. '세컨더리에서 읽어도 괜찮다.' 를 뜻하는 설정을 연결에 명시적으로 설정하면 읽기를 활성화 할 수 있다.



## 복제 셋 구성 변경

- 복제 셋 구성은 언제든지 변경할 수 있고 멤버 추가, 삭제, 변경이 가능하다.

```
rs.add("localhost:27020")
rs.remove("localhost:27020")
```

- 재구성 성공 여부는 셸에서 rs.config()를 실행해 확인할 수 있다.

```
rs.config()
{
	"_id" : "dbrs",
	"version" : 1,
	"term" : 2,
	"protocolVersion" : NumberLong(1),
	"writeConcernMajorityJournalDefault" : true,
	"members" : [
		{
			"_id" : 0,
			"host" : "mongo1:27017",
			"arbiterOnly" : false,
			"buildIndexes" : true,
			"hidden" : false,
			"priority" : 1,
			"tags" : {
				
			},
			"slaveDelay" : NumberLong(0),
			"votes" : 1
		},
		{
			"_id" : 1,
			"host" : "mongo2:27017",
			"arbiterOnly" : false,
			"buildIndexes" : true,
			"hidden" : false,
			"priority" : 1,
			"tags" : {
				
			},
			"slaveDelay" : NumberLong(0),
			"votes" : 1
		},
		{
			"_id" : 2,
			"host" : "mongo3:27017",
			"arbiterOnly" : false,
			"buildIndexes" : true,
			"hidden" : false,
			"priority" : 1,
			"tags" : {
				
			},
			"slaveDelay" : NumberLong(0),
			"votes" : 1
		}
	],
	"settings" : {
		"chainingAllowed" : true,
		"heartbeatIntervalMillis" : 2000,
		"heartbeatTimeoutSecs" : 10,
		"electionTimeoutMillis" : 10000,
		"catchUpTimeoutMillis" : -1,
		"catchUpTakeoverDelayMillis" : 30000,
		"getLastErrorModes" : {
			
		},
		"getLastErrorDefaults" : {
			"w" : 1,
			"wtimeout" : 0
		},
		"replicaSetId" : ObjectId("63718c90eed497ad35e38f60")
	}
}

```

- 구성을 변경할 때마다 version 필드의 값이 증가한다.
- 멤버를 추가하고 제거할 뿐 아니라 이미 존재하는 멤버를 수정할 수도 있다.

```
var config = rs.config()
//members[0] 번째의 host 변경
config.members[0].host = "localhost:27017"
rs.reconfig(config)
```

- rs.reconfig()를 사용하면 어떤 구성이든 변경할 수 있다.



## 복제 셋 설계 방법

- 과반수 개념
  - 프라이머리를 선출하려면 멤버의 과반수 이상이 필요하다
  - 프라이머리는 과반수 이상이어야만 프라이머리 자격을 유지할 수 있다.
  - 쓰기는 과반수 이상에 복제되었을 때 안전해진다.
- 과반수 구하는 식
  - (복제 셋 내 멤버의 수 / 2) + 1 = 과반수

| 복제 셋 내 멤버의 수 | 복제 셋의 과반수 |
| -------------------- | ---------------- |
| 1                    | 1                |
| 2                    | 2                |
| 3                    | 2                |
| 4                    | 3                |
| 5                    | 3                |
| 6                    | 4                |
| 7                    | 4                |

- 과반수는 복제 셋의 구성에 따라 산정되므로 얼마나 많은 멤버가 다운되거나 사용할 수 없는 상태인지는 중요하지 않다.
  - 예를 들어 복제 멤버가 5이고 그 중에 3개가 다운된 상태,  2개는 살아있는 상태일때
    - 살아있는 멤버는 모두 세컨더리다.
    - 만약 둘 중하나가 프라이머리였어도 과반수에 미치지 않는다는 것을 안 순간부터 프라이머리 자격을 내려놓는다.
    - 이용 가능한 복제 셋 멤버가 과반수에 못미치면 모든 멤버는 세컨더리가 된다.
- 남아있는 멤버로 프라이머리를 선출하지 않는 이유
  - 이전과 같이 5개의 복제 셋이 있다고 가정하고 3개는 A 네트워크에 2개는 B 네트워크에 있다고 가정
  - 네트워크 파티션이 생긴 경우 파티션 양쪽에서 프라이머리를 선출하지 않도록 하는 경우가 생길 수 있기 때문에 남아있는 멤버로 프라이머리를 선출하지 않는다.
  - 프라이머리를 선출할 때 과반수 이상을 요구하는 방식은 프라이머리를 여러 개 갖게 되는 상황을 방지하는 깔끔한 해결책이다.
- 프라이머리를 하나만 갖도록 셋을 구성하는 것이 중요하다.
- 권장되는 일반적인 구성
  - 하나의 데이터 센터에 복제 셋의 과반수가 있는 구성
    - 프라이머리 데이터 센터가 있고 그 안에 복제 셋의 프라이머리를 위치시킬 때 적합한 설계
    - 프라이머리 데이터 센터가 정상일 경우는 문제가 없지만 데이터센터가 이용 불가가되면 다른 데이터센터에서는 새로운 프라이머리를 선출하지 못한다.
  - 각 데이터 센터 내 서버의 개수가 동일하고 또 다른 위치에도 과반수가 있는 서버의 구성
    - 양 쪽 데이터 센터 내 서버에서 복제 셋의 과반수를 확인할 수 있으므로 두 데이터 센터의 선호도가 동일할 때? 적합한 설계다.
- 요구사항이 더 복잡하면 다른 구성이 필요할 수도 있지만, 복제 셋이 불리한 조건에서 어떻게 과반수를 획득할 것인지를 명심해야 한다.
- 프라이머리를 하나 이상 갖도록 몽고DB가 지원한다면 모든 복잡성이 해결될 것이라고 생각할 수 있는데 다중 마스터는 그 자체로 복잡성을 수반한다.
  - ex: 쓰기 충돌. 쓰기 충돌은 개발자가 코드화하기 쉬운 모델이 아니다.
  - 따라서 몽고DB는 오직 단일 프라이머리만 지원한다.



### 어떻게 선출하는가? 

- 프라이머리를 선출하는 검사 과정
  1. 요청받은 멤버가 프라이머리에 도달할 수 있는가?
  2. 선출되고자 하는 멤버의 복제 데이터가 최신인가?
  3. 대신 선출돼야 하는 우선순위가 더 높은 멤버는 없는가?
- 몽고DB는 버전 3.2에서 복제 프로토콜 버전 1을 도입했다. 
  - RAFT 합의 프로토콜을 기반으로 아비터, 우선순위, 비투표 멤버, 쓰기 결과 확인 등 몽고DB 특유의 복제 개념을 포함한다.
  - 프로토콜 버전1은 장애 조치 시간 단축 등 새로운 기능의 기반이 되며, 잘못된 프라이머리 상황을 감지하는 데 걸리는 시간을 크게 줄인다.
  - 용어 ID를 사용함으로써 이중 투표를 방지한다.
  - 가장큰 특징은 RAFT와 유사하다.
    - RAFT알고리즘: 비교적 독립적인 하위 문제로 분리되는 합의 알고리즘. 합의는 여러 서버나 프로세스가 값에 동의하는 과정. RAFT는 동일한 일련의 명령이 동일한 일련의 결과를 생성하고 배포 멤버 전체에서 동일한 일련의 상태에 도달하도록 합의를 보장한다. k8s에서도 사용중이다.
- 복제 셋 멤버는 2초마다 서로 다른 하트비트를 보내고 10초 이내에 멤버가 하트비트를 반환하지 않으면 그 멤버를 '접근할 수 없음' 으로 표시한다.
- 선출 알고리즘은 우선순위가 가장 높은 세컨더리가 선출을 호출하도록 최선의 노력을 한다.
  - 우선순위는 특정 멤버에 가중치로 지정할 수 있다.
  - 우선순위가 더 높은 세컨더리가 더 낮은 세컨더리보다 상대적으로 더 빨리 선출되며 이길 가능성도 더 높다.
  - 심지어 우선순위가 더 높은 세컨더리가 있더라도 더 낮은 세컨더리가 잠시 프라이머리로 선출될 수 있으나 복제 셋 멤버들은 우선순위가 가장 높은 멤버가 프라이머리가 될 때까지 계속 선출한다.
- 어떤 멤버가 프라이머리로 선출되려면 복제 데이터가 반드시 최신이어야한다.
  - 복제된 모든 작업은 오름차순 식별에 따라 엄격하게 정렬되므로, 후보는 도달할 수 있는 모든 멤버보다 작업이 늦거나 같아야한다.



## 멤버 구성 옵션

- 특정 멤버가 우선적으로 프라이머리가 되게 하거나 클라이언트에 보이지 않게 해 읽기 요청이 라우팅되지 않도록 할 수 있다.
- 다양한 구성 옵션은 복제 셋 구성의 멤버 서브도큐먼트에 명시할 수 있다.



### 우선순위

- 우선순위는 특정 멤버가 얼마나 프라이머리가 되기를 원하는지 나타내는 지표다.
- 0과 100사이의 값으로 지정할 수 있고 기본값은 1이다. 
- 만약 우선순위가 0이라면 그 멤버는 절대 프라이머리가 될 수 없다. 이러한 멤버를 수동적 멤버라고 한다.
- 모든 멤버의 우선순위가 1인 상황에서 아래와 같이 우선순위가 1.5인 멤버가 새롭게 복제셋에 추가되면 추가된 멤버는 다른 복제 셋 멤버와 같은 최신 데이터로 동기화 된 후에 프라이머리로 승격한다.

```
rs.add({"host": "server-4:27017", "proiorty": "1.5"})
```



### 숨겨진 멤버

- 클라이언트는 숨겨진 멤버에 요청을 라우팅하지 않으며, 숨겨진 멤버는 복제 소스로서 바람직하지 않다.

```
rs.isMaster()
{
  ..
  "host": [
    "server1",
    "server2",
    "server3"
  ],
  ..
}
```

- server3을 숨기려면 hidden 필드를 구성에 추가해야 한다. 멤버가 숨겨지려면 우선순위가 0이어야 하고 프라이머리는 숨길 수 없다.

```
var config = rs.config()
config.members[2].hidden = true
config.members[2].prioirty = 0
rs.reconfig(config)
```

- 다시 isMaster를 실행하면 아래와 같다.

```
rs.isMaster()
{
  ..
  "host": [
    "server1",
    "server2"
  ],
  ..
}
```

- rs.status(), rs.config()는 여전히 멤버를 보여주며 isMaster()에서만 멤버가 사라진다.
- 클라이언트는 복제 셋에 연결할 때 isMaster를 호출하므로 숨겨진 멤버는 읽기 요청을 처리하는 데 사용하지 않는다.
- 멤버를 다시 노출시키려면 hidden옵션을 false로 변경하거나 제거하면 된다.



### 아비터 선출

- 멤버가 2개인 복제 셋은 대부분의 요구사항에서 명확한 단점이 있지만 소규모로 배포하는 사람들은 두 개 이상의 멤버를 갖는것을 관리, 운영, 비용을 고려했을때 별 가치가 없다고 생각한다.
- 몽고DB는 프라이머리 선출에 참여하는 용도로만 쓰이는 아비터라는 특수한 멤버를 지원한다.
- 아비터는 데이터를 가지지 않으며 클라이언트에 의해 사용되지 않는다.
- 일반적으로는 아비터가 없는 배포가 바람직하다.
- 아비터는 mongod 서버의 동작과 아무런 연관이 없기 때문에 사양이 낮은 서버에서 경량화 프로세스로 실행할 수 있다.
- 가능하면 다른 멤버와 분리된 별도의 도메인에서 아비터를 실행함으로써 아비터가 복제 셋에 외부 관점을 갖게하는게 좋다.

```
rs.addArb("server-5:27017")
rs.add({"_id": 4, "host": "server-5:27017", "arbiterOnly" : true})
```

- 아비터는 복제 셋에 추가되고 나면 영원히 아비터다. 변경이 불가능하다.
- 아비터는 큰 클러스터상에서 동점 상황을 없앨 수 있다는 장점이 있다.



`아비터는 최대 하나까지만 사용하라`

- 노드 개수가 홀수면 아비터는 필요하지 않다. 흔히 만약의 경우를 대비해서 여분의 아비터를 추가해야 한다고 오해하는데 이는 낭비다.
- 복제 셋에 멤버가 세개 있을때 프라이머리를 선출하려면 멤버 두개가 필요하다. 아비터를 추가하면 복제 셋 멤버는 네 개가 되고 프라이머리를 선발하는 데 세 개가 필요하다. 따라서 복제 셋은 잠재적으로 덜 안정적인 상태가 되는데 복제 셋의 멤버가 67%가 아니라 75%가 살아있어야 하기 때문이다.
- 결국 아비터를 추가할때는 반드시 멤버가 짝수인 상황에서 홀수가 되는 방향일 때 추가해야 한다.



`아비터 사용의 단점`

- 데이터 노드와 아비터 중 하나를 골라야 한다면 데이터 노드를 선택하자.
- 작은 규모의 복제 셋에서 데이터 노드 대신 아비터를 사용하면 운영 업무가 더 어려워질 수 있다.
  - 일반 멤버가 두 개 일때 하나가 데이터를 복구할 수 없이 완전히 죽어버린다면 새로운 멤버를 추가해야하는데 데이터의 양이 매우 많다면 이는 서버에 상당한 부하를 줄 수 있고 애플리케이션에도 영향을 줄 수 있다.
- 데이터를 보관하는 멤버가 세 개라면 하나가 완전히 죽더라도 숨 쉴 틈이 있다. 따라서 아비터 대신 홀수 개의 일반 멤버를 추가하는 것이 좋다.



### 인덱스 구축

- 때때로 세컨더리는 프라이머리에 존재하는 것과 동일한 인덱스를 갖지 않아도 되고 인덱스가 없어도 된다.
- 세컨더리를 데이터 백업이나 오프라인 배치 작업에만 사용한다면 buildIndexes: false를 멤버 구성에 명시하면 된다. 이 옵션은 세컨더리가 인덱스를 구축하지 않도록 한다.
- buildIndexes: false 은 변경 불가이고 인덱스 비구축 멤버를 다시 인덱스 구축 멤버로 바꾸러면 셋에서 멤버를 제거하고 데이터를 모두 지운 뒤 해당 멤버를 다시 복제셋에 추가해 처음부터 다시 동기화해야 한다.
- 숨겨진 멤버와 마찬가지로 우선순위가 0이어야한다.