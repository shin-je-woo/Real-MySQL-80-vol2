# 16. 복제

## 16.1 개요

- 복제는 소스 서버의 데이터 변경 사항을 레플리카 서버로 동기화하는 기능이다. 소스 서버에서 데이터와 스키마 변경이 먼저 발생하고, 레플리카 서버는 전달받은 변경 내역을 자신의 데이터에 반영한다.
- 복제의 핵심 목적은 확장성과 가용성 확보이다. 특히 읽기 트래픽 분산, 장애 대응, 백업 부하 분리, 분석 전용 서버 운영, 원거리 서비스 지연 완화에 자주 사용된다.
- 스케일 업은 한 대의 서버 성능을 높이는 방식이고, 스케일 아웃은 동일 데이터를 가진 서버 수를 늘려 분산 처리하는 방식이다. 복제는 스케일 아웃을 가능하게 해 서비스 안정성을 높인다.
- 백업을 소스 서버가 아니라 레플리카 서버에서 수행하면 운영 중인 소스 서버의 부하와 잠금 영향을 줄일 수 있다.
- 분석 쿼리는 대량 읽기와 복잡한 연산으로 서비스용 쿼리에 영향을 주기 쉬우므로, 레플리카를 분석 전용으로 사용하는 구성이 유리하다.
- 애플리케이션 서버와 DB 서버가 지리적으로 멀리 떨어져 있으면 통신 지연이 커질 수 있는데, 복제를 사용하면 애플리케이션과 가까운 지역에 레플리카를 두어 응답 속도를 개선할 수 있다.

## 16.2 복제 아키텍처

- MySQL 복제는 바이너리 로그를 기반으로 동작한다. 소스 서버는 변경 이벤트를 바이너리 로그에 기록하고, 레플리카 서버는 이를 받아 릴레이 로그에 저장한 뒤 재실행한다.

    <img width="431" height="442" alt="image" src="https://github.com/user-attachments/assets/9b2ed256-4bc4-4072-a309-47a37aaf0e18" />
    
- 바이너리 로그에는 데이터 변경뿐 아니라 데이터베이스나 테이블 구조 변경, 계정 관련 변경도 기록된다.
- 복제 처리에 관여하는 스레드는 세 개다.
    - 소스 서버의 바이너리 로그 덤프 스레드: 레플리카 요청에 따라 바이너리 로그 이벤트를 전송한다.
    - 레플리카의 I/O 스레드: 소스에서 이벤트를 받아 릴레이 로그 파일에 기록한다.
    - 레플리카의 SQL 스레드: 릴레이 로그 이벤트를 읽어 레플리카 서버에 적용한다.
- 레플리카의 I/O 스레드와 SQL 스레드는 독립적으로 동작한다. 따라서 SQL 적용이 다소 느려도 I/O 스레드는 계속 이벤트를 받아 적재할 수 있다.
- 레플리카 서버에 문제가 생겨도 소스 서버는 계속 동작할 수 있지만, 레플리카는 최신 상태를 유지하지 못할 수 있다.
- 복제가 시작되면 레플리카는 세 종류의 복제 관련 데이터를 관리한다.
    - 릴레이 로그: 소스의 바이너리 로그 이벤트를 저장하는 로컬 파일이다.
    - 커넥션 메타데이터: 소스 서버 접속 정보와 현재 읽고 있는 바이너리 로그 위치 정보다. 기본적으로 `mysql.slave_master_info` 테이블에 저장된다.
    - 어플라이어 메타데이터: 레플리카가 마지막으로 적용한 이벤트 위치 정보다. 기본적으로 `mysql.slave_relay_log_info` 테이블에 저장된다.

## 16.3 복제 타입

### 16.3.1 바이너리 로그 파일 위치 기반 복제

- 이 방식은 소스 서버의 바이너리 로그 파일명과 파일 내 위치값으로 이벤트를 식별한다.
- 레플리카는 자신이 어디까지 읽고 어디까지 적용했는지 파일명과 위치를 기준으로 관리하므로, 중단 후 재시작 시에도 이어서 동기화를 재개할 수 있다.
- 복제에 참여하는 모든 MySQL 서버는 고유한 `server_id`를 가져야 한다. 동일한 `server_id`를 가진 이벤트는 자신이 생성한 이벤트로 판단해 무시되므로, 중복되면 복제가 의도와 다르게 동작할 수 있다.
- 16.3.1.1 설정 준비
    - 소스 서버는 반드시 바이너리 로그가 활성화되어 있어야 한다.
    - MySQL 8.0에서는 기본적으로 바이너리 로그가 활성화되어 있고, `server_id`도 기본값 1로 설정된다. 실제 복제 환경에서는 각 서버마다 서로 다른 `server_id`를 명시해야 한다.
    - 필요하면 `log_bin`, `sync_binlog`, `binlog_cache_size`, `max_binlog_size`, `binlog_expire_logs_seconds` 등을 함께 설정한다.
    - 소스 서버에서 `SHOW MASTER STATUS`로 현재 바이너리 로그 파일과 위치를 확인한다.
    - 레플리카 서버도 고유한 `server_id`를 가져야 하며, 필요하면 `relay_log`, `relay_log_purge`, `read_only`, `log_slave_updates`를 설정한다.
    - `relay_log_purge=ON`이면 적용 완료된 릴레이 로그 파일이 자동 삭제된다.
    - `log_slave_updates`를 켜면 레플리카가 복제로 반영한 변경도 자신의 바이너리 로그에 기록한다. 이후 이 레플리카를 또 다른 서버의 소스로 쓸 수 있다.
- 16.3.1.2 복제 계정 준비
    - 레플리카는 소스 서버에 접속해 바이너리 로그를 읽어야 하므로 전용 계정이 필요하다.
    - 복제 계정에는 `REPLICATION SLAVE` 권한이 필요하다.
    - 복제 계정을 별도로 두는 이유는 접속 정보가 메타데이터에 저장되므로 보안을 분리하기 위해서다.
- 16.3.1.3 데이터 복사
    - 초기 복제를 위해 소스 서버 데이터를 레플리카에 적재해야 한다.
    - 일반적으로 `mysqldump`를 사용할 때 `-single-transaction`과 `-master-data`를 함께 사용한다.
    - `-single-transaction`은 InnoDB 테이블을 일관성 있게 덤프하도록 돕고, `-master-data`는 덤프 시점의 바이너리 로그 파일과 위치를 덤프 파일에 기록한다.
    - `-master-data` 사용 시 `FLUSH TABLES WITH READ LOCK`이 잠깐 실행되므로, 장시간 대기 상황이 발생하지 않도록 주의해야 한다.
    - 덤프 파일을 레플리카로 옮겨 적재한다.
