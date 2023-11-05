## 1. 디바이스 목록 조회 REST API 구축하기
다음과 같은 API 요청과 API 응답을 가지는 REST API를 Lambda 함수와 API Gateway를 이용하여 구축해본다.

- API 요청
	
	```	
	GET /devices
	```
	
- API 응답

	```json
	{
		"things": [ 
		     { 
		    	"thingName": "string",
		      	"thingArn": "string"
		     }, 
		     ...
		   ]
	}
	```

--
### 단계1: AWS Toolkit을 통해 Lambda 함수 생성
1. 다음 정보를 바탕으로 AWS Lambda 프로젝트를 JetBrains용 AWS Toolkit을 이용하여 생성한다.
	- **Project name**: *ListingDeviceLambda*
	- **Rumtime**:*java11*
 	- **SDK**: 11버전의 SDK를 선택하거나 없으면 다운로드한 후 선택 
 	
	
2. 생성된 *ListingDeviceLambda*의 **build.gradle** 파일을 열고, 다음 의존성을 추가하고, **변경사항을 반영**합니다.

	```
	dependencies {
		...
       implementation platform('com.amazonaws:aws-java-sdk-bom:1.12.529')
       implementation 'com.amazonaws:aws-java-sdk-iot'
       ...
   }
   ```

3. **src/main/java/helloworld/App.java** 파일을 다음 코드로 바꿉니다.
	
	```java
	package helloworld;
	import java.util.List;
	import com.amazonaws.services.iot.AWSIot;
	import com.amazonaws.services.iot.AWSIotClientBuilder;
	import com.amazonaws.services.iot.model.ListThingsRequest;
	import com.amazonaws.services.iot.model.ListThingsResult;
	import com.amazonaws.services.iot.model.ThingAttribute;
	import com.amazonaws.services.lambda.runtime.Context;
	import com.amazonaws.services.lambda.runtime.RequestHandler;
	import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyResponseEvent;
	import java.util.Map;
	import java.util.HashMap;
	
	public class App implements RequestHandler<Object, APIGatewayProxyResponseEvent> {
	
	    @Override
	    public APIGatewayProxyResponseEvent handleRequest(Object input, Context context) {
	
	        // AWSIot 객체를 얻는다.
	        AWSIot iot = AWSIotClientBuilder.standard().build();
	
	        // ListThingsRequest 객체 설정.
	        ListThingsRequest listThingsRequest = new ListThingsRequest();
	
	        // listThings 메소드 호출하여 결과 얻음.
	        ListThingsResult result = iot.listThings(listThingsRequest);
	
	
	        Map<String, String> headers = new HashMap<>();
	        headers.put("Content-Type", "application/json");
	        headers.put("X-Custom-Header", "application/json");
	
	        APIGatewayProxyResponseEvent response = new APIGatewayProxyResponseEvent()
	                .withHeaders(headers);
	
	        // result 객체로부터 API 응답모델 문자열 생성하여 반
	        return response.withStatusCode(200).withBody(getResultStr(result));
	    }
	
	    /**
	     * ListThingsResult 객체인 result로 부터 ThingName과 ThingArn을 얻어서 Json문자 형식의
	     * 응답모델을 만들어 반환한다.
	     * {
	     * 	"things": [
	     *	     {
	     *			"thingName": "string",
	     *	      	"thingArn": "string"
	     *	     },
	     *		 ...
	     *	   ]
	     * }
	     */
	    private String getResultStr(ListThingsResult result) {
	        List<ThingAttribute> things = result.getThings();
	
	        String resultString = "{ \"things\": [";
	        for (int i =0; i<things.size(); i++) {
	            if (i!=0)
	                resultString +=",";
	            resultString += String.format("{\"thingName\":\"%s\", \"thingArn\":\"%s\"}",
	                    things.get(i).getThingName(),
	                    things.get(i).getThingArn());
	
	        }
	        resultString += "]}";
	        return resultString;
	    }
	
	}
	```
	
4. **src/test/java/helloworld/AppTest.java** 파일의 코드를 주석처리하여 컴파일 오류를 제거시킨다.

--	
### 단계2: Lambda 함수의 로컬 테스트

작성된 Lambda함수가 정상적으로 동작하는 지를 테스트해 보기 위해서 다음 절차를 수행합니다.

- [**필수**] Docker 프로세스가 실행된 상태이어야 함 
  
1. IntelliJ IDEA IDE의 화면 상단 타이틀 바에서 "[Local] HelloWorldFunction" 옆의 **연두색 실행 버튼 (삼각형)을 클릭**
  
