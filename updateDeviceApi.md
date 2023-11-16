## 3. 디바이스 상태 변경 REST API 구축하기
다음과 같은 API 요청과 API 응답을 가지는 REST API를 Lambda 함수와 API Gateway를 이용하여 구축해본다.

- API 요청
	
	```		
	PUT /devices/{deviceID}
	```	
	- payload 
	
		```json
		{ 
			"tags" : [
				{
					"attrName": "temperature",
					"attrValue": "27.0"
				},
				{
					"attrName": "LED",
					"attrValue": "OFF"
				}
			]
		}
		```		
		
- 응답 모델
	- [UpdateThingShadow](https://docs.aws.amazon.com/ko_kr/iot/latest/developerguide/API_UpdateThingShadow.html)의 [응답 상태 문서](https://docs.aws.amazon.com/ko_kr/iot/latest/developerguide/device-shadow-document-syntax.html#device-shadow-example-response-json) 

--
### 단계1: AWS Toolkit을 통해 Lambda 함수 생성
1. 다음 정보를 바탕으로 AWS Lambda 프로젝트를 JetBrains용 AWS Toolkit을 이용하여 생성한다.
	- **Project name**: *UpdateDeviceLambda*
	- **Rumtime**:*java11*
 	- **SDK**: 11버전의 SDK를 선택하거나 없으면 다운로드한 후 선택 
 	
	
2. 생성된 * GetDeviceLambda*의 **build.gradle** 파일을 열고, 다음 의존성을 추가하고, **변경사항을 반영**합니다.


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
	
	import java.nio.ByteBuffer;
	import java.util.ArrayList;
	import com.amazonaws.services.lambda.runtime.Context;
	import com.amazonaws.services.lambda.runtime.RequestHandler;
	import com.amazonaws.services.iotdata.AWSIotData;
	import com.amazonaws.services.iotdata.AWSIotDataClientBuilder;
	import com.amazonaws.services.iotdata.model.UpdateThingShadowRequest;
	import com.amazonaws.services.iotdata.model.UpdateThingShadowResult;
	import com.fasterxml.jackson.annotation.JsonCreator;
	
	/**
	 * Handler for requests to Lambda function.
	 */
	public class App implements RequestHandler<Event, String> {
	
	    public String handleRequest(final Event event, final Context context) {
	        AWSIotData iotData = AWSIotDataClientBuilder.standard().build();
	
	        String payload = getPayload(event.tags);
	
	        UpdateThingShadowRequest updateThingShadowRequest  =
	                new UpdateThingShadowRequest()
	                        .withThingName(event.device)
	                        .withPayload(ByteBuffer.wrap(payload.getBytes()));
	
	        UpdateThingShadowResult result = iotData.updateThingShadow(updateThingShadowRequest);
	        byte[] bytes = new byte[result.getPayload().remaining()];
	        result.getPayload().get(bytes);
	        String output = new String(bytes);
	
	        return output;
	    }
	
	    private String getPayload(ArrayList<Tag> tags) {
	        String tagstr = "";
	        for (int i=0; i < tags.size(); i++) {
	            if (i !=  0) tagstr += ", ";
	            tagstr += String.format("\"%s\" : \"%s\"", tags.get(i).tagName, tags.get(i).tagValue);
	        }
	        return String.format("{ \"state\": { \"desired\": { %s } } }", tagstr);
	    }
	}
	
	class Event {
	    public String device;
	    public ArrayList<Tag> tags;
	
	    public Event() {
	        tags = new ArrayList<Tag>();
	    }
	}
	
	class Tag {
	    public String tagName;
	    public String tagValue;
	
	    @JsonCreator
	    public Tag() {
	    }
	
	    public Tag(String n, String v) {
	        tagName = n;
	        tagValue = v;
	    }
	}
	
	```
	
4. **src/test/java/helloworld/AppTest.java** 파일의 코드를 주석 처리한다.

--	
### 단계2: Lambda 함수의 로컬 테스트

작성된 Lambda함수가 정상적으로 동작하는 지를 테스트해 보기 위해서 다음 절차를 수행합니다.

- [**필수**] Docker 프로세스가 실행된 상태이어야 함 
  
1. IntelliJ IDEA IDE의 화면 상단 타이틀 바에서 "[Local] HelloWorldFunction" 옆의 **연두색 실행 버튼 (삼각형)을 클릭**
  
2. [**Edit Configuration**] 다이얼로그 화면에서 **Text -- Event Templates --** 부분의 드롭다운 메뉴 중에서 *API Gateway AWS Proxy*를 선택하고, 다음 입력 문자열을 입력한다.
	- 조회할 사물의 이름이 *MyMKRWiFi1010*인 경우를 가정
	-  device 속성: 변경할 사물의 이름
	- tags 속성: 변경할 사물의 태그 객체 배열 (이름과 값으로 정의됨) 

		```JSON
		{
			"device": "MyMKRWiFi1010",
			"tags" : [
				{
					"tagName": "temperature",
					"tagValue": "25.2"
				},
				{
					"tagName": "LED",
					"tagValue": "OFF"
				}	
			]
		}
		```
  
  - **Run** 클릭
    

3. **Console** 창에 다음과 같은 형식의 메시지가 마지막에 출력되는 지 확인합니다. (본인의 aws 계정에 생성된 사물의 상태가 Json 형식으로 반환됨)
   
   ```
   ...	
SSTART RequestId: e49a9f7e-bf5d-415a-b72d-9e5754830e79 Version: $LATEST
Picked up JAVA_TOOL_OPTIONS: -XX:+TieredCompilation -XX:TieredStopAtLevel=1
END RequestId: e49a9f7e-bf5d-415a-b72d-9e5754830e79
REPORT RequestId: e49a9f7e-bf5d-415a-b72d-9e5754830e79	Init Duration: 0.82 ms	Duration: 12795.96 ms	Billed Duration: 12796 ms	Memory Size: 512 MB	Max Memory Used: 512 MB	
"{\"state\":{\"desired\":{\"temperature\":\"25.2\",\"LED\":\"OFF\"}},\"metadata\":{\"desired\":{\"temperature\":{\"timestamp\":1700158887},\"LED\":{\"timestamp\":1700158887}}},\"version\":1147,\"timestamp\":1700158887}"
   
   ```
--	
### 단계3: Lambda 함수의 배포

- **GetDeviceLambda** 프로젝트 탐색창에서 **template.yaml**을 찾아서 선택하고, 선택된 상태에서 오른쪽 마우스 클릭하여 **SyncServerless Application (formerly Deploy)** 메뉴를 선택
  
  - [**Confirm development stack**] 다이얼로그 화면에서 **Confirm** 선택
  
  - [**SyncServerless Application (formerly Deploy)**] 다이얼로그 화면에서, **Create Stack**에 적절한 이름(예, *UpdateDeviceLambda*)을 입력 하고, S3 Bucket 중에 하나를 선택(S3 Bucket이 없으면 **Create** 버튼을 눌러 생성 후 선택)하고, **CloudFormation Capabilities:** 에서 **IAM** 체크박스를 선택한 후, **Sync** 클릭
    
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
	- 나열된 함수 목록 중에서 *UpdateDeviceLambda-…* 함수를 선택한다.
	- **구성** 탭에서 **권한** 메뉴를 선택하면, **실행역할**을 찾을 수 있다.
  - **역할 이름**을 클릭하면, **권한 정책** 파트에서 해당 역할에 설정된 정책들을 확인할 수 있다.
  - 만약, 권한정책에 **AWSIoTFullAccess** 정책이 포함되어 있지 않다면, **권한 추가>>정책연결** 메뉴를 클릭하여 **AWSIoTFullAccess** 정책을 검색하고 선택한다음 **권한추가** 버튼을 클릭한다.
 - 원격 테스트를 위해서 **AWS Toolkit** 창의 탐색기에서  **Lambda**를 확장하여 *UpdateDeviceLambda-HelloWorldFunction-XXX*선택하고, 오른쪽 마우스 클릭하여 **Run '[Remote] HelloServer...'**메뉴를 선택
	- 로컬 테스트에서 사용했던 동일한 입력 값으로 테스트를 진행하고, 동일한 결과가 나오는지를 확인한다.
	
--	
### 단계5 API Gateway 콘솔에서 REST API 생성

1. [API Gateway 콘솔](https://ap-northeast-2.console.aws.amazon.com/apigateway/)로 이동합니다.
2.이전에 생성한 *my-device-api*를 선택합니다.
3. 리소스 이름(**/{device}**)을 선택합니다. 
4. **메서드** 섹션에서 **메소드 생성**을 클릭합니다.
5. **메서드 유형** 드롭다운 메뉴에서 **PUT**을 선택합니다.
6. **통합 유형**에서 *Lambda 함수*를 선택합니다.
7. **Lambda 함수**에서 Lambda 함수를 생성한 리전을 선택한 후 드롭다운 메뉴에서 *UpdateDeviceLambda-HelloWorldFunction*을 선택합니다.  **메서드 생성**을 클릭합니다.

다음 단계는 API Gateway를 통해 들어오는 클라이언트의 입력을 Lambda 함수에 전달하기 위해서 클라이언트의 입력을 Lambda 함수의 입력으로 매핑하는 과정에 대해서 진행합니다.
	- API Gatway에서 **모델**은 클라이언트의 입력 데이터 구조를 [JSON 스키마 draft 4](https://tools.ietf.org/html/draft-zyp-json-schema-04)를 사용하여 정의한 것으로서, 이를 이용하여 API Gateway가 클라이언트 입력에 대한 검사 및 SDK 생성에 사용됩니다.
	
1. 화면 왼쪽 탐색창에서 **모델**을  선택한 다음 **모델 생성**을 클릭합니다.
2. 모델 이름에 *UpdateDeviceInput*을 입력합니다.
3. 콘텐츠 유형에 *application/json*을 입력합니다.
4. Model description(모델 설명)은 비워 둡니다.
5. 다음 스키마 정의를 Model schema(모델 스키마) 편집기에 복사합니다.
			
	```
	{
		 "$schema": "http://json-schema.org/draft-04/schema#",
		 "title": "UpdateDeviceInput",
	 	 "type" : "object",
		 "properties" : {
		 	"tags" : {
		 		"type": "array",
				"items": {
			              "type": "object",
			              "properties" : {
			                	"tagName" : { "type" : "string"},
			                	"tagValue" : { "type" : "string"}
		        		}
				}
			}
		}
	}
	```

6. /{device} PUT 메서드를 선택하고 **통합 요청(Integration Request)**을 선택하여 본문 매핑 템플릿을 설정합니다.
	- 화면의 하단의 **템플릿 생성**을 클릭한다.
	- **콘텐츠 유형** 드롭다운 메뉴에서 *application/json*을 입력
	-  **탬플릿 생성** 드롭다운 메뉴에서 *UpdateDeviceInput*을 선택하고, 템플릿 본문에 다음을 입력합니다.

		```
		#set($inputRoot = $input.path('$'))
		{
		    "device": "$input.params('device')",
		    "tags" : [
		    ##TODO: Update this foreach loop to reference array from input json
		        #foreach($elem in $inputRoot.tags)
		        {
		            "tagName" : "$elem.tagName",
		            "tagValue" : "$elem.tagValue"
		        } 
		        #if($foreach.hasNext),#end
		        #end
		    ]
		}
		```
		
	- **탬플릿 생성**을 선택합니다.
21. **/devices/{device} – PUT – 메소드 실행** 창으로 이동하여, **테스트** 탭을 클릭합니다.
22. device 경로에 본인이 만든 사물 이름(예, *MyMKRWiFi1010*)을 입력합니다. 
23. **요청 본문**에 아래와 같은 내용을 입력합니다.

	```
	{
		"tags" : [
				{
					"tagName": "temperature",
					"tagValue": "25.2"
				},
				{
					"tagName": "LED",
					"tagValue": "OFF"
				}	
		]
	}
	```

16. **테스트**를 클릭하고, 다음과 같은 결과가 나오는 지 확인합니다.
	
	```
/devices/{device} - PUT 메서드 테스트 결과
요청
/devices/MyMKRWiFi1010
지연 시간
5661
상태
200
응답 본문
"{\"state\":{\"desired\":{\"temperature\":\"25.2\",\"LED\":\"ON\"}},\"metadata\":{\"desired\":{\"temperature\":{\"timestamp\":1699077493},\"LED\":{\"timestamp\":1699077493}}},\"version\":1120,\"timestamp\":1699077493}"
	...
	```
	
--
### 단계6: CORS 활성화 및 API Gateway 콘솔에서 REST API 배포


JavaScript는 **Cross-Origin Resource Sharing (CORS)** 요청을 기본적으로 제한합니다. 즉, JavaScript 코드가 동일 서버 내의 리소스를 접근하는 것은 허용하지만, 다른 서버의 리소스를 사용하고자 하는 경우에는 CORS 헤더 정보가 포함되어 있어야 합니다. 

- 더 자세한 정보는 https://developer.mozilla.org/ko/docs/Web/HTTP/Access_control_CORS 참조

**REST API 리소스에 대해 CORS 지원 활성화**

1. 리소스에서 **/devices/{device}**를 선택하고, **CORS 활성화**를 클릭합니다.
3. 게이트웨이 응답과 Access-Control-Allow-Methods의 모든 체크박스를 선택합니다.
4. **저장**를 선택합니다.


지금까지 API를 생성했지만 아직 실제로 사용할 수는 없습니다. 배포해야 하기 때문입니다.

1. 화면 상단의 **API베포** 버튼을 클릭합니다.
2. **배포 스테이지** 드롭다운 메뉴에서 이전에 생성한 *prod*를 선택합니다.
4. **배포**을 클릭합니다.
5. **URL 호출** 에 표시된 URL을 복사합니다.


--
### 단계7: REST API 테스트
1. **prod 스테이지 편집기**의 맨 위에 있는 **호출URL**을 적어 둡니다.
2. [POSTMAN](https://www.getpostman.com/) 등의 도구를 사용하여 테스트 해 봅니다.
	- 사용절차
		1. 포스트맨 회원가입 후 로그인
		2. **Workspaces** > **My Workspace** 선택
		3. **+** 버튼을 눌러 새로운 요청 생성
			1. 요청 메소드로 **PUT** 선택
			2.  **호출URL**뒤에 */devices/MyMKRWiFI1010*을 덧붙인 URL을 입력
			3. **Body** 탭을 선택하고, **raw**와 **JSON**을 선택한 후, 아래 내용을 입력창에 복사 붙어녛기 한다.
	
				```
				{
					"tags" : [
							{
								"tagName": "temperature",
								"tagValue": "25.2"
							},
							{
								"tagName": "LED",
								"tagValue": "OFF"
							}	
					]
				}
			
				```
			4. **Send** 버튼을 클릭한다.
			
	![](figures/api-prod-run3.png)
	

3. 앞에서 정의한 응답모델과 동일한 형식의 JSon 문자열이 반환된 것을 확인할 수 있습니다.
