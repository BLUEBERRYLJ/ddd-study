# Untitled

## 트랜잭션 스크립트

- 트랜잭션 스크립트 패턴: 프로시저의 시스템의 비즈니스 로직 구성
- 프로시저는 퍼블릭 인터페이스를 통해 구현

### 구현

**각 작업은 성공하거나 실패할 수 있지만, 유효하지 않은 상태를 만들어서는 안된다**

[잘못된 예]

```csharp
public class LogVisit{
	public void Execute(Guid userId, DataTime visitedOn){
	_db.Execute("UPDATE Users SET last_visit=@p1 WHERE user_id=@p2", visitedOn, userId);
	_db.Execute("INSERT INTO VisitLog(user_id, visit_date)i VALUES(@p1, @p2)", userId, visitedOn);
}

```

위와 같은 상태일 경우 update 문이 성공하고 insert 문이 실패했을 시, 시스템이 일관되지 않은 상태가 된다.

따라서 아래와 같이 트랜잭션을 만들 수 있다.
[수정된 예]

```csharp
public void Execute(userId, visitedOn{
	try{
		_db.StartTransaction();
		_db.Execute("UPDATE Users SET last_visit=@p1 WHERE user_id=@p2", visitedOn, userId);
		_db.Execute("INSERT INTO VisitLog(user_id, visit_date)i VALUES(@p1, @p2)", userId, visitedOn);
	} catch{
		db.Rollback()
	}
})

```

### 분산 트랜잭션

분산 시스템을 사용하여, update 쿼리는 현재 데이터배이스에, 기존의 insert 쿼리는 message를 발생해 처리를 하는 경우에는 아래와 같은 상황이 발생할 수 있다.

```csharp
public class LogVisit{
	public void Execute(Guid userId, DataTime visitedOn){
		_db.Execute("UPDATE Users SET last_visit=@p1 WHERE user_id=@p2", visitedOn, userId);
	_message.Publish("VISITS_TOPIC", new {UserID = userId, VisitedDate = visitedOn});
}

```

이 경우, execute 하는 코드와 message를 publish 하는 코드 사이에 일어나는 에러들은 시스템의 정합성을 손상시킬 수 있다.
UPDATE문은 실행되었지만 message는 publish되지 않기 때문이다.

=> 질문! db.Execute와 message publish 사이의 함수를 try catch에 넣고 rollback하면 되지 않나?

### 암시적 분산 트랜잭션

```csharp
public class LogVisit{
	public void Execute(Guid userId){
		_db.Execute("UPDATE Users SET visits = visits + 1 WHERE user_id = @p1", userId);
	}
}
```

- 위와 같은 코드에서는 마지막 방문 날짜를 추적하는 것이 아닌 메서드를 호출하면 해당 카운터 값이 1씩 증가하는 구조이다.
- 하나의 메서드가 하나의 데이터베이스의 하나의 테이블에서만 값을 업데이트하므로 일관성이 유지될 가능성이 높음 -> 완벽하게 유지되지 않을 수도 있음
    - 위 메서드가 REST 서비스의 일부에서 쓰이고 있고 그 중간에 네트워크 중단이 발생할 경우
    - 동일한 프로세스에서 실행되고 있지만 클라이언트가 서버로부터 ok 요청을 받기 전에 프로세스가 실패할 경우

위의 두 경우는 사용자가 실패를 했다고 생각하여, 다시 해당 메서드를 실행하게 되고 이 경우 카운터가 2번 증가한다.

이 방식을 개선하기 위해서, 작업을 멱등성으로 만들 필요가 있다.

1. 사용자에게 카운터 값을 전달하는 방법
해당 메서드를 사용하기 전에 클라이언트는 카운터 값을 미리 조회한다 -> 메서드를 호출하면서 로컬에서 값을 증가시키고, 해당 값을 파라미터로 메소드에 담아서 보낸다. -> 서버는 파라미터로 받은 값로 update한다.
[개선된 예시]
2. 낙관적 동시성 제어
메소드를 호출하기 전, 카운터에 현재 값을 읽고 파라미터에 해당 값을 담아서 보낸다 -> 서버는 클라이언트가 처음 읽은 값과 현재의 값이 동일한 경우에만 카운터 값을 업데이트 한다.

### 트랜잭션 스크립트를 사용하는 경우

- 트랜잭션 스크립트 패턴 장점
    - 매우 간단한 문제 도메인에 효과적 (단순 추출-변환-적재 와 같은 작업)
    - 따라서 지원 하위 도메인에 적합
    - 런타임 성능, 비즈니스 로직 이해 등을 최소화
- 트랜잭션 스크립트 패턴 단점
    - 비즈니스 로직이 복잡할 수 록 트랜잭션 간 로직이 쉽게 중복
        - 로직이 중복된다는건 같은 코드를 여러 번 써야함 -> 어떤 코드에서 만약 해당 로직을 빼놓는다면? -> 대참사
    - 따라서 핵심 하위 도메인에는 트랜잭션 스크립트를 사용하면 안됨

---

## 액티브 레코드

**액티브 레코드: 데이터베이스 테이블 또는 뷰의 행을 감싸고 데이터베이스 접근을 캡슐화하고 해당 데이터에 도메인 로직을 추가하는 오브젝트**

### 구현

- 액티브 레코드 라고 하는 전용 객체를 사용
- 자료구조, 레코드 생성, 읽기, 업데이트, 삭제 를 위한 데이터 접근 방법 구현
- ORM 및 데이터 접근 프레임워크와 연관성 있는 방식

### 특징

- 트랜잭션 스크립트가 db를 직접 접근하는 대신 액티브 레코드 객체를 조작
    - 데이터베이스에 대한 접근을 최적화하는 트랜잭션 스트립트
- 작업 완료 시 트랜잭션의 원자성으로 인해 작업이 성공 또는 실패 (all or nothing)
- 메모리 상의 객체를 데이터베이스 스키마에 매핑하는 복잡성을 숨길 수 있다.
    - 복잡한 자료구조를 데이터베이스 스키마에 매필하는 복잡성 해소
- 영속성 담당 부분 이외에 비즈니스 로직이 포함될 수 있다.
- 일반적으로 Getter와 Setter가 있다.

```csharp
public class CreateUser{

	public void Execute(userDetails){
		try{
			_db.StrartTransaction();

			var user = new User();
			user.Nmae = userDetails.Name;
			user.Email = userDetails.Email;
			user.Save();
		} catch{
			_db.RollBack();
			throw;
		}

	}
}

```

### 액티브 레코드를 사용하는 경우

- CRUD 와 같은 간단한 비즈니스 로직만 지원할 수 있다.
- 지원 하위 도메인, 일반 하위 도메인, 외부 솔루션 연동, 모델 변환 작업에 적합

—

## 실용적인 접근 방식

- 0.00001% 의 손실을 방지하는 것이 비즈니스 성과와 수익성에 얼마나 영향을 주는지
- 위험도 등을 파악해서 적절한 보호 정책을 펼치자