- 16.3.1.4 복제 시작
    - 덤프 파일 헤더의 `CHANGE MASTER TO` 구문에서 파일명과 위치를 확인해 레플리카에 복제 설정을 적용한다.

        <img width="309" height="493" alt="image" src="https://github.com/user-attachments/assets/b999b73b-f7f2-48b0-b779-a0fc1a5d61a7" />
        
    - 설정 후 `SHOW REPLICA STATUS`에서 `Replica_IO_Running`, `Replica_SQL_Running`이 모두 Yes인지 확인한다.
    - `Seconds_Behind_Source=0`이면 소스와 레플리카가 동기화된 상태로 볼 수 있다.
    - 두 스레드 값이 Yes가 아니면 호스트, 포트, 계정, 비밀번호, 네트워크를 우선 점검해야 한다.
- 16.3.1.5 위치 기반 복제에서 트랜잭션 건너뛰기
    - 대표적인 복제 중단 원인 중 하나는 중복 키 오류다.
    - 위치 기반 복제에서는 `sql_slave_skip_counter`로 문제 이벤트 그룹을 건너뛸 수 있다.
    - 다만 이 값은 쿼리 하나가 아니라 이벤트 그룹 개수를 의미한다.
    - 트랜잭션 지원 테이블에서는 하나의 트랜잭션 전체가 하나의 이벤트 그룹이 될 수 있으므로, 의도하지 않은 여러 DML이 함께 건너뛰어질 수 있다.
    - 따라서 이 방식은 응급 복구용으로만 신중히 써야 한다.

### 16.3.2 글로벌 트랜잭션 아이디(GTID) 기반 복제

- 위치 기반 복제의 한계는 같은 이벤트라도 서버마다 파일명과 위치가 달라진다는 점이다. 장애 시 다른 서버를 소스로 승격하면 기존 위치 정보만으로는 이어서 복제하기 어렵다.
- GTID 기반 복제는 각 트랜잭션을 전역 ID로 식별하므로, 장애 후 다른 서버를 새 소스로 삼을 때 레플리카가 이미 실행한 트랜잭션과 아직 실행하지 않은 트랜잭션을 더 쉽게 구분할 수 있다.
- GTID 형식은 `source_id:transaction_id`다.
- 현재 사용 중인 GTID는 `mysql.gtid_executed` 테이블이나 `gtid_executed` 시스템 변수, `SHOW MASTER STATUS`로 확인할 수 있다.
- GTID는 단일 값뿐 아니라 범위와 여러 UUID 조합의 GTID 셋 형태로 표현된다.
- `mysql.gtid_executed`는 실행된 GTID를 저장하며, 시간이 지나면 구간 압축을 통해 연속 범위를 한 레코드로 줄인다.
- 16.3.2.1 GTID의 필요성
    - 장애 시 위치 기반 복제는 새 소스로 전환해도 파일 위치가 맞지 않을 수 있어 나머지 레플리카를 쉽게 재구성하기 어렵다.
    - GTID 기반 복제는 동일 트랜잭션에 동일 식별자를 유지하므로, 토폴로지 변경과 장애 복구가 훨씬 단순해진다.
    - 읽기 분산, 서버 승격, 서버 축소/확장 같은 운영 변경에도 유리하다.
- 16.3.2.2 글로벌 트랜잭션 아이디
    - GTID는 바이너리 로그에 기록된 트랜잭션에만 부여된다.
    - 비활성화 상태의 `sql_log_bin`으로 실행된 트랜잭션은 바이너리 로그에 기록되지 않으므로 GTID도 부여되지 않는다.
    - GTID 셋은 연속 구간을 `1-5`처럼 축약해 표현할 수 있고, 서로 다른 UUID의 값은 콤마로 구분한다.
- 16.3.2.3 글로벌 트랜잭션 아이디 기반 복제 구축
    - 소스와 레플리카 모두 `gtid_mode=ON`, `enforce_gtid_consistency=ON`이어야 한다.
    - 또한 각 서버의 `server_id`, `server_uuid`는 복제 그룹 내에서 고유해야 한다.
    - GTID 활성화된 소스에서 `mysqldump`로 레플리카를 구축할 때는 `-set-gtid-purged` 옵션이 중요하다.
    - 이 옵션은 덤프 시작 시점 GTID를 덤프 파일에 기록해, 적재 후 레플리카의 `gtid_purged`와 `gtid_executed`가 올바르게 초기화되도록 돕는다.
    - `-set-gtid-purged=AUTO`는 GTID 활성화 소스에서 자동으로 필요한 구문을 기록한다.
    - 단순 마이그레이션 용도라면 `-set-gtid-purged=OFF`로 바이너리 로그 비활성화 구문이 기록되지 않게 해야 할 수 있다.
    - 백업 도구나 XtraBackup으로 복구한 경우에도 복구 완료 후 `gtid_executed`, `gtid_purged`가 복구 시점 GTID에 맞게 초기화된다.
- 16.3.2.4 복제 시작
    - GTID 기반 복제에서는 파일명과 위치 대신 `SOURCE_AUTO_POSITION=1`만 지정한다.
    - 레플리카는 자신의 `gtid_executed`를 기준으로 아직 실행하지 않은 트랜잭션만 소스에서 가져온다.
    - `SHOW REPLICA STATUS`에서 `Auto_Position=1` 여부를 확인하면 GTID 기반 연결인지 알 수 있다.
- 16.3.2.5 GTID 기반 복제에서 트랜잭션 건너뛰기
    - GTID 기반 복제에서는 `sql_slave_skip_counter`를 사용할 수 없다.
    - 문제 트랜잭션을 건너뛰려면 레플리카에서 해당 GTID를 가진 빈 트랜잭션을 수동으로 생성해야 한다.
    - 절차는 복제를 멈추고, `gtid_next`에 문제 GTID를 지정한 뒤 `BEGIN; COMMIT;`으로 더미 트랜잭션을 만들고, 다시 `gtid_next='AUTOMATIC'`으로 복원한 후 복제를 재시작하는 방식이다.
    - `SHOW REPLICA STATUS`의 `Retrieved_Gtid_Set`과 `Executed_Gtid_Set`를 비교하면 어떤 GTID가 아직 적용되지 못했는지 파악할 수 있다.