2. [**Edit Configuration**] 다이얼로그 화면에서 **Text -- Event Templates --** 부분의 드롭다운 메뉴 중에서 *API Gateway AWS Proxy*를 선택 
  
  - **Run** 클릭
    

3. **Console** 창에 다음과 같은 형식의 메시지가 마지막에 출력되는 지 확인합니다. (본인의 aws 계정에 생성된 사물 목록이 Json 형식으로 반환됨)
   
   ```
   ...	
   "{ \"things\": [{\"thingName\":\"MyMKRWiFi1010\", \"thingArn\":\"arn:aws:iot:ap-northeast-2:884579964612:thing/MyMKRWiFi1010\"}]}"
   
   ```
--	
### 단계3: Lambda 함수의 배포

- **ListingDeviceLambda** 프로젝트 탐색창에서 **template.yaml**을 찾아서 선택하고, 선택된 상태에서 오른쪽 마우스 클릭하여 **SyncServerless Application (formerly Deploy)** 메뉴를 선택
  
  - [**Confirm development stack**] 다이얼로그 화면에서 **Confirm** 선택
  
  - [**SyncServerless Application (formerly Deploy)**] 다이얼로그 화면에서, **Create Stack**에 적절한 이름(예, *ListingDeviceLambda*)을 입력 하고, **CloudFormation Capabilities:** 에서 **IAM** 체크박스를 선택한 후, **Sync** 클릭
    
    - [**참고**] 한참 동안 진행이 안되면 현재 스텝을 한번더 수행해 본다.  
  
  - 콘솔 창에 다음 결과가 맨 마지막 줄에 출력되는 지를 확인
    
    ```
    ...
	Stack creation succeeded. Sync infra completed.


	Process finished with exit code 0
    ``` 
--	
### 단계4: Lambda 함수의 원격 테스트
- AWS Lambda함수가 다른 AWS 서비스 (예, DynamoDB)를 사용하기 위해서는 필요한 권한이 Lambda함수에 연결된 **실행 역할 정책(Execution Role Polcity)**에  포함되어 있어야 합니다.
  
  - **실행 역할 정책 (Execution Role Polcity)**은 Lambda 함수가 실행되는 동안에만 사용되며, **AWS Identity and Access Management (IAM) 역할**과 연관됩니다. 가령, Lambda 함수가 IoT의 사물 목록을 조회할 권한이 필요한 경우, 실행 역할 정책은 **AWSIoTDataAcess** 권한을 가진 정책이 연결되어 있어야 합니다.
  
- **람다함수의 실행 역할 정책** 업데이트
  
  - **AWS Lambda 콘솔**에서 **함수** 페이지를 연다.
	- 나열된 함수 목록 중에서 *ListDeviceLambda-…* 함수를 선택한다.
	- **구성** 탭에서 **권한** 메뉴를 선택하면, **실행역할**을 찾을 수 있다.
  - **역할 이름**을 클릭하면, **권한 정책** 파트에서 해당 역할에 설정된 정책들을 확인할 수 있다.
  - 만약, 권한정책에 **AWSIoTFullAccess** 정책이 포함되어 있지 않다면, **권한 추가>>정책연결** 메뉴를 클릭하여 **AWSIoTFullAccess** 정책을 검색하고 선택한다음 **권한추가** 버튼을 클릭한다.
 - 원격 테스트를 위해서 **AWS Toolkit** 창의 탐색기에서  **Lambda**를 확장하여 *ListDeviceLambda-HelloWorldFunction-XXX*선택하고, 오른쪽 마우스 클릭하여 **Run '[Remote] HelloServer...'**메뉴를 선택
	- 로컬 테스트에서 사용했던 동일한 입력 값으로 테스트를 진행하고, 동일한 결과가 나오는지를 확인한다.
	
--	
### 단계5 API Gateway 콘솔에서 REST API 생성

