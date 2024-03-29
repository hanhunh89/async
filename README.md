# MSA 구축을 위한 비동기 서비스 사례

## 들어가면서
 어딜가나 MSA가 화제다. 내가 일했던 조직에서 진행했던 MSA를 위한 비동기 프로젝트를 소개하고자 한다.<br>

 김또띠 과장은 오늘도 출근하자 마자 결재할 내용이 산더미다. 인사시스템에서 휴가/출장을, 재정시스템에서 법인카드 사용을,<br>
 보안시스템에서 외부인 출입신청을 결재해야 한다. 메일은 하루에 수십개씩 오고, 전자결재 시스템에서는 접수할 문서가 밀려있다.<br>
 이 모든것들을 결재할 때마다 각 서비스에 접속을 해야 한다.<br> 가끔은 결재해야하는것을 잊을때도 있다.<br>
 사내 포탈에 접속했을 때 결재해야 할 내용에 대해 알람을 보내줄 수 없을까?<br>

 사내 "알림이" 시스템은 위와 같은 고민에서 시작되었다. <br>
 이 글에서 팀 편의기능에서 시작한 알림이가 전사적으로 전파되며 경험한 내용을 공유하고자 한다.

## 1. 팀장님 왜 결재 안해줘요?
 내가 일하던 회사는 보안에 아주 민감한 곳이었다. 외부인이 들어오려면 2일 전 보안부서에 신청을 해야했다.
a) "팀원 신청" b) "팀장 결재" c) "보안부서 결재"를 거치는 구조였다.
 너무나 바쁜 팀장은 결재하는것을 자주 잊어버렸다. 이럴땐 "긴급 출입" 기능을 이용했다. 장애대응이나 응급환자 발생 등의 긴급상황에 사용하는 기능이었다.
이 기능을 사용하면 일단 인원을 출입시키고 사후에 보안부서의 결재를 받았다. 팀장이 결재를 잊을때면 "긴급 출입"으로 외부인원 출입을 신청했다. 결국 감사팀에 적발되고 말았다.
 구조적으로 이러한 문제를 해결해야 했다. "알림이" 탄생의 시작이었다.

  
## 2. 알림이의 탄생

버려지는 pc에 알림이 서버를 구축했다. 방법은 간단했다. 출입신청 시스템에서 결재대기 건수를 파싱했다.
그 자료를 팀의 홈페이지 화면에 보여주었다. 아래의 그림과 유사하게 만들었다. 
![이미지 대체 텍스트](1.jpg)<br>
팀 홈페이지가 로딩될 때마다 전자결재 등 각 서비스에 접속해서 결재대기 건수를 파싱해왔다. 전자결재를 클릭하면 전자결재 화면으로 넘어가게 만들었다. 
거기서 내용을 검토하고, 결재할 수 있었다. 그 이후 팀장이 결재를 잊어버리는 일은 확 줄었다. 

## 3. 나도 쓰고싶은데요? 그러다 뒤져...

알리미가 입소문을 타기 시작했다. 알림이에 대한 수요가 늘었다. 결국 사내 메인 포탈에 적용되기로 결정했다. 용감하게 바로 사내 메인 포탈에 적용했다.
의외로(?) 잘 돌아갔다. 알림이에 대한 호평이 쏟아졌다. 오만가지 서비스에서 알림이 추가를 요청했다. 드디어 기다리던 장애가 발생했다. 하지만 생각지도 못한 문제였다.<br>

마지막으로 추가한 서비스는 한달에 한번쯤 접속하는 서비스였다. 서버는 커녕 수명이 지난 pc를 사용하고 있었다. 알림이에 연동하기 전까지 하루 접속량이 10건이 되지 않았다. 
하지만 알림이에 연동하고 나서부터 접속량이 매우 늘었다. 메인 포탈이 호출될 때 마다 해당 서비스가 호출되었다. 접속량이 100배가 넘게 늘었을 것이다. 순식간에 서버가 죽었다. <br>

해당 서비스를 알림이에서 제외시키면 될 문제였다. 하지만 우리는 알림이를 범용적으로 쓰고싶었다. 

## 4. 비동기로 자료를 넘겨주세요.
알림이의 구조를 체계적으로 변경해야 했다. 이참에 사내 지원을 받아 하드웨어 자원도 빵빵하게 늘리기로 했다. 그래서 다음과 같이 알림이를 설계했다. <br>