- 16.3.2.6 GTID 기반 복제 제약 사항
    - GTID 환경에서는 `enforce_gtid_consistency=ON` 때문에 GTID 일관성을 해칠 수 있는 일부 쿼리를 실행할 수 없다.
    - 대표적으로 트랜잭션 지원/미지원 테이블을 함께 변경하는 쿼리, `CREATE TABLE ... SELECT ...`, 트랜잭션 내 `CREATE TEMPORARY TABLE` 또는 `DROP TEMPORARY TABLE` 사용 등이 제한된다.
    - GTID 기반 복제에서는 `IGNORE_SERVER_IDS`가 사실상 필요 없다. 이미 실행한 트랜잭션을 GTID로 식별해 자동 무시할 수 있기 때문이다.
- 16.3.2.7 Non-GTID 기반 복제에서 GTID 기반 복제로 온라인 변경
    - MySQL 5.7.6부터 서버 재시작 없이 GTID 모드를 순차적으로 변경할 수 있다.
    - 전환에 사용되는 핵심 시스템 변수는 `enforce_gtid_consistency`와 `gtid_mode`다.
    - `enforce_gtid_consistency`는 `WARN`으로 먼저 올려 경고 여부를 확인하고, 문제가 없으면 `ON`으로 변경한다.
    - `gtid_mode`는 `OFF -> OFF_PERMISSIVE -> ON_PERMISSIVE -> ON` 순으로 한 단계씩만 바꿀 수 있다.
    - 전환 전 모든 서버에서 잔여 익명 트랜잭션이 없는지 `ongoing_anonymous_transaction_count`로 확인해야 한다.
    - 최종적으로 `gtid_mode=ON`이 되면 `my.cnf`에도 `gtid_mode=ON`, `enforce_gtid_consistency=ON`을 반영해야 재시작 후에도 유지된다.
    - 이후 레플리카에서 `STOP REPLICA`, `CHANGE REPLICATION SOURCE TO SOURCE_AUTO_POSITION=1`, `START REPLICA` 순으로 GTID 기반 복제로 전환한다.
    - 반대로 GTID를 끄려면 위 순서를 역순으로 진행하면 된다.
    - `gtid_mode=ON`에서는 익명 트랜잭션이 생기지 않으므로, PIT 복구처럼 바이너리 로그의 익명 트랜잭션이 필요한 작업은 제약이 생길 수 있다.

## 16.4 복제 데이터 포맷

- MySQL 복제에서 중요한 것은 이벤트를 어떤 방식으로 바이너리 로그에 기록하느냐다. `binlog_format`으로 `STATEMENT`, `ROW`, `MIXED`를 선택할 수 있고, 선택에 따라 저장 공간, 복제 안정성, 디버깅 편의성이 달라진다.

### 16.4.1 Statement 기반 바이너리 로그 포맷

- 실행한 SQL 문장을 그대로 바이너리 로그에 기록하는 방식이다.
- 장점은 로그 크기가 작고, 사람이 읽고 해석하기 쉽다는 점이다. 감사나 원인 분석에도 유리하다.
- 단점은 비결정적 실행 결과가 발생할 수 있는 쿼리에서 소스와 레플리카의 데이터가 달라질 수 있다는 점이다.
    - `ORDER BY` 없는 `UPDATE` 또는 `DELETE ... LIMIT`
    - `NOWAIT`, `SKIP LOCKED`
    - `UUID()`, `USER()`, `RAND()`, `VERSION()` 같은 함수 사용
    - 결과가 달라질 수 있는 사용자 정의 함수나 스토어드 프로그램 사용
- 대량 데이터 변경 시에는 SQL 한 문장이 실제로 많은 행을 바꾸더라도 레플리카가 그 SQL을 다시 실행해야 하므로 처리 비용이 커질 수 있다.
- 트랜잭션 격리 수준도 중요하다. `REPEATABLE READ` 이상이 아니면 복제 시점 차이로 소스와 레플리카의 결과가 달라질 수 있어 Statement 포맷 사용이 허용되지 않는다.

### 16.4.2 Row 기반 바이너리 로그 포맷

- SQL 문장이 아니라 변경된 행 자체를 바이너리 로그에 기록하는 방식이다.
- 가장 큰 장점은 복제 안전성이다. 소스에서 어떤 SQL이 실행되었는지보다 실제 변경 결과를 레플리카에 적용하므로 비결정적 함수나 복잡한 쿼리에서도 일관성이 높다.
- 다음과 같은 경우 Statement보다 Row가 더 안전하거나 자동으로 선택되기 쉽다.
    - `INSERT ... SELECT`
    - `AUTO_INCREMENT` 사용 INSERT
    - 적절한 인덱스가 없어 대량 행을 변경하는 `UPDATE` 또는 `DELETE`
- 단점은 로그 크기가 커질 수 있다는 점이다. 변경된 행 수가 많거나 `BLOB`, 큰 컬럼이 포함되면 바이너리 로그가 급격히 커질 수 있다.
- 레플리카에서는 원본 SQL을 바로 읽을 수 없고, 변경된 행 기준으로 적용된다. 사람이 내용을 확인하려면 `mysqlbinlog -v` 같은 옵션으로 pseudo-SQL 형태로 풀어봐야 한다.
- DDL은 Row 포맷이어도 여전히 Statement 형태로 기록된다.

### 16.4.3 Mixed 포맷

- 기본은 Statement로 기록하되, 안전하지 않은 쿼리라고 판단되면 자동으로 Row로 전환하는 방식이다.
- 겉보기에는 두 방식의 장점을 모두 취하는 것처럼 보이지만, 실제로는 MySQL 내부 판단에 의존한다는 점을 기억해야 한다.
- 따라서 Mixed가 무조건 최선은 아니다. 애플리케이션 쿼리 특성과 운영 요구사항을 보고 선택해야 한다.

### 16.4.4 Row 포맷의 용량 최적화

- Row 포맷은 복제 안정성이 높지만 로그 크기 증가가 문제다. 이를 줄이기 위한 핵심 수단이 두 가지다.
- **바이너리 로그 Row 이미지 조정**
    - `binlog_row_image`로 변경 전후 기록 범위를 조정한다.
    - `full`
        - 기본값이다.
        - 변경 여부와 관계없이 필요한 행 이미지를 넓게 기록한다.
        - 해석은 쉽지만 로그가 커진다.
    - `minimal`
        - 꼭 필요한 컬럼만 기록한다.
        - 로그 크기를 줄이는 데 유리하다.
    - `noblob`
        - `full`과 비슷하지만 변경되지 않은 `BLOB` 또는 `TEXT` 컬럼은 제외한다.
