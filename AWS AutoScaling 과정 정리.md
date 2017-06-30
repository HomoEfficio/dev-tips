# AWS AutoScaling 과정 정리

## AutoScaling Trigger

1. AutoScaling Group의 AutoScaling Policy에 의해, 또는 AutoScaling Group의 Desired 수치를 수동으로 증가시키면 새 인스턴스 생성이 시작됨

## Instance Loading

1. 약 10초 후 AWS > EC2 > Auto Scaling > Auto Scaling Groups > Activity History 탭에 새 인스턴스의 id가 표시되며 'Not yet in Service'라고 표시됨
1. 약 70초 후 'Not yet in Service'가 'Successful'로 변경되며 인스턴스는 실제 구동할 수 있는 상태가 되고 터미널로도 접근 가능해짐
    - `Successful`은 인스턴스의 네트워크 서비스 등 기본적인 로딩이 완료되었음을 의미할 뿐, 인스턴스 상에 구성될 사용자 애플리케이션까지 구동할 수 있는 상태를 의미하는 것은 아님
    - Health Check 방식을 의미하는 `Health Check Type`의 기본값은 `EC2` 이며, `EC2` Status Check 주기는 1분이고, 
    - 실제 `Successful`로 전환되는 시기도 `Not yet in Service`로 표시된 후 대략 1분 이후인 걸로봐서 AutoScaling Group에서 설정한 Health Check의 결과를 기준으로 `Successful`로 전환되는 것으로 추정
    - AutoScaling Group에서 설정한 `Health Check Grace Period` 기간인 300초 동안은 Status Check의 결과가 OK가 아니더라도 AutoScaling Group에서 제외되지 않음    
1. `Successful`로 전환됨과 거의 동시에, 즉 `Health Check Grace Period`가 만료되기 전에 AWS > EC2 > Load Balancing > Load Balancers > instances 탭에서 새 인스턴스가 `OutOfService` 상태로 목록에 표시됨
    - 즉 인스턴스 자체의 로딩은 성공했지만, 사용자 애플리케이션이 아직 로딩되지 않은 상태

## Health Check

1. 터미널로 접근해서 nginx의 access.log를 확인하면, `Health Check Grace Period`가 만료되기 전부터 이미 ELB에 설정한 주기(10초)와 대상 경로(`/`)로 ELB가 Health Check를 시도하고 있음
1. `/` 가 사용자 애플리케이션에 바인딩 되어 있는 경우, 사용자 애플리케이션은 아직 로딩 완료된 상태가 아니므로, ELB의 Health Check는 HTTP 502 반환하며 ELB의 Health Check 실패
    - ELB의 Health Check가 ELB 설정의 `Unhealthy Threshold`에 설정한 회수 이상 지속되면 ELB는 새 인스턴스를 Unhealthy Host로 판정
    - 이 시기에 발생하는 HTTP 502 에러는, 새 인스턴스가 ELB에 `InService` 상태가 아니므로, ELB의 5xx 에러 집계 화면에는 카운트되지 않음

## In Service

1. 사용자 애플리케이션의 로딩이 완료되면 ELB의 Health Check는 HTTP 200 을 반환받으며 성공
1. ELB Health Check가 ELB 설정의 `Healthy Threshold`로 설정한 회수 이상 성공하면 인스턴스가 Healthy 한 것으로 간주되고 `InService` 상태가 되며 ELB가 실제 트래픽을 새 인스턴스에 전달하기 시작
