
![[Pasted image 20260413112058.png]]

**Foreground Threads:** 클라이언트 요청을 처리하는 스레드
**Background Threads**: 로그 기록이나 disk flush같은 내부 작업을 담당하는 스레드
### Threads_connected

- 현재 MySQL에 연결된 Connection의 개수이다.
- 지금 터미널로 접속해서 명령어를 치고 있는 그 연결. 
- `One-Thread-Per-Connection`
### Threads_running

- 현재 실제로 일을 하고 있는(CPU를 점유 중인) 일꾼의 수이다.
### Threads_created

- MySQL 서버가 켜진 이후 지금까지 새로 뽑은 총 일꾼 수이다.
	- 적을 수록 건강한 상태이다. -> 스레드 생성 비용은 비싸다. ㅠㅜ
### Threads_cached

- 일을 다 마치고 다음 손님 올 때까지 대기실에서 쉬고 있는 일꾼 수이다.
- `One-Thread-Per-Connection` 모델에서도 매번 스레드를 죽이지 않고, `thread_cache_size` 설정만큼은 Cache에 넣어둔다.


### 테이블 생성 전
```
mysql> show global status like 'Threads%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 0     |
| Threads_connected | 1     |
| Threads_created   | 1     |
| Threads_running   | 2     |
+-------------------+-------+
```

- 서버 시작: 시스템 일꾼 1명 생성 1 Connection = 1 Thread (`Created: 1`, `Connected: 1`)

- 원래 MySQL 서버 내부의 자잘한 업무(로그 정리, 통계 업데이트)를 담당하는 스레드가 하나 존재한다. 서버가 돌아가려면 최소한 이 친구가 필요하다. + 1개
- 내가 터미널로 MySQL 서버 접속한 순간 + 1개 (1Connection = 1 Foreground Thread)
	- > 따라서, running = 기본 관리자 1 + 나의 연결 1 = 총 2개


### 테이블 생성 후
```
mysql> show global status like 'Threads%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 1     |
| Threads_connected | 1     |
| Threads_created   | 2     |
| Threads_running   | 2     |
+-------------------+-------+
```

테이블을 생성하기 위해 Thread를 1명 뽑음 (`Created: 1`)
테이블 다 만들고 캐시로 들어감 (`Cached: 1`)


### 기물 이동
```
mysql> show global status like 'Threads%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 0     |
| Threads_connected | 2     |
| Threads_created   | 2     |
| Threads_running   | 3     |
+-------------------+-------+
4 rows in set (0.01 sec)
```

- 터미널(나)만 있던 DB 서버에 자바 프로그램(앱)이 접속. (`Connected: 2`)
	-> 기물을 옮기려면 DB에 명령을 전달할 통로가 필요하기 때문
		save() 메서드를 통해 getConnection()
- 기물을 옮기면 백엔드 서버가 `UPDATE` 쿼리를 날림.
	-> 그때 MySQL의 Thread 1 개가 추가됨. But, created가 아닌 cached에서 갖고옴. 
- cache에서 쉬고 있던 1명 델고 와서 지금은 0.
	-> 기물 이동 들어왔기 때문에 일 해야함.
- Threads_created는 그대로 2.
	-> cache 에 쉬고 있는 Thread 사용.
	-> 스레드 생성 비용(비싼 작업)을 아끼고 있는 건강한 상태.

---
### Thread Cache 실험
```
SHOW VARIABLES LIKE 'thread_cache_size';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| thread_cache_size | 9     |
+-------------------+-------+
```
기본적으로 'thread_cache_size' = 9이다.
`SET GLOBAL thread_cache_size = 0;` 으로 바꾼 후 exit 후 접속 반복 
접속할 때마다 `Threads_created` +1 
대기실이 없으니 매번 비싼 비용을 들여 created하게 됨.
- 적절한 값을 설정하면 사용자 접속을 위해 스레드를 만들어야 하는 부하를 감소시켜 성능 향상.

 
### Hikari CP 연결 실험
```Java
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import java.sql.Connection;
import java.sql.SQLException;

public class DatabaseManager {

    private static HikariDataSource ds;

    static {
        HikariConfig config = new HikariConfig();

        config.setJdbcUrl("jdbc:mysql://localhost:3307/jangi_db");
        config.setUsername("root");
        config.setPassword("1234");

        config.setMaximumPoolSize(20);
        config.setMinimumIdle(20);
        
        config.setPoolName("JangiPool");

        ds = new HikariDataSource(config);
    }

    public static Connection getConnection() throws SQLException {
        return ds.getConnection();
    }
}
```
`maximum-pool-size`를 20으로 설정하고 서버를 켠다.

```
mysql> show global status like 'Threads%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 1     |
| Threads_connected | 21    |
| Threads_created   | 2     |
| Threads_running   | 2     |
+-------------------+-------+
```
`Threads_connected`가 21(백1 + 풀20)로 팍 튀는 걸 확인할 수 있다.
HiriCp의 Thread Pool 이 20개의 Thread를 미리 생성해놓고, 내가 접속할 때 Connection 1 추가됨.

### 이 상태에서 기물 이동
```
mysql> show global status like 'Threads%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 0     |
| Threads_connected | 21    |
| Threads_created   | 2     |
| Threads_running   | 3     |
+-------------------+-------+
4 rows in set (0.00 sec)
```
`Threads_connected`는 21로 고정인데, `Threads_running`만 슥 올라갔다 내려옴.
아, 히카리CP가 미리 통로를 다 뚫어놔서 DB가 새로 일꾼을 뽑을 필요가 없구나
### MySQL Thread Pool 사용
```
mysql> show global status like 'Threads%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 1     |
| Threads_connected | 1     |
| Threads_created   | 8     |
| Threads_running   | 2     |
+-------------------+-------+
```
다만, 현재는 running Threads 가 적기 때문에 Thread Pool을 사용하는게 오히려 negative(-) 인 상황
Thread Pool은 사용자 접속이 많아지면 사용하자.

이상적인 사용 예시
사용 전)
```
mysql> show global status like 'Threads%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 10    |
| Threads_connected | 101   |
| Threads_created   | 101   |
| Threads_running   | 22    |
+-------------------+-------+
```
사용 후)
```
mysql> show global status like 'Threads%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 10    |
| Threads_connected | 101   |
| Threads_created   | 8     |
| Threads_running   | 8     |
+-------------------+-------+
```