- **바이너리 로그 트랜잭션 압축**
    - MySQL 8.0.20부터 Row 포맷 트랜잭션을 압축해서 기록할 수 있다.
    - 소스는 트랜잭션 변경 데이터를 압축해 하나의 payload 이벤트로 기록하고, 레플리카는 이를 받아 해제 후 적용한다.
 
        <img width="551" height="274" alt="image" src="https://github.com/user-attachments/assets/87910212-f40c-46d3-90ff-c81ba91585d3" />
        
    - 주요 변수는 다음과 같다.
        - `binlog_transaction_compression`
        - `binlog_transaction_compression_level_zstd`
    - 장점은 디스크 사용량과 네트워크 대역폭을 줄일 수 있다는 점이다.
    - 단점은 압축과 해제에 CPU와 메모리 오버헤드가 추가된다는 점이다.

## 16.5 복제 동기화 방식

- 소스와 레플리카 사이에서 커밋 결과를 언제 보장할지에 따라 복제 동기화 방식이 나뉜다.
- 여기서는 비동기 복제와 반동기 복제를 비교한다.

### 16.5.1 비동기 복제

- MySQL의 기본 복제 방식이다.
    
    <img width="519" height="422" alt="image" src="https://github.com/user-attachments/assets/1783560c-86b6-4da1-987f-6980dd18fd76" />
    
- 소스는 자신의 트랜잭션을 커밋하고 클라이언트에 바로 응답한다. 레플리카가 그 변경 이벤트를 실제로 받았는지는 기다리지 않는다.
- 장점은 성능이다.
    - 레플리카 응답을 기다리지 않으므로 트랜잭션 지연이 가장 적다.
    - 레플리카 수가 늘어나도 소스 성능 저하가 상대적으로 적다.
    - 읽기 분산, 분석 전용 레플리카 구성에 유리하다.
- 단점은 데이터 유실 가능성이다.
    - 소스 장애 시 최근 커밋 중 일부가 아직 레플리카에 전달되지 않았을 수 있다.
    - 장애 후 레플리카를 소스로 승격하면 누락 트랜잭션을 사용자가 직접 확인하거나 수동 복구해야 할 수도 있다.
- 일반적인 운영 환경에서는 레플리카 지연이 아주 길지 않을 수 있지만, 항상 안전하다는 보장은 없다. 민감한 읽기 요청은 레플리카보다 소스에서 직접 읽도록 분리하는 것이 좋다.

### 16.5.2 반동기 복제

- 비동기보다 더 강한 데이터 무결성을 목표로 하는 방식이다.
- 소스가 커밋 도중 레플리카의 ACK를 일부 기다린다.
- 핵심은 레플리카에 트랜잭션이 전달되었음을 보장하는 것이지, 레플리카에 적용까지 완료됨을 보장하는 것은 아니라는 점이다.
- **AFTER_SYNC 방식**
    
    <img width="571" height="520" alt="image" src="https://github.com/user-attachments/assets/08111d95-ebf8-48e7-925a-334f674b3934" />
    
    - 소스가 바이너리 로그 기록 후, 스토리지 엔진 커밋 전에 레플리카 ACK를 기다린다.
    - 장점은 장애 시 팬텀 리드 위험을 줄이고, 장애 복구 관점에서 더 안전하다는 점이다.
    - 소스에만 반영되고 레플리카에는 아직 없는 상태를 줄이므로, `AFTER_COMMIT`보다 무결성이 강하다.
- **AFTER_COMMIT 방식**
    
    <img width="583" height="533" alt="image" src="https://github.com/user-attachments/assets/6a071628-80c5-44a6-a3a6-e7cf78c96441" />
    
    - 소스가 스토리지 엔진 커밋까지 끝낸 뒤 레플리카 ACK를 기다린다.
    - 소스 장애 시 이미 커밋된 데이터가 레플리카에 없을 수 있어, 이후 재사용 시 사용자가 본 데이터와 실제 레플리카 데이터가 어긋날 가능성이 있다.
- MySQL 8.0 기본은 `AFTER_SYNC`다.
- 장점
    - 최소 1대 이상의 레플리카에 트랜잭션 전달을 보장할 수 있다.
    - 비동기보다 장애 시 데이터 손실 가능성을 줄인다.
- 단점
    - 레플리카 응답 대기 시간만큼 커밋 지연이 늘어난다.
    - 네트워크 왕복 시간이 큰 환경에서는 지연이 더 커진다.
    - 지정 시간 동안 ACK를 못 받으면 자동으로 비동기처럼 동작할 수 있다.
- 즉, 반동기 복제는 완전 동기화가 아니라 부분 동기화 보장이라고 이해해야 한다.

### 16.5.2.1 반동기 복제 설정 방법

- 반동기 복제는 플러그인 기반 기능이다.
- 소스와 레플리카에 각각 플러그인을 설치해야 한다.
    - 소스: `rpl_semi_sync_master`
    - 레플리카: `rpl_semi_sync_slave`
- 설치 후 `information_schema.plugins`나 `SHOW PLUGINS`로 상태를 확인한다.
- 핵심 변수는 다음과 같다.
    - `rpl_semi_sync_master_enabled`
    - `rpl_semi_sync_master_timeout`
    - `rpl_semi_sync_master_trace_level`
    - `rpl_semi_sync_master_wait_for_slave_count`
    - `rpl_semi_sync_master_wait_no_slave`
    - `rpl_semi_sync_master_wait_point`
    - `rpl_semi_sync_slave_enabled`
    - `rpl_semi_sync_slave_trace_level`
- 반동기 복제 적용 전 이미 복제가 돌고 있었다면 레플리카 I/O 스레드를 재시작해야 반영된다.
- 적용 확인은 다음 상태값으로 한다.
    - 소스: `Rpl_semi_sync_master_status`
    - 레플리카: `Rpl_semi_sync_slave_status`
- 재시작 후에도 유지하려면 my.cnf에 설정을 추가해야 한다.

## 16.6 복제 토폴로지

- MySQL 복제는 비교적 단순하게 구성할 수 있고, 목적에 맞게 다양한 형태로 확장할 수 있다.
- MySQL 5.7부터 멀티 소스 복제가 도입되면서 하나의 레플리카가 둘 이상의 소스를 가질 수 있게 되었고, 복제 토폴로지 선택 폭이 넓어졌다.

### 16.6.1 싱글 레플리카 복제 구성

<img width="304" height="285" alt="image" src="https://github.com/user-attachments/assets/b6d21058-43a6-4909-ba08-19280a14e50b" />