1. 각 서비스에서 "결재 필요" 이벤트가 생기면 알림이로 비동기 통신을 보낸다. 이 때 "서비스명/사용자정보/결재건수/발생시간/고유번호" 정보를 json 형태로 보낸다.
   각 서비스는 비동기 형태로 데이터를 보내기 때문에 알리미 서비스 연동으로 인한 부하는 없다. 알리미 체계의 장애로 인한 영향성도 없다.
2. 데이터를 수신한 알림이는 새로 수신한 데이터가 최신 데이터인지 확인한다. 비동기 서비스에서는 최근에 수신한 데이터가 먼저 수신한 데이터보다 더 나중에 발생하였단 것을 보증할 수 없기 때문이다. 
   각 서비스는 알림이에게 데이터를 보낼 때 마다 고유번호를 증가시킨다. 일림이에서는 이 고유번호를 통해 어떤 데이터가 최신의 데이터인지 알 수 있다. 최신의 데이터를 수집할 경우, 사용자가 각각의 서비스에서 결재할 내용을 최신화한다.<br>
   
이렇게 각 서비스와 알림이의 부담을 최소화하면서 서비스를 개선하였다. 장애없이 잘 돌아가는 것을 보며 아주 뿌듯했다. 

## 5. 비동기로 싸그리 바꿔버리자.
다른팀도 알림이에 관심을 갖기 시작했다. 수많은 서비스와 연동하면서 고난의 행군을 걸을 것이라 예상했것만, 너무나도 손쉽게 해결해버렸기 때문이다. 특히 연동하는 서비스가 많았던 팀에서 관심이 지대했다. 대 비동기 시대가 열리려고 하고 있었다. 이 때 새로운 문제가 생겼다. 알림이 하나만 있을 때는 상관없었다. 하지만 비동기로 보내야 하는 서비스가 늘어났다. 각 서비스의 담당자들이 짜증을 내기 시작했다. 똑같은 비동기 데이터를 여러 곳으로 보내야 했다. 새로운 대안이 필요했다. 

## 6. 각 서비스간의 통신을 표준화시키자
이 문제를 해결하려면 꿈을 크게 가져야 했다. 우리는 각 서비스간의 통신을 표준화하기로 했다. 때마침 오래된 레거시 시스템을 새로운 시스템으로 갈아엎는 프로젝트를 계획하고 있었기 때문에 가능한 생각이었다. 비동기 통신이 필요한 서비스는 직접 통신하지 않고 브로커를 통하여 통신하게 했다. 데이터 생산자가 브로커에 데이터를 보내면 브로커는 그 데이터를 저장한다. 소비자는 브로커에 접속해서 데이터를 소모했다. 예를들면, 신입사원이 들어오면 인사관리체계는 새로운 신입사원의 정보를 브로커에게 비동기로 전송했다. 다른 서비스들은 브로커에 접속해서 신입사원의 정보를 받아갔다. 인사시스템의 부담이 확연하게 줄었다. 인사시스템의 장애가 타 시스템의 장애로 퍼지지도 않았다. 매우 성공적이었다. 하지만 메시지 브로커에 연동하는 서비스가 늘어나면서 브로커가 죽는일이 발생했다. 

## 7. 브로커 강화하기 ! 실패했습니다. 브로커가 자연의 품으로 돌아갔습니다. 
브로커의 다중화 구축이 필요했다. 다중화 자체는 어려운 문제가 아니었다. WAS와 DB를 ACTIVE-STANBY로 구성하기만 하면 되니까. 하지만 해피엔딩은 없는 법. 여기서 우리의 프로젝트는 멈췄다. 그렇게 각 서비스간의 통신을 표준화시키는것은 실패했다. 거기까지였다<br><br>

........여름이었다 ★

## 마치며
이 글이 너무 허망하다고 생각하는가? 나의 마음을 일부라도 느끼다니, 위로가 된다. 현실의 서비스는 언제나 우리의 기대와는 다르게 움직인다. 우리가 시도했던 방식은 현재 MSA에서 널리 쓰이는 방식이다. 요즘에는 apache kafka, RabbitMQ와 같은 프레임워크를 주로 사용한다. 이 글을 통해서 비동기 기반의 MSA간 연동을 아주 간략하게나마 이해했길 바란다. 