1. [API Gateway 콘솔](https://ap-northeast-2.console.aws.amazon.com/apigateway/)로 이동합니다.
2. **API 생성**을 선택합니다.
3. **API 유형 선택**에서 *REST API*의 **구축**를 클릭합니다.
4. **REST API 생성**에서 *새 API*를 선택합니다.
5. **이름 및 설명** 설정에서 다음과 같이 합니다.
	- **API 이름**에 *my-device-api*를 입력합니다.
	- 필요한 경우 **설명** 필드에 설명을 입력합니다. 설명을 입력하지 않으려면 비워 둡니다.
	- **엔드포인트 유형** 설정을 *지역*으로 그대로 둡니다.
6. **API 생성(Create API)**을 선택합니다.
7. **리소스** 아래에 **/** 이외에는 아무 것도 보이지 않을 것입니다. 이는 API의 기본 경로 URL에 해당하는 루트 수준 리소스입니다.
8. **리소스** 아래에서 **/**를 선택한 후, **리소스 생성**을 클릭합니다.
10. **리소스** 이름에 *devices*를 입력합니다. 
11. **리소스 생성**을 선택합니다.
12. **메서드** 섹션에서 **메소드 생성**을 클릭합니다.
13. **메서드 유형** 드롭다운 메뉴에서 **GET**을 선택합니다.
14. **통합 유형**에서 *Lambda 함수*를 선택합니다.
15. **Lambda 함수**에서 Lambda 함수를 생성한 리전을 선택한 후 드롭다운 메뉴에서 *ListingDeviceLambda-HelloWorldFunction*을 선택합니다.  **메서드 생성**을 클릭합니다.
16. **테스트**를 클릭하고, 아무 입력 없이 **테스트**버튼을 클릭하여 다음과 같은 결과가 나오는 지 확인합니다.

	```
	/devices - GET 메서드 테스트 결과
	요청
	/devices
	지연 시간
	6399
	상태
	200
	응답 본문
	{"statusCode":200,"headers":{"X-Custom-Header":"application/json","Content-Type":"application/json"},"body":"{ \"things\": [{\"thingName\":\"MyMKRWiFi1010\", \"thingArn\":\"arn:aws:iot:ap-northeast-2:884579964612:thing/MyMKRWiFi1010\"}]}"}
	응답 헤더
	{
	  "Content-Type": "application/json",
	  "X-Amzn-Trace-Id": "Root=1-653e463c-0518263d2fe5ca0ab5d5bf9b;Sampled=0;lineage=ff092584:0"
	}
	...
	```
	
--
### 단계6: CORS 활성환 및 API Gateway 콘솔에서 REST API 배포


JavaScript는 **Cross-Origin Resource Sharing (CORS)** 요청을 기본적으로 제한합니다. 즉, JavaScript 코드가 동일 서버 내의 리소스를 접근하는 것은 허용하지만, 다른 서버의 리소스를 사용하고자 하는 경우에는 CORS 헤더 정보가 포함되어 있어야 합니다. 

- 더 자세한 정보는 https://developer.mozilla.org/ko/docs/Web/HTTP/Access_control_CORS 참조

**REST API 리소스에 대해 CORS 지원 활성화**

1. 리소스에서 **/devices**를 선택하고, **CORS 활성화**를 클릭합니다.
3. 게이트웨이 응답과 Access-Control-Allow-Methods의 모든 체크박스를 선택합니다.
4. **저장**를 선택합니다.

2단계를 완료하면 API를 생성했지만 아직 실제로 사용할 수는 없습니다. 배포해야 하기 때문입니다.

1. **베포** 버튼을 클릭합니다.
2. **배포 스테이지** 드롭다운 메뉴에서 **[새 스테이지]**를 선택합니다.
3. **스테이지 이름**에 *prod*를 입력합니다.
4. **배포**을 선택합니다.


--
### 단계7: REST API 테스트
1. **prod 스테이지 편집기**의 맨 위에 있는 **호출 URL**을 적어 둡니다.
2. 웹 브라우저 주소창에 *"호출 URL/devices"*을 입력한 후 엔터를 쳐 봅니다.
	- 이번 REST API는 GET 메소드만을 이용한 것이므로, 웹 브라우저에서도 테스트가 가능하지만, 일반적으로 API 테스트는 [cURL](https://curl.haxx.se/) 또는 [POSTMAN](https://www.getpostman.com/) 등의 도구를 사용합니다
	

3. [2.1](api-gateway.html##2.1)절에서 정의한 응답모델과 동일한 형식의 JSon 문자열이 반환된 것을 확인할 수 있습니다.

### 단계8: REST API 활용한 JavaScript 기반 웹 프로그래밍
<!--
- JavaScript는 **Cross-Origin Resource Sharing (CORS)** 요청을 기본적으로 제한합니다. 즉, JavaScript 코드가 동일 서버 내의 리소스를 접근하는 것은 허용하지만, 다른 서버의 리소스를 사용하고자 하는 경우에는 CORS 헤더 정보가 포함되어 있어야 합니다. 
	- 더 자세한 정보는 https://developer.mozilla.org/ko/docs/Web/HTTP/Access_control_CORS 참조
- 따라서, [API Gateway 콘솔을 사용하여 리소스에서 CORS 활성화
](https://docs.aws.amazon.com/ko_kr/apigateway/latest/developerguide/how-to-cors.html#how-to-cors-console) 절차에 따라서 **CORS**를 활성화 해야 합니다.
	1. https://console.aws.amazon.com/apigateway 에서 API Gateway 콘솔에 로그인합니다.
	2. API 목록에서 API를 선택합니다.
	3. 리소스에서 리소스를 선택합니다. 그렇게 하면 리소스 상의 모든 메서드에 대해 CORS가 활성화됩니다.
	4. 작업 드롭다운 메뉴에서 **CORS 활성화(Enable CORS)**를 선택합니다.
	5. **CORS 활성화 및 기존의 CORS 헤더 대체(Enable CORS and replace existing CORS headers)**를 선택합니다. 	
	6. **메소드 변경사항 확인** 창에서 **예, 기존 값을 대체하겠습니다.** 를 선택합니다.
	7. **작업** 드롭다운 메뉴에서 **Deploy API(API 배포)**를 선택합니다.
	8. **배포 스테이지** 드롭다운 메뉴에서 **prod**를 선택합니다.
	9. **배포**을 선택합니다.
-->
- **나의 디바이스 목록 조회**
	- **[list\_devices.html](release/list_devices.html)** 시작 html 페이지
	
		```html
		<!DOCTYPE html>
		<html lang="en">
		    <head>
		        <meta charset="UTF-8">
		        <title>AWS Open API Sample</title>
		
		        <!-- JQuery 라이브러리 설정 -->
		        <script src="https://code.jquery.com/jquery-3.4.1.min.js" integrity="sha256-CSXorXvZcTkaix6Yvo6HppcZGetbYMGWSFlBw8HfCJo=" crossorigin="anonymous"></script>
		
		        <!-- 디바이스 조회 자바스크립트 로딩-->
		        <script src="list_devices.js"></script>
		    </head>
		    <body>
		        <h3>My AWS API Sample</h3>
		
		
		        <h4> 나의 디바이스 목록 <input type="button" value="조회" onclick="Start();" />
		        </h4>
		
		        <table id="mytable">
		            <thead style="background-color:lightgrey">
		                <th>디바이스 명</th>
		                <th>디바이스 ARN</th>     
		            </thead>
		            <tbody> </tbody>
		        </table>
		  
		        <div id="result">No Data</div>
		    </body>
		</html>	
		```

	- **[list\_devices.js](release/list_devices.js)**: JQuery 기반 Javascript 코드 (**API\_URI** 변수를 API gateway에서 생성한 URI로 수정해야 합니다.)

		```javascript
		// API 시작
		function Start() {
		    invokeAPI();
		    emptyTable();
		}
		
		var invokeAPI = function() {
		
		    // 디바이스 조회 URI
		    // prod 스테이지 편집기의 맨 위에 있는 "호출 URL/devices"로 대체해야 함
		    var API_URI = 'https://XXXXXXXXXX.execute-api.ap-northeast-2.amazonaws.com/prod/devices'; 		        
		    $.ajax(API_URI, {
		        method: 'GET',
		        contentType: "application/json",
		        
		        
		        success: function (data, status, xhr) {
		
		            var result = JSON.parse(data.body);
		            setDataList(result.things);  // 성공시, 데이터 출력을 위한 함수 호출
		            console.log(data);
		        },
		        error: function(xhr,status,e){
		          //  document.getElementById("result").innerHTML="Error";
		            alert("error");
		        }
		    });
		};
		
		// 테이블 데이터 삭제
		var emptyTable = function() {
		    $( '#mytable > tbody').empty();
		    document.getElementById("result").innerHTML="조회 중입니다.";
		}
		
		// 데이터 출력을 위한 함수
		var setDataList = function(data){
		
		    $( '#mytable > tbody').empty();
		    data.forEach(function(v){
		
		        var tr1 = document.createElement("tr");
		        var td1 = document.createElement("td");
		        var td2 = document.createElement("td");
		        td1.innerText = v.thingName;
		        td2.innerText = v.thingArn;
		        tr1.appendChild(td1);
		        tr1.appendChild(td2);
		        $("table").append(tr1);
		    })
		
		    if(data.length>0){
		            // 디바이스 목록 조회결과가 있는 경우 데이터가 없습니다 메시지 삭제
		        document.getElementById("result").innerHTML="";
		    } else if (data.length ==0) {
		        document.getElementById("result").innerHTML="No Data";
		    }
		}
		```	
- **실행 화면**
	- 초기화면 

		![](figures/list-run1.png)
		
	- **조회** 버튼을 클릭한 후
	
		![](figures/list-run2.png)