- 가장 기본적인 형태다. 하나의 소스 서버에 하나의 레플리카 서버가 연결된다.
- 보통 애플리케이션은 소스 서버에 직접 접근하고, 레플리카는 예비 서버나 백업 수행 용도로 두는 형태가 일반적이다.
- 레플리카를 서비스 조회 트래픽까지 담당하게 하면, 레플리카 장애가 곧 서비스 장애로 이어질 수 있다.
- 따라서 1:1 구성에서는 레플리카를 예비용으로만 두는 편이 적합하다.
- 배치, 통계, 서비스와 직접 연관되지 않은 조회는 레플리카에서 수행해도 무방하다.

### 16.6.2 멀티 레플리카 복제 구성

<img width="499" height="281" alt="image" src="https://github.com/user-attachments/assets/e3ba8bea-2156-4c34-972c-581922d2cc99" />

- 하나의 소스 서버에 2개 이상의 레플리카를 연결하는 형태다.
- 읽기 요청 분산이 가장 대표적인 용도다. 서비스 성장으로 읽기 부하가 증가하면 멀티 레플리카로 전환해 읽기 트래픽을 분산할 수 있다.
- 배치, 통계, 분석처럼 성격이 다른 작업을 레플리카별로 역할 분리해서 운영할 수도 있다.
- 다만 모든 레플리카를 업무용으로 꽉 채워 쓰면 장애 시 대체 여유가 사라진다.
- 그래서 최소 1대 정도는 대체 서버 겸 백업 용도로 남겨두는 구성이 바람직하다.

### 16.6.3 체인 복제 구성

<img width="723" height="463" alt="image" src="https://github.com/user-attachments/assets/c44f5b20-36ac-4809-96af-74e361674de8" />

- 레플리카 수가 많아져 소스 서버의 바이너리 로그 배포 부담이 커질 때 고려하는 구조다.
- 중간 레플리카가 다시 하위 레플리카들의 소스 역할을 맡아, 소스 서버의 부담을 줄인다.
- 1차 복제 그룹은 OLTP 서비스 용도, 2차 복제 그룹은 통계나 배치 용도처럼 계층별 역할 분리가 가능하다.
- 서버 교체나 업그레이드 시에도 유용하다. 새 장비를 하위 계층부터 붙여 동기화한 뒤 점진적으로 트래픽을 넘길 수 있다.
    
    <img width="513" height="317" alt="image" src="https://github.com/user-attachments/assets/399cdf5b-52f1-44a7-8c0d-c64c6cbbfd7b" />
    
    <img width="726" height="474" alt="image" src="https://github.com/user-attachments/assets/972dd3eb-4ce8-4315-bba4-9b4d0fac2c5f" />

    <img width="718" height="477" alt="image" src="https://github.com/user-attachments/assets/1dc36fe6-3062-497e-b296-e1c721291965" />

    <img width="707" height="311" alt="image" src="https://github.com/user-attachments/assets/1531e8f2-e81b-40f9-b22e-a79816dab675" />

- 주의할 점도 분명하다.
    - 중간 계층 서버는 레플리카이면서 동시에 소스 역할을 해야 하므로 바이너리 로그와 `log_slave_updates`가 필요하다.
    - `log_slave_updates` 는 자신의 binlog에도 복제 내역을 기록하는 옵션이다.
    - 중간 계층 장애 시 그 아래 하위 레플리카들도 함께 복제가 멈춘다.
- 즉, 소스 서버 부담은 줄일 수 있지만 장애 전파 범위와 운영 복잡도는 커진다.

### 16.6.4 듀얼 소스 복제 구성

<img width="301" height="295" alt="image" src="https://github.com/user-attachments/assets/3a0c1f91-0c6a-4798-b365-fed53b1c75b6" />

- 두 MySQL 서버가 서로 소스이자 레플리카가 되는 형태다.
- 가장 큰 특징은 양쪽 모두 쓰기가 가능하다는 점이다.
- 용도에 따라 두 형태로 나뉜다.
    - ACTIVE-PASSIVE
        - 실제 쓰기는 한 서버에서만 수행한다.
        - 다른 서버는 즉시 쓰기 가능한 예비 서버 역할을 한다.
        - 장애 시 빠른 쓰기 전환이 필요한 환경에 적합하다.
    - ACTIVE-ACTIVE
        - 두 서버 모두 쓰기를 수행한다.
        - 지역적으로 떨어진 두 지점에서 각각 쓰기를 받아야 할 때 사용할 수 있다.
        - 하지만 복제 지연 동안 두 서버 데이터가 일시적으로 불일치할 수 있다.
- 듀얼 소스 구성의 핵심 주의점은 다음과 같다.
    - 동일한 데이터를 양쪽 서버에서 동시에 변경하면 충돌이나 예상치 못한 최종 결과가 발생할 수 있다.
    - Auto-Increment 키 충돌이 발생할 수 있다.
- 따라서 ACTIVE-ACTIVE에서는 동일 데이터 동시 변경을 피해야 하고, Auto-Increment 사용도 지양하는 편이 좋다.
- 꼭 Auto-Increment를 써야 한다면 `auto_increment_increment`, `auto_increment_offset` 값을 조정해 충돌을 줄여야 한다.
- 실제로는 멀티 소스나 듀얼 소스가 쓰기 확장에 큰 도움이 되지 않는 경우가 많고, 오히려 충돌과 복잡도만 높이는 경우가 많다.
- 쓰기 확장이 필요하면 멀티 소스 복제보다 샤딩을 권장한다.

### 16.6.5 멀티 소스 복제 구성

<img width="419" height="288" alt="image" src="https://github.com/user-attachments/assets/04cc1a4f-f741-4143-9fa5-7ad5430e2d9b" />

- 하나의 레플리카 서버가 둘 이상의 소스 서버를 동시에 복제하는 형태다.
- 주 용도는 여러 소스의 데이터를 한 서버로 모으는 것이다.
    - 여러 MySQL 서버의 서로 다른 데이터를 한 서버로 통합
    - 여러 서버의 테이블 데이터를 하나의 테이블로 통합
    - 여러 서버의 데이터를 모아 한 서버에서 백업 수행
- 분석이나 통합 조회처럼 데이터 집계가 필요한 경우 매우 유용하다.
- 서비스 확장 과정에서 사전에 샤딩된 여러 서버의 데이터를 다시 한곳에 모으는 용도로도 활용할 수 있다.

### 16.6.5.1 멀티 소스 복제 동작

<img width="597" height="239" alt="image" src="https://github.com/user-attachments/assets/f4df8a90-b080-4b31-a9be-f56e885408b3" />

