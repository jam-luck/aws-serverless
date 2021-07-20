# 서버리스 아키텍처로 구성한 휴가봇/캘린더
![](https://i.imgur.com/fhZ7GN9.png)

사용자는 webhook과 S3내의 정적페이지에 접근하여 연결된 API에 접근할 수 있습니다. API Gateway는 Lambda함수와 연결되어 DynamoDB를 이용한 CRUD 작업을 수행하게 됩니다. 

저는 AWS활용 아키텍처 구축과 API 개발을 담당했습니다.

## 사용서비스
![](https://i.imgur.com/ry5X6v6.png)  

## 기획배경
* 기존 서비스를 사용할 수 없는 환경
* 휴가는 필요한 사람들만 사용하여 평소 사용량이 많지않다
* 팀의 한달간 휴가목록이 많지 않다
* 삭제가 가능했으면 좋겠다
* 다른 팀원들의 휴가도 확인했으면 좋겠다

## 장점
* 휴가는 필요한 사람들만 사용하여 평소 사용량이 많지않다
-> 서버리스, 온디맨드로 구성하여 서버없이 요청한만큼만 과금되어 비용효율적이다.
* 팀의 한달간 휴가목록이 많지 않다
-> DynamoDB의 한번에 불러올 데이터량이 한정되어 있지만 휴가 목록이 적어 충분히 사용가능
* 삭제가 가능했으면 좋겠다
-> API DELETE 지원
* 다른 팀원들의 휴가도 확인했으면 좋겠다
-> DB검색시 글로벌 보조인덱스를 통해 팀원의 휴가 검색하도록 설계
## API Gateway
![](https://i.imgur.com/gKYkQ6P.png)
리소스에 따라 필요한 CRUD를 Restful하게 구성하였습니다.

1. 리소스생성
2. 메서드생성
3. 메서드와 Lambda 연결
4. CORS 허용(S3내의 정적페이지에서 리소스사용하기 위함)
5. 람다 연결 후 API 배포

## AWS Lambda
람다함수는 Node.js로 작성했으며 DynamoDB를 사용합니다.  

API POST,GET,DELETE메서드에 각각 람다를 연결하였고. POST, DELETE부분은 제가 작성하였고 GET은 팀원분이 작성했습니다.  

CORS 이슈를 해결하기 위해 헤더에 해당내용을 포함하여 리턴해주었습니다.

```javascript=
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient();
const CORS_HEADER = {
  "Access-Control-Allow-Origin": "*", // Required for CORS support to work
  "Access-Control-Allow-Credentials": true, // Required for cookies, authorization headers with HTTPS
};
exports.handler = async (event) => {
    let response;
    console.log(event);
    var body=JSON.parse(event.body);
    console.log(body);
    let params = {
        Item: {
            date: body.startDate,
            empid: body.empId,
            name: body.name,
            type: body.type,
            teamid: body.teamId,
            startdate: body.startDate,
            enddate: body.endDate,
            content: body.content,
            monthteamid: body.startDate.substr(0,7)+body.teamId //연-월+팀id 검색용 보조 인덱스
        },
        TableName: "vacationDB",
    };
    try {
        await dynamodb.put(params).promise();
    } catch (e) {
        console.log(e);
        response = {
            statusCode: 500,
            body: JSON.stringify("이번달 휴가 저장중 에러가 발생하였습니다: " + e),
            headers: CORS_HEADER,
        };
        return response;
    }
    // 휴가가 두달에 걸쳐져 있는 경우 다음달에 휴가내용 추가 저장
    if(body.startDate.substr(5,2) != body.endDate.substr(5,2)){ 
        params.Item.date = body.endDate;
        params.Item.monthteamid = body.endDate.substr(0,7)+body.teamId;
        try {
            await dynamodb.put(params).promise();
        } catch (e) {
            console.log(e);
            response = {
                statusCode: 500,
                body: JSON.stringify("다음달 휴가 저장중 에러가 발생하였습니다: " + e),
                headers: CORS_HEADER,
            };
            return response;
        }
    }
    response = {
        statusCode: 200,
        body: JSON.stringify("데이터가 성공적으로 저장되었습니다."),
        headers: CORS_HEADER,
    };
    return response;
};
```
post 부분 샘플코드.
### Lambda - dynamoDB정책
![](https://i.imgur.com/O9pDkgA.png)
기본적으로 람다는 DynamoDB 접근 권한이 없기 때문에 정책을 생성하여 Lambda에 권한을 부여했습니다.

## DynamoDB
![](https://i.imgur.com/UjTbmGp.png)

DB를 설계할 때 파티션키+정렬키로 유일성을 확보하도록했고 글로벌 보조인덱스를 두어서 검색에 사용할 수 있도록 하였습니다. 또한 온디맨드로 설정해서 사용량만큼 요금을 내어 비용상 유리하게 설정했습니다.

## S3
![](https://i.imgur.com/d109qzU.png)

S3는 프론트엔드를 담당한 팀원들이 빌드한 정적페이지를 호스팅하고 퍼블릭 엑세스가 가능하도록 설정하여 사용자들이 정적페이지에 접근하도록 하였습니다.  

## 이슈
* CORS 이슈
* 휴가가 두달에 걸쳐있을 경우 처리하는 방법
* JSON 포멧
* 외부 접근관련 보안이슈
* 기타.. S3, DynamoDB, API Gateway, Lambda 사용이 거의 처음이여서 구축에 대한 모든과정이 이슈였습니다. 


## 결과
![](https://i.imgur.com/7rKbsSm.png)  
![](https://i.imgur.com/AT8FF40.png)
봇과 채팅채널을 통해서 휴가를 등록, 삭제, 조회할 수 있습니다.  
![](https://i.imgur.com/kpcfUUS.png)
![](https://i.imgur.com/1PEV4Xa.png)

캘린더 소스코드는 팀원 신다정님 깃에 업로드 되어있습니다.  
https://github.com/ShinDajeong/calendar 
