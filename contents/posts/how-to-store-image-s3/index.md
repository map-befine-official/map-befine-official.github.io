---
title: "백엔드와 협력해 S3에 이미지 저장하기"
description: "괜찮을지도 크루들을 위해 작성된 글입니다."
date: 2023-10-18
update: 2023-10-18
tags:
  - 블로그
---

> 이 글은 우테코 괜찮을지도팀의 `패트릭`가 작성했습니다.

백엔드 크루 '매튜'와 협력해 S3에 이미지를 저장하는 작업을 진행하였습니다. 생각보다 시간도 오래 걸리고 어려움도 많았지만 많이 배웠고 보람있는 시간이었습니다.

### 이미지 관련 사용자 피드백 
이미지를 string으로 저장하고 있어 이미지를 저장하는데 번거럽다는 피드백을 받았습니다.

##  formData를 통해 이미지 서버에 전송하기

file type의 Input 태그를 만듭니다.<br>
file을 서버로 전송하면 해당 file 데이터를 multipart/form-data형태로 받습니다.

```typescript
<ImageInputButton id="file" type="file" name="image" onChange={onHandler} />
``` 

form 데이터를 동적으로 생성하고 전송 가능한 객체인 formData를 써서 내용을 key와 value 형식으로 보내도록 했습니다.

```typescript
const formData = new FormData();

if (formImage) {
   formData.append('image', formImage);
}
``` 

### 문제 상황 : 1차 서버 에러 발생
서버 Post 요청 시 서버쪽에서 아래와 같은 에러가 발생했습니다.
```java
// 에러
Caused by: org.apache.tomcat.util.http.fileupload.FileUploadException: the request was rejected because no multipart boundary was found
at org.apache.tomcat.util.http.fileupload.impl.FileItemIteratorImpl.init(FileItemIteratorImpl.java:189)
at org.apache.tomcat.util.http.fileupload.impl.FileItemIteratorImpl.getMultiPartStream(FileItemIteratorImpl.java:205)
at org.apache.tomcat.util.http.fileupload.impl.FileItemIteratorImpl.findNextItem(FileItemIteratorImpl.java:224)
at org.apache.tomcat.util.http.fileupload.impl.FileItemIteratorImpl.<init>(FileItemIteratorImpl.java:142)
at org.apache.tomcat.util.http.fileupload.FileUploadBase.getItemIterator(FileUploadBase.java:252)
at org.apache.tomcat.util.http.fileupload.FileUploadBase.parseRequest(FileUploadBase.java:276)
at org.apache.catalina.connector.Request.parseParts(Request.java:2799)
... 46 common frames omitted
``` 

에러의 원인은 boundary를 찾지 못해 나는 것이었고 formData에 파일이 있을 경우 브라우저는 자동으로 boundary를 붙여준다는 것을 알았습니다. 여기서 Post 요청을 할 때 개발자가 content-type을 multipart/form-data로 지정해주면 브라우저가 자동으로 생성한 boundary를 덮어써 버립니다. 그리고 서버는 요청 본문을 올바르게 파싱하지 못하여 파일 및 폼 데이터를 올바르게 처리하지 못합니다.<br>
그래서 content-type을 지우면 해결이 될거라고 생각해 지웠습니다. 그러나 역시 문제는 쉽게 해결되지 않는 법! 또 다른 에러가 발생하였습니다.


### 문제 상황 : 2차 서버 에러 발생
json 데이터를 application/json 타입으로 명시해 주지 않아 octet-stream(8비트 단위의 이진 데이터)로 인식하여 발생한 에러였습니다. 'Blob(Binary Large Object)'을 사용하여 해결할 수 있었습니다.

```java
const data = JSON.stringify(objectData);
// Blob처리로 JSON 문자열이 이진 데이터로 변환. 또한 type에 application/json을 명시
const jsonBlob = new Blob([data], { type: 'application/json' });

formData.append('request', jsonBlob);
``` 


이러한 과정을 통해 서버에 이미지와 데이터를 전송할 수 있었고 이를 바탕으로 S3에 이미지를 저장할 수 있었습니다!