- 각 소스별 복제 처리는 독립적인 채널(Channel) 단위로 동작한다.
- 채널마다 개별 I/O 스레드, 릴레이 로그, SQL 스레드가 분리된다.

### 16.6.5.2 멀티 소스 복제 구축

- 단일 소스 복제와 비교해 절차 자체가 완전히 다르지는 않다.
- 차이는 여러 소스의 초기 데이터를 하나의 레플리카에 적재해야 한다는 점이다.
- 모두 빈 서버라면 단순히 연결만 하면 되지만, 둘 이상 소스에서 데이터를 가져와야 하면 충돌 가능성을 확인해야 한다.
- 백업 도구 선택이 중요하다.
    - `mysqldump`
        - 논리 백업이므로 병합 시 충돌 문제가 상대적으로 적다.
        - 대신 대용량 데이터에는 느릴 수 있다.
    - XtraBackup
        - 대용량 데이터 복구에는 유리하다.
        - 하지만 InnoDB 시스템 테이블스페이스까지 통째로 다루므로 여러 소스 데이터를 한 서버로 합칠 때는 제약이 크다.
- 그래서 여러 소스 데이터를 초기 적재할 때는 `mysqldump`와 XtraBackup을 상황에 따라 섞어 쓰는 것이 현실적이다.

## 16.7 복제 고급 설정

- 복제 고급 설정은 운영 중 복구 가능성, 복제 지연 완화, 장애 후 재시작 안정성, 특정 데이터만 복제하는 제어를 위한 기능이다.
- 핵심 기능은 지연된 복제, 멀티 스레드 복제, 크래시 세이프 복제, 필터링된 복제다.

### 16.7.1 지연된 복제(Delayed Replication)

- 지연된 복제는 레플리카가 소스 서버의 트랜잭션을 즉시 적용하지 않고 지정된 시간만큼 늦게 적용하는 기능이다.
- 실수로 데이터나 테이블을 삭제했을 때, 아직 삭제 이벤트가 적용되지 않은 레플리카에서 데이터를 복구할 수 있다.
- 지연 복제에서도 소스 서버의 바이너리 로그는 레플리카의 릴레이 로그로 즉시 복사된다. 지연되는 것은 SQL 스레드가 릴레이 로그 이벤트를 실행하는 시점이다.
- 설정 옵션은 다음과 같다.
    - MySQL 8.0.23 미만: `MASTER_DELAY`
    - MySQL 8.0.23 이상: `SOURCE_DELAY`

```
CHANGE MASTERTO MASTER_DELAY=86400;
CHANGE REPLICATION SOURCETO SOURCE_DELAY=86400;
```

- MySQL 8.0부터 바이너리 로그에는 두 타임스탬프가 기록된다.
    - `original_commit_timestamp(OCT)`: 원본 소스 서버에서 커밋된 시각이다.
    - `immediate_commit_timestamp(ICT)`: 직계 소스 서버에서 커밋된 시각이다.
- 체인 복제에서는 OCT와 ICT가 다를 수 있으며, MySQL 8.0은 ICT 기준으로 지연을 계산해 이전 버전의 이벤트 단위 지연 문제를 개선했다.

### 16.7.2 멀티 스레드 복제(Multi-threaded Replication)

- 멀티 스레드 복제는 레플리카 서버에서 복제 이벤트를 여러 워커 스레드가 병렬로 처리하는 기능이다.
- 단일 스레드 복제에서는 소스 서버에서 동시에 실행된 DML도 레플리카에서 순차 적용되어 복제 지연이 쉽게 발생한다.
- 멀티 스레드 복제에서는 SQL 스레드가 코디네이터가 되고, 실제 이벤트 실행은 워커 스레드가 담당한다.

<img width="874" height="298" alt="image" src="https://github.com/user-attachments/assets/9cf116aa-d1f3-438e-a819-d6b647059148" />

- 핵심 시스템 변수는 다음과 같다.
    - `slave_parallel_type`: 병렬 처리 방식이다.
    - `slave_parallel_workers`: 워커 스레드 수다.
    - `slave_pending_jobs_size_max`: 워커 큐에 할당 가능한 최대 메모리 크기다.
- `slave_parallel_workers=0`은 단일 스레드 복제다.
- `slave_parallel_workers=1`은 워커가 하나지만 코디네이터 비용이 추가되므로, 단일 스레드 목적이면 0을 사용한다.

### 16.7.2.1 데이터베이스 기반 멀티 스레드 복제

- 데이터베이스 기반 방식은 데이터베이스 단위로 이벤트를 나눠 병렬 처리한다.
    
    <img width="818" height="524" alt="image" src="https://github.com/user-attachments/assets/60ce680a-d8a4-4713-b8df-b48b9ed9ad47" />

- 데이터베이스가 하나뿐이면 병렬 처리 이점이 거의 없다.
- 여러 데이터베이스가 있고 각 데이터베이스의 DML이 독립적으로 발생하는 환경에서는 효과가 있다.
- 같은 데이터베이스에 대한 이벤트는 실제로 다른 테이블이나 다른 레코드를 변경하더라도 병렬 처리되지 않을 수 있다.
    
    <img width="650" height="408" alt="image" src="https://github.com/user-attachments/assets/59fee1f1-62a3-4779-9674-349a884bc0ac" />


```
slave_parallel_type='DATABASE'
slave_parallel_workers=N
```

- 이 방식에서는 레플리카 서버의 바이너리 로그 기록 순서가 소스 서버의 트랜잭션 순서와 달라질 수 있다.
- 따라서 레플리카에서 특정 트랜잭션이 보인다고 해서 그 이전 트랜잭션이 모두 반영됐다고 단정하기 어렵다.

### 16.7.2.2 LOGICAL CLOCK 기반 멀티 스레드 복제

- `LOGICAL_CLOCK` 방식은 데이터베이스 단위가 아니라 트랜잭션 간 의존 관계를 기준으로 병렬 처리한다.
- 같은 데이터베이스 안에서도 병렬 처리가 가능하므로 데이터베이스 기반 방식보다 활용도가 높다.
- logical clock은 실제 시간이 아니라 이벤트 선후 관계를 판단하기 위한 논리적 순번이다.

### 16.7.2.2.1 바이너리 로그 그룹 커밋

<img width="582" height="484" alt="image" src="https://github.com/user-attachments/assets/0c72ad27-b067-4ed9-9b54-137824493564" />

- MySQL은 스토리지 엔진 커밋과 바이너리 로그 기록의 일관성을 위해 Prepare와 Commit의 두 단계를 사용한다.
- MySQL 5.6부터 바이너리 로그 그룹 커밋이 도입되어 여러 트랜잭션을 묶어 처리할 수 있게 됐다.

