# 큰 데이터베이스 테이블 복사하기

## 상황

새로운 로직 테스트하기 위하여 큰 테이블 (60G 이상, 약 2천만개 row)의 복제본을 생성해야함.

대기 데이터베이스 서버가 없고 현재 서비스는 상시 실행중이어야 하므로 mysql 서버를 중지할 수 없고 lock이 걸리는 상황도 최대한 피하려고 한다.

외래키 관련 제약 사항은 없어 단순히 테이블 데이터만 복제하면 되는 상황이다.

## 시도 1 - mysql 명령어 사용

### 실행순서

```mysql
CREATE TABLE DATA_BACKUP AS SELECT * FROM DATA;
```

### 결과
1시간 반 넘게 실행해도 절반도 복사하지 못했음. 다른 효율적인 방안을 선택해야함.


## 시도 2 - 로컬 시스템에서 파일 복사

### 실행순서 

1. 빈 테이블 생성
```mysql
CREATE TABLE DATA_BACKUP LIKE DATA;
```
`mysql` 루트 폴더에서 데이터베이스 이름으로 된 폴더 내에 `DATA_BACKUP.ibd`파일이 생성된다. 

2. `TABLESPACE` 삭제
```mysql
ALTER TABLE DATA_BACKUP DISCARD TABLESPACE;
```
파일 시스템에서 테이블 관련 파일이 삭제된다.

3. 파일 시스템에서 테이블 복사
```bash
$ cd /var/lib/mysql
$ sudo cp DATA.idb DATA_BACKUP.idb
```

4. `TABLESPACE` 불러오기
```mysql
ALTER TABLE DATA_BACKUP IMPORT TABLESPACE;
```

### 결과
8분 30초정도 실행 후 실패함.

### 에러 메시지
```log
[2023-10-23 22:51:52] [HY000][1808] Schema mismatch (CFG file is missing and source table is found to have row versions. CFG file is must to IMPORT tables with row versions.)
[2023-10-23 22:51:52] [HY000][1810] InnoDB: IO Read error: (2, No such file or directory) Error opening './DATA_BACKUP.cfg', will attempt to import without schema verification
[2023-10-23 22:51:52] [HY000][1808] Schema mismatch (CFG file is missing and source table is found to have row versions. CFG file is must to IMPORT tables with row versions.)
[2023-10-23 22:51:52] [HY000][1030] Got error 168 - 'Unknown (generic) error from engine' from storage engine
```

cfg 메타데이터 파일이 없어서 데이터를 가져오지 못했다.

## 시도 3 - 메타데이터 작성 및 파일 복사

### 주의사항

메타데이터 export 과정에서 해당 테이블 전체에 read lock이 걸린다.

### 실행순서

1. 원본 테이블에서 메타데이터 내보내기
```mysql
FLUSH TABLES DATA FOR EXPORT;
```

약 130ms 소요.

테이플 export 혹은 백업 용도로 사용되는 명령어이다. 아래와 같은 작업이 수행된다.

- 테이블 잠금 (read lock)
- 커밋되지 않은 트랜잭션 쓰기

위 작업이 끝나더라도 테이블은 자동으로 언락되지 않는다.

생성된 메타데이터 파일은 UNLOCK시에 자동으로 삭제된다. 그러므로 UNLOCK 수행 전에 파일시스템에서 복사해야한다. 

다만 테이블 데이터 파일은 크기가 커서 수분에서 수십분이 걸릴 수 있으므로 그 동안에 해당 테이블 읽기 및 쓰기는 불가능하게 된다.

2. 파일 시스템에서 파일 복사 
```bash
$ cd /var/lib/mysql
$ sudo cp DATA.idb DATA_BACKUP_1.idb
$ sudo cp DATA.cfg DATA_BACKUP_1.cfg
$ sudo chown mysql:mysql DATA_BACKUP_1.idb DATA_BACKUP_1.cfg
```

추후 생성할 데이터베이스와 이름이 겹치므로 에러를 방지하기 위하여 `_1` 접미사를 추가한다.

약 8분 30초 소요.

3. 원본 테이블 lock 해제
```mysql
UNLOCK TABLES;
```

4. 빈 테이블 생성
```mysql
CREATE TABLE DATA_BACKUP LIKE DATA;
```
`mysql` 루트 폴더에서 데이터베이스 이름으로 된 폴더 내에 `DATA_BACKUP.ibd`파일이 생성된다. 

5. `TABLESPACE` 삭제
```mysql
ALTER TABLE DATA_BACKUP DISCARD TABLESPACE;
```
파일 시스템에서 테이블 관련 파일이 삭제된다.

6. 백업한 데이터 이름 변경
```bash
$ cd /var/lib/mysql
$ sudo mv DATA_BACKUP_1.idb DATA_BACKUP.idb
$ sudo mv DATA_BACKUP_1.cfg DATA_BACKUP.cfg
```

7. `TABLESPACE` 불러오기
```mysql
ALTER TABLE DATA_BACKUP IMPORT TABLESPACE;
```

23분 소요

### 결과
성공

1h 30m 초과 >> (31m 30s)으로 단축
