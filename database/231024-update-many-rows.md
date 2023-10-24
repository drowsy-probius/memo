# 많은 데이터 업데이트

## 상황

1.6M개의 row의 각 칼럼(`TYPE_CODE`)을 하나의 값(`OLD_TYP` -> `NEW_TYP`)으로 업데이트해야한다.

업데이트를 수행하면 해당 테이블에 lock이 걸린다. 그 동안에 다른 프로세스가 해당 테이블에서 작업을 쓰기 작업을 수행할 수 없게된다.

시스템에서는 일정 시간 간격마다 해당 테이블을 사용하는 작업이 수행된다. 데이터 업데이트는 배치 작업에 영향이 최대한 적게 가도록 수행해야한다.

배치 작업은 새로운 데이터를 해당 테이블에 작성하는 일이다. 기존에 작성된 데이터를 변경하는 일은 발생하지 않는다.

## 시도 1 - 단순 업데이트 수행

### 코드
```mysql
START TRANSACTION;

UPDATE DATA_TABLE
   SET TYPE_CODE = 'NEW_TYP'
 WHERE TYPE_CODE = 'OLD_TYP';

ROLLBACK;
```

### 결과

모든 row의 업데이트가 하나의 쿼리에서 수행되므로 전체 완료까지 1h 이상 소요된다. 너무 오래 걸리므로 프로세스 강제종료를 하였다.

> 다른 콘솔을 열고 `show processlist;`를 입력하면 현재 실행중인 프로세스목록과 어떤 데이터베이스를 어떤 명령어로 lock을 얻었는지 확인할 수 있다.

> `kill pid`


## 시도 2 - 분할 업데이트

반복문을 수행해야 하므로 DDL대신 파이썬을 사용하였다.

### 코드
```python
... # imports mysql, ...

try:
  TABLE_NAME = "DATA_TABLE"
  PRIMARY_KEYS = "`PK1`,`PK2`,`PK3`"

  global_conn = database.get_connection()
  select_cursor = global_conn.cursor(buffered=True)
  update_cursor = global_conn.cursor()

  select_query = f"""
  SELECT {PRIMARY_KEYS}
    FROM {TABLE_NAME}
   WHERE TYPE_CODE = 'OLD_TYP'
  """
  logger.info(f"target query: {select_query}")
  select_cursor.execute(
    select_query,
    multi=True,
  )
  total_size = select_cursor.rowcount
  logger.info(f"Total row fetched: {total_size} rows")

  affected_rows = 0
  fetch_size = 500

  next_set = select_cursor.fetchmany(fetch_size)

  # 현재 세션에서 binlog 생성하지 않도록 하여 과도한 로그 생성을 막는다 
  update_cursor.execute("SET sql_log_bin = 0;")
  while len(next_set):
    next_set_string = ", ".join(
      [f"""({', '.join([f"'{elem}'" for elem in tp])})""" for tp in next_set]
    )
    query = f"""
      UPDATE {TABLE_NAME}
         SET TYPE_CODE = 'NEW_TYP'
       WHERE ({PRIMARY_KEYS}) 
          IN ( {next_set_string} )
         AND TYPE_CODE = 'OLD_TYP'
    """
    update_cursor.execute(query)
    affected_rows += update_cursor.rowcount

    logger.info(f"{affected_rows} of {total_size} rows are affected.")
    next_set = select_cursor.fetchmany(fetch_size)
except Exception as e:
  print(e)
finally:
  # binlog 옵션 초기화
  update_cursor.execute("SET sql_log_bin = 1;")
  global_conn.commit()
  select_cursor.close()
  update_cursor.close()
  global_conn.close()
  logger.info("Resource Relaese Done")
```

### 결과

첫 `SELECT`에서 시간이 오래 걸리지 않는다.

전체 수행 시간은 첫번째 방법보다 오래 걸릴 수 있다. 그렇지만 각 쿼리 수행 시간은 30초를 넘기지 않는다. 따라서 `Lock wait timeout` 에러를 방지할 수 있을 것이다.

3h 30m + ...

비효율적인가?
