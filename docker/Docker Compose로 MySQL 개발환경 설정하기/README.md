# 로컬 환경

- MacOS
- Docker 27.1.1
- Docker Compose v2.29.1
- DB IDE : DataGrip

# docker-compose

docker-compose는 단일 서버에서 여러개의 컨테이너를 하나의 서비스로 정의하여 컨테이너를 묶음으로 관리할 수 있는 개발 환경을 제공하는 관리도구 입니다.

## docekr-compose를 사용하는 이유

- yaml파일로 애플리케이션의 서비스를 구성하여 팀원에게 동일한 개발환경을 제공
- 하나의 애플리케이션에 필요한 여러 서비스를 하나의 서비스로 정리하여 컨테이너를 묶음으로 관리
- 각 컨테이너의 의존성, 네트워크, 볼륨 등을 함께 정의

---

# docker-compose로 MySQL 구성하기

## 버전 확인

```bash
$ docker-compose --version
```

먼저 버전을 확인하여 설치 유무를 체크합니다.   
Windows와 MacOS는 [Docker Desktop](https://docker.com/products/docker-desktop/) 설치 시 자동으로 설치가 이루어집니다.

## yaml 파일 생성

새 디렉토리를 생성하여 생성한 디렉토리 하위에 yaml 파일을 생성합니다.   
![Alt text](/docker/Docker Compose로 MySQL 개발환경 설정하기/images/1.png)

## 설정

생성한 yml에 다음과 같이 설정합니다.

```yaml
services:
  mysql:
    image: mysql:8.0 # 생성이미지:버전
    container_name: mysql # 생성 할 컨테이너 이름
    restart: always # 수동종료 전까지 항상 켜지도록 유지 (sleep 방지)
    ports:
      - "13306:3306" # 포트번호 host:docker 로컬 포트에 도커 포트를 마운트
    volumes:
      # 로컬저장경로:도커저장경로 / 컨테이너 종료 후에도 데이터를 로컬에 저장하여 유지 (로컬 경로변경 가능)
      - ./db/mysql/data:/var/lib/mysql
      # 로컬저장경로:도커저장경로 / 해당 경로에 작성된 DDL을 컨테이너 생성 시 자동실행 (.sql .sh 파일 실행) (로컬 경로변경 가능)
      - ./db/mysql/init:/docker-entrypoint-initdb.d
    environment: #===== db 환경변수 =====#
      MYSQL_ROOT_PASSWORD: 1234 # root 계정 비밀번호 설정
```

### db 환경변수 추가 명령어

- 생성이미지:latest → 최신버전으로 설정
- 데이터 베이스 생성
    - MYSQL_DATABASE: edu → 데이터베이스 생성 이름
    - MYSQL_CHARSET: utf8mb4 → 인코딩 문자
    - MYSQL_COLLATION: utf8mb4_unicode_ci → 대조 문자
- MYSQL_DATABASE 명령어로 데이터베이스를 생성하면 아래 명령어로 생성한 유저는 해당 데이터베이스의 슈퍼유저가 됩니다.
    - MYSQL_USER: user → 생성할 유저 이름
    - MYSQL_PASSWORD: 1234 → 유저 비밀번호 (USER, PASSWORD 두가지 모두 명시해야함)
- TZ: Asia/Seoul → 타임존 설정

### restart 추가 명령어

- no → 수동으로 재시작 설정
- on-failure → 오류 발생 시 재시작

## 실행

실행 전 앞서 yml을 생성한 디렉토리의 경로로 이동합니다.   
![Alt text](/docker/Docker Compose로 MySQL 개발환경 설정하기/images/2.png)

- Docker Container 실행

```bash
$ docker-compose up -d
```

- Docker Container 종료

```bash
$ docker-compose down
```

- up
    - docker run 명령어와 비슷한 개념을 가지고 있습니다.
    - yml 파일에 작성한 설정대로 이미지를 내려받고 컨테이너를 실행합니다. (컨테이너 없을 시 생성 후 실행)
    - yml 파일에는 네트워크나 볼륨에 대한 정의도 작성할 수 있어 주변 환경을 하나로 묶어 생성할 수 있습니다.
- down
    - 컨테이너와 네트워크를 종료 및 제거합니다.
    - 볼륨과 이미지는 제거되지 않습니다.
    - 컨테이너와 네트워크를 제거하지 않고 종료만 하고자 한다면 stop 명령어를 사용합니다.
- -d
    - 백그라운드에서 docker-compose를 실행하기 위해 사용합니다.
    - docker-compose up만으로 실행 시 테스트가 끝날 때까지 터미널을 실행하고 있어야합니다. (터미널 종료시 컨테이너 작동x)

![Alt text](/docker/Docker Compose로 MySQL 개발환경 설정하기/images/3.png)   
Docker 컨테이너 실행 확인 이미지

![Alt text](/docker/Docker Compose로 MySQL 개발환경 설정하기/images/4.png)   
데이터 저장 경로 이미지

data: 해당 컨테이너의 데이터가 저장됩니다. 컨테이너 종료 후에도 데이터를 디렉토리에 저장하여 유지합니다.

          - ./db/mysql/data:/var/lib/mysql 앞서 설정 한 명령어로 도커 저장경로와 마운트

init: 해당 디렉토리에 작성된 DDL을 컨테이너 생성 시 자동실행합니다. .sql .sh 파일을 실행시 자동 실행 (초기 데이터 구성)

        - ./db/mysql/init:/docker-entrypoint-initdb.d 앞서 설정 한 명령어로 도커 저장경로와 마운트

# 계정 생성 및 권한 설정

```sql
-- 유저 목록 조회
select *
from mysql.user;

-- 유저 생성
-- create user '{유저명}'@'{host명}' identified by '{패스워드}';
create user 'tao'@'localhost' identified by '1234';

-- 유저 삭제
-- drop user '{유저명}'@'{host명}'
drop user 'tao'@'localhost';

-- 해당 데이터베이스에 모든 접근권한 설정
-- grant all privileges on {DB명}.{테이블명, * = 모든 테이블} to '{유저명}'@'{host명}';
grant all privileges on tistory.* to 'tao'@'localhost';

-- 해당 계정에 create, selet 권한 설정
-- grant {권한} on {DB명}.{테이블명, *= 모든테이블} to '{유저명}'@'{host명}';
grant create, select on tistory.* to 'tao'@'localhost';

-- 해당 계정에 권한 해제
-- revoke {권한} on {DB명}.{테이블명, *= 모든테이블} from '{유저명}'@'{host명}';
revoke create on tistory.* from 'tao'@'localhost';

-- 접근권한 설정 적용
flush privileges;
```

## DB 접근권한 목록

| 권한                          | 내용                                                                      |
|-----------------------------|-------------------------------------------------------------------------|
| **all [privileges]**        | grant option을 제외한 모든 권한                                                 |
| **alter**                   | alter table을 할 수 있는 권한                                                  |
| **alter routine**           | stored routines를 alter하고 drop할 수 있는 권한                                  |
| **create**                  | database와 table을 생성할 수 있는 권한                                            |
| **create routine**          | stored routine을 생성할 수 있는 권한                                             |
| **create temporary tables** | create temporary table을 사용할 수 있는 권한                                     |
| **create user**             | create user, drop user, rename user와 revoke all privileges를 사용할 수 있는 권한 |
| **create view**             | views를 생성하고 alter할 수 있는 권한                                              |
| **delete**                  | delete 할 수 있는 권한                                                        |
| **drop**                    | databases, tables, views를 drop 할 수 있는 권한                                |
| **event**                   | event scheduler를 위해 event를 사용할 수 있는 권한                                  |
| **execute**                 | stored routines를 실행할 수 있는 권한                                            |
| **file**                    | 서버에 생성된 파일을 읽고 쓸 수 있는 권한                                                |
| **grant option**            | 다른 계정에 grant를 하고 권한을 revoke할 수 있는 권한                                    |
| **index**                   | 인덱스를 생성하고 drop할 수 있는 권한                                                 |
| **insert**                  | insert 구문을 사용할 수 있는 권한                                                  |
| **lock tables**             | select 권한을 가진 테이블에 lock tables를 할 수 있는 권한                               |
| **process**                 | 모든 프로세스를 볼 수 있게 show processlist를 할 수 있는 권한                             |
| **reload**                  | flush구문을 사용할 수 있는 권한                                                    |
| **select**                  | select 할 수 있는 권한                                                        |
| **show databases**          | 모든 데이터베이스에 대해 show databases를 할 수 있는 권한                                 |
| **show view**               | show create view를 사용 할 수 있는 권한                                          |
| **shutdown**                | mysqladmin shutdown을 할 수 있는 권한                                          |
| **super**                   | change master to, kill, purge binary logs와 set global 문장을 실행할 수 있는 권한   |
| **trigger**                 | trigger를 생성하고 drop할 수 있는 권한                                             |
| **update**                  | update 할 수 있는 권한                                                        |