<img width="832" height="290" alt="image" src="https://github.com/user-attachments/assets/914f68b9-1113-4592-8bb6-945a6378b77c" />

- 그룹 커밋 관련 핵심 변수는 다음과 같다.
    - `binlog_group_commit_sync_delay`: 바이너리 로그 동기화를 지연할 시간이다.
    - `binlog_group_commit_sync_no_delay_count`: 지연 시간과 관계없이 대기 가능한 최대 트랜잭션 수다.
    - `binlog_order_commits`: 바이너리 로그 기록 순서대로 스토리지 엔진 커밋을 수행할지 제어한다.

### 16.7.2.2.2 Commit-parent 기반 LOGICAL CLOCK 방식

- Commit-parent 방식은 같은 시점에 커밋된 트랜잭션을 레플리카에서 병렬 실행할 수 있게 하는 방식이다.
- 같은 `commit_seq_no`를 가진 트랜잭션은 병렬 처리 가능하다.
- 그룹 커밋 트랜잭션 수가 많을수록 레플리카 병렬 처리율이 높아진다.
- 이를 늘리기 위해 그룹 커밋 지연 값을 조정할 수 있지만, 소스 서버의 클라이언트 응답이 느려질 수 있다.

### 16.7.2.2.3 잠금(Lock) 기반 LOGICAL CLOCK 방식

- 잠금 기반 방식은 MySQL 5.7.6부터 사용된다.
- 커밋 시점이 완전히 같지 않아도 커밋 처리 구간이 겹치면 병렬 처리 가능하다고 판단한다.
- 바이너리 로그에는 다음 값이 기록된다.
    - `sequence_number`: 트랜잭션의 논리적 순번이다.
    - `last_committed`: 해당 트랜잭션 이전에 커밋된 최신 트랜잭션의 순번이다.
- 커밋 처리 시점이 겹치는 트랜잭션이 많을수록 병렬 처리 가능성이 커진다.
- 이 방식도 그룹 커밋 설정의 영향을 받는다.

### 16.7.2.2.4 WriteSet 기반 LOGICAL CLOCK 방식

- WriteSet 방식은 MySQL 8.0.1에서 도입됐다.
- 커밋 시점이 아니라 트랜잭션이 변경한 데이터를 기준으로 병렬 처리 가능 여부를 판단한다.
- 서로 다른 데이터를 변경한 트랜잭션은 병렬 실행할 수 있다.
- 병렬성을 가장 크게 개선하지만, 변경 데이터 해시 계산과 히스토리 비교 비용이 추가된다.

```
binlog_format=ROW
binlog_transaction_dependency_tracking={WRITESET|WRITESET_SESSION}
transaction_write_set_extraction=XXHASH64

slave_parallel_type=LOGICAL_CLOCK
slave_parallel_workers=N
```

- `binlog_transaction_dependency_tracking` 값은 다음과 같다.
    - `COMMIT_ORDER`: 기존 잠금 기반 방식이다.
    - `WRITESET`: 서로 다른 데이터를 변경한 트랜잭션을 병렬 처리한다.
    - `WRITESET_SESSION`: 같은 세션의 트랜잭션은 병렬 처리하지 않는다.
- `transaction_write_set_extraction`은 WriteSet 해시 알고리즘을 지정한다.
    - `OFF`
    - `MURMUR32`
    - `XXHASH64`
- WriteSet은 테이블의 유니크 키 기준으로 생성된다. PRIMARY KEY도 포함된다.

### 16.7.2.3 멀티 스레드 복제와 복제 포지션 정보

- 멀티 스레드 복제에서는 여러 워커가 이벤트를 병렬 처리하므로 완료 순서가 릴레이 로그 순서와 다를 수 있다.
- 워커별 포지션 정보는 `relay_log_info_repository` 설정에 따라 테이블 또는 파일에 저장된다.
- 복제 상태에서 보이는 포지션은 항상 최신 실행 위치가 아니라 체크포인트 기준일 수 있다.
- `slave_preserve_commit_order=1`이면 레플리카에서 소스 서버 커밋 순서와 동일한 순서로 커밋한다.
- 이 설정은 트랜잭션 갭을 줄이고 크래시 세이프 복제에도 유리하다.
- 체크포인트 관련 변수는 다음과 같다.
    - `slave_checkpoint_period`: 체크포인트 실행 주기다.
    - `slave_checkpoint_group`: 트랜잭션 개수 기준 체크포인트 주기다.

### 16.7.3 크래시 세이프 복제(Crash-safe Replication)

- 크래시 세이프 복제는 MySQL이 비정상 종료된 후에도 복제가 일관되게 재개되도록 하는 설정이다.
- 복제 포지션 정보와 실제 처리된 이벤트가 불일치하면 동일 이벤트가 중복 실행되거나 누락될 수 있다.
- 대표적인 결과가 `Duplicate key` 오류다.

### 16.7.3.1 서버 장애와 복제 실패

<img width="766" height="506" alt="image" src="https://github.com/user-attachments/assets/79b84403-c5c0-4db1-9686-35c74913d3f5" />

- I/O 스레드와 SQL 스레드는 각각 어디까지 가져왔고 적용했는지 포지션 정보를 남긴다.
- 파일 기반 포지션 정보는 이벤트 처리와 포지션 갱신이 원자적으로 묶이지 않아 비정상 종료 시 불일치가 생길 수 있다.
- MySQL 5.6부터 포지션 정보를 TABLE로 관리할 수 있다.
- SQL 스레드는 트랜잭션 적용과 포지션 정보 갱신을 하나의 트랜잭션으로 묶을 수 있다.
    
    <img width="680" height="392" alt="image" src="https://github.com/user-attachments/assets/2be6ebf7-6995-4563-b76e-9171827473e6" />

- I/O 스레드는 릴레이 로그 파일 쓰기와 포지션 갱신을 완전히 원자적으로 묶기 어렵다.
- 그래서 `relay_log_recovery=ON`이 필요하다.
- 크래시 세이프 복제의 최소 핵심 설정은 다음과 같다.

```
relay_log_recovery=ON
relay_log_info_repository=TABLE
```

- MySQL 8.0에서는 `relay_log_info_repository` 기본값이 `TABLE`이다.
- 이 설정은 MySQL 서버 비정상 종료에 대한 복구를 돕는 것이며, 운영체제나 장비 크래시로 데이터 파일 자체가 손상되는 상황까지 보장하지는 않는다.

### 16.7.3.2 복제 사용 형태별 크래시 세이프 복제 설정

