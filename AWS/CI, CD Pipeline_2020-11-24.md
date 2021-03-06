<h1>AWS CI/CD Pipeline</h1>

* 릴리즈 프로세스 스테이지
  1. Source
    * 필요한 소스 코드의 체크인
    * 새로운 코드의 리뷰 진행
  2. Build
    * 코트 컴파일
    * 유닛 테스트
    * 코드 스타일 체크(Lint)
  3. Test
  4. Deploy

* CI : Source ==> Build
* CD : Source ==> Build ==> Test ==> Deploy

<h2>CI/CD의 효과</h2>

* 개발에 필요한 시간이 줄어든다.

<h3>CI의 목표</h3>

1. 새 코드가 체크이되면 자동으로 빌드를 다시 시작한다.
2. 일관되고 반복 가능한 환경에서 코드 작성 및 테스트 가능
3. 지속적으로 배포할 준비가 된 아티팩트
4. 빌드 실패 시 지속적인 피드백 루프 닫기

* Amazon CodeBuild
  * 소스코드를 컴파일하고 테스트를 실행하며, 소프트웨어 패키지를 생성하는 완전관리형 빌드 서비스
  * 지속적으로 확장하고 여러 빌드를 동시에 처리할 수 있다.
  * 관리 해야할 빌드 서버가 없다.
  * 각 빌드는 일관되고 변하지 않는 환경을 위해 새로운 Docker 컨테이너에서 실행된다.
  * Docker와 AWS CLI는 모든 공식 CodeBuild 이미지에 설치.

<h3>CD의 목표</h3>

1. 테스트를 위해 스테이징 환경에 새로운 변경 사항을 자동으로 배치
2. 고객에게 영향을 주지 않고 안전하게 프로덕션에 배포
3. 고객에게 더 빠르게 제공 : 배포 빈도를 높이고 변경 리드 타임 및 변경 실패율을 줄인다.

* Amazon CodeDeploy
  * 모든 인스턴스 및 Lambda에 대한 코드 배포 자동화
  * 응용 프로그램 업데이트의 복잡성을 처리한다.
  * 응용 프로그램 배포 중 다운 타임 방지
  * 장애가 감지되면 자동 rollback을 한다.
  * Amazon EC2, Amazon ECS, Lambda 또는 온프레미스 서버에 배포.
<hr/>

<h2>CD의 문제점</h2>

* Challenges in the application life cycle
  * 개발자가 문제가 있는 코드를 식별하는데에 시간이 걸린다.
  * 코드 분석 도구에는 코드 품질 및 효율성에 대한 업계 표준 모범 사례가 없다.
  * 가장 비용이 많이드는 코드 줄을 시각화하고 수정하는 방법은 어렵다.
<hr/>

<h2>Git Branch Polity</h2>

* 하나의 Repository는 하나의 application만 만든다.
* 살아있는 branch는 현재 작업이 진행중인 branch이다.
* Merge를 최소화하라.(Rebase)
<hr/>

<h2>HoL 사용 서비스</h2>

* Amazon Cloud9
* Amazon CloudFormation
* Amazon CodeCommit
* Amazon CodeDeploy
* Amazon CodeBuild
* Amazon CodeGuru (us-west-2 Region)
<hr/>