16.7.3.2.1 파일 위치 기반 복제와 싱글 스레드 동기화

```
relay_log_recovery=ON
relay_log_info_repository=TABLE
```

- 파일 위치 기반 복제와 싱글 스레드 복제에서는 위 설정이 기본 크래시 세이프 설정이다.

16.7.3.2.2 파일 위치 기반 복제와 멀티 스레드 동기화

- 커밋 순서가 소스 서버와 일치하면 다음 설정이 핵심이다.

```
relay_log_recovery=ON
relay_log_info_repository=TABLE
slave_parallel_type=LOGICAL_CLOCK
slave_preserve_commit_order=1
```

- 커밋 순서가 일치하지 않으면 `sync_relay_log=1`이 필요할 수 있다.

```
relay_log_recovery=ON
relay_log_info_repository=TABLE
sync_relay_log=1
```

- 실무적으로는 `LOGICAL_CLOCK`과 `slave_preserve_commit_order=1`을 사용해 갭이 생기지 않게 하는 구성이 권장된다.

16.7.3.2.3 GTID 기반 복제와 싱글 스레드 동기화

- GTID 기반 복제에서는 자동 포지션 옵션이 핵심이다.

```
relay_log_recovery=ON
MASTER_AUTO_POSITION=1
SOURCE_AUTO_POSITION=1
```

- MySQL 8.0.23 미만은 `MASTER_AUTO_POSITION=1`, MySQL 8.0.23 이상은 `SOURCE_AUTO_POSITION=1`을 사용한다.
- 이 경우 복구 시 `mysql.slave_relay_log_info`가 아니라 `mysql.gtid_executed`를 참조한다.
- `gtid_executed`가 매 트랜잭션 적용 시 함께 갱신되지 않는 환경에서는 다음 설정도 필요하다.

```
sync_binlog=1
innodb_flush_log_at_trx_commit=1
```

16.7.3.2.4 GTID 기반 복제와 멀티 스레드 동기화

- 기본 설정은 GTID 기반 싱글 스레드 동기화와 같다.
- MySQL 8.0.18과 5.7.28부터는 GTID 기반이면서 AUTO_POSITION을 사용하면 불필요한 트랜잭션 갭 메우기 작업이 자동으로 생략된다.
- 이전 버전에서는 릴레이 로그 이벤트 손실로 복구가 실패할 수 있으며, 이때는 다음처럼 복제를 초기화해 재개할 수 있다.

```
STOP SLAVE;
RESET SLAVE;
START SLAVE;
```

### 16.7.4 필터링된 복제(Filtered Replication)

- 필터링된 복제는 소스 서버의 특정 이벤트만 레플리카 서버에 적용되도록 제한하는 기능이다.
- 필터링은 소스 서버와 레플리카 서버 양쪽에서 설정할 수 있다.
- 소스 서버 필터링은 바이너리 로그 기록 단계에서 적용된다.
- 레플리카 서버 필터링은 릴레이 로그 이벤트 실행 단계에서 적용된다.

### 소스 서버 필터링

- 소스 서버 필터링은 데이터베이스 단위로만 가능하다.
- 주요 옵션은 다음과 같다.
    - `binlog-do-db`: 지정한 데이터베이스 이벤트만 바이너리 로그에 기록한다.
    - `binlog-ignore-db`: 지정한 데이터베이스 이벤트를 바이너리 로그에 기록하지 않는다.
- 실행 중 동적 변경은 불가능하며, MySQL 시작 시 설정해야 한다.
- 여러 데이터베이스를 지정할 때 쉼표로 나열하지 말고 옵션을 반복해서 지정해야 한다.

```
binlog-do-db=db1
binlog-do-db=db2
```

### 레플리카 서버 필터링

- 레플리카 서버 필터링은 소스 서버 필터링보다 유연하며, 동적으로 변경할 수 있다.
- 핵심 명령은 `CHANGE REPLICATION FILTER`다.

```
CHANGE REPLICATION FILTER filter[, filter, ...] [FOR CHANNEL channel_name]
```

- 주요 필터 옵션은 다음과 같다.
    - `REPLICATE_DO_DB`: 복제 대상 데이터베이스를 지정한다.
    - `REPLICATE_IGNORE_DB`: 복제 제외 데이터베이스를 지정한다.
    - `REPLICATE_DO_TABLE`: 복제 대상 테이블을 지정한다.
    - `REPLICATE_IGNORE_TABLE`: 복제 제외 테이블을 지정한다.
    - `REPLICATE_WILD_DO_TABLE`: 와일드카드로 복제 대상 테이블을 지정한다.
    - `REPLICATE_WILD_IGNORE_TABLE`: 와일드카드로 복제 제외 테이블을 지정한다.
    - `REPLICATE_REWRITE_DB`: 특정 데이터베이스명을 다른 데이터베이스명으로 치환해 적용한다.
- 이미 복제가 실행 중이면 SQL 스레드를 멈추고 필터를 설정한 뒤 다시 시작해야 한다.
- `CHANGE REPLICATION FILTER`로 설정한 값은 MySQL 재시작 시 초기화된다. 유지하려면 설정 파일에 작성해야 한다.

### 바이너리 로그 포맷과 필터링 주의점

- 필터링 복제에서 가장 중요한 주의점은 바이너리 로그 포맷에 따라 같은 쿼리도 필터링 결과가 달라질 수 있다는 것이다.
- Statement 포맷은 `USE` 문으로 지정된 디폴트 데이터베이스를 기준으로 필터링한다.
- Row 포맷은 실제 변경된 테이블이 속한 데이터베이스를 기준으로 필터링한다.
- DDL은 Row 포맷에서도 Statement 포맷처럼 로깅되므로 `USE` 문 기준의 영향을 받는다.
- 따라서 필터링을 일관되게 사용하려면 다음을 지켜야 한다.
    - Row 포맷 사용 시 DDL은 `USE` 문으로 디폴트 데이터베이스를 설정하고, 쿼리에서 데이터베이스명을 직접 지정하지 않는다.
    - Statement 또는 Mixed 포맷 사용 시 DML과 DDL 모두 `USE` 문으로 디폴트 데이터베이스를 설정하고, 쿼리에서 데이터베이스명을 직접 지정하지 않는다.
    - Statement 또는 Mixed 포맷 사용 시 복제 대상 테이블과 복제 제외 테이블을 함께 변경하는 DML을 사용하지 않는다.
- 소스 서버에서 `binlog-do-db` 또는 `binlog-ignore-db`를 사용하는 경우도 바이너리 로그 포맷에 따라 결과가 달라질 수 있다.
