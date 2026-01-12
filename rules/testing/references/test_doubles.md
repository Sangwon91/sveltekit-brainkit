# Rule 3: Test Doubles (Mock vs Stub)

## 블라디미르 코리코프의 단순한 분류

복잡한 Test Double 분류(Fake, Stub, Mock, Spy)는 불필요합니다. **두 가지만** 기억하세요:

| 분류 | 목적 | 검증 방식 |
|------|------|-----------|
| **Stub** | 테스트 대상에 **입력(데이터)을 제공** | 상태(State) 검증 |
| **Mock** | 테스트 대상이 외부에 **출력(호출)했는지 확인** | 행위(Behavior) 검증 |

> [!IMPORTANT]
> **핵심 구분:** Stub은 **들어오는 의존성**(SUT가 데이터를 받음), Mock은 **나가는 의존성**(SUT가 외부에 작용)

## Stub: 입력을 위한 대역

Stub은 테스트 대상에게 데이터를 제공합니다. **어떻게 호출되었는지 검증하지 않습니다.**

```python
# ✅ Stub: 데이터 제공 (들어오는 의존성)
class FakeUserRepository:
    """인메모리 저장소 - Stub의 일종"""
    def __init__(self):
        self.users = {}
    
    def save(self, user):
        self.users[user.id] = user
        
    def get(self, user_id):
        return self.users.get(user_id)

# 테스트에서 사용
def test_get_user_profile():
    repo = FakeUserRepository()
    repo.users["1"] = User(id="1", name="Alice")
    
    service = UserService(repo)
    profile = service.get_profile("1")
    
    # ✅ 상태 검증: 결과만 확인
    assert profile.name == "Alice"
```

```typescript
// ✅ Stub: 데이터 제공 (들어오는 의존성)
const stubApi = {
  getUser: vi.fn().mockReturnValue({ id: '1', name: 'Alice' })
};

test('displays user name', () => {
  const result = getUserDisplay(stubApi);
  // ✅ 상태 검증: 결과만 확인
  expect(result).toBe('User: Alice');
  // ❌ 이렇게 하지 마세요: expect(stubApi.getUser).toHaveBeenCalled()
});
```

> [!CAUTION]
> **Stub에서 호출 검증하지 마세요!** Stub은 입력 제공용입니다. 호출 검증은 테스트를 구현 세부사항에 결합시켜 **리팩토링 내성을 파괴**합니다.

## Mock: 출력 검증을 위한 대역

Mock은 테스트 대상이 외부 시스템에 **올바른 부수 효과(side effect)를 발생시켰는지** 검증합니다.

```python
# ✅ Mock: 외부 호출 검증 (나가는 의존성)
def test_sends_welcome_email_on_signup(mocker):
    mock_email = mocker.Mock()
    mock_logger = mocker.Mock()  # 로깅도 외부 부수효과
    
    service = SignupService(email_sender=mock_email, logger=mock_logger)
    service.signup(email="user@example.com")
    
    # ✅ 행위 검증: 외부에 올바른 메시지를 보냈는지
    mock_email.send.assert_called_once_with(
        to="user@example.com",
        template="welcome"
    )
```

```typescript
// ✅ Mock: 외부 호출 검증 (나가는 의존성)
test('sends analytics on purchase', () => {
  const mockAnalytics = { track: vi.fn() };
  
  const service = new PurchaseService(mockAnalytics);
  service.complete({ orderId: '123', amount: 100 });
  
  // ✅ 행위 검증: 외부에 올바른 이벤트를 보냈는지
  expect(mockAnalytics.track).toHaveBeenCalledWith('purchase', {
    orderId: '123',
    amount: 100
  });
});
```

## 명확한 구분 가이드

```
테스트 대상(SUT)이 의존성에서 데이터를 받나요?
→ 들어오는 의존성 → Stub 사용 → 상태만 검증

테스트 대상(SUT)이 의존성에게 무언가를 시키나요?
→ 나가는 의존성 → Mock 사용 → 호출 검증
```

### 예시로 이해하기

| 시나리오 | 의존성 유형 | 사용할 것 | 검증 방식 |
|----------|-------------|-----------|-----------|
| DB에서 사용자 조회 | 들어오는 | **Stub** | 결과 상태 검증 |
| 외부 API에서 환율 조회 | 들어오는 | **Stub** | 결과 상태 검증 |
| 이메일 전송 | 나가는 | **Mock** | 호출 검증 |
| 로그 기록 | 나가는 | **Mock** | 호출 검증 |
| 결제 처리 (외부 서비스) | 나가는 | **Mock** | 호출 검증 |
| 분석 이벤트 전송 | 나가는 | **Mock** | 호출 검증 |

## 리팩토링 내성을 위한 핵심 원칙

> [!IMPORTANT]
> **Mock을 과용하면 리팩토링 내성이 파괴됩니다.** Mock은 **외부로 나가는 부수 효과**에만 사용하세요.

### ❌ 잘못된 사용: 내부 구현 검증

```python
# ❌ Anti-pattern: 내부 구현 세부사항 검증
def test_calculate_total_bad(mocker):
    mock_calculator = mocker.Mock()
    mock_calculator.add.return_value = 100
    
    service = OrderService(calculator=mock_calculator)
    result = service.get_total([10, 20, 30])
    
    # ❌ 내부적으로 add를 어떻게 호출하는지 검증
    # 리팩토링하면 테스트가 깨짐!
    mock_calculator.add.assert_called_with(10, 20)
```

### ✅ 올바른 사용: 최종 결과 검증

```python
# ✅ Good: 최종 결과만 검증
def test_calculate_total_good():
    service = OrderService()
    result = service.get_total([10, 20, 30])
    
    # ✅ 결과만 검증 - 내부 구현이 바뀌어도 테스트 통과
    assert result == 60
```

## 정리

| 원칙 | 설명 |
|------|------|
| **Stub = 입력** | 데이터 제공 용도. **호출 검증 금지** |
| **Mock = 출력** | 외부 부수효과 검증 용도. **결과에 영향 없는 내부 호출에는 사용 금지** |
| **가능하면 실제 객체 사용** | Test Double은 최소한으로. 빠르고 결정적인 의존성은 실제 객체 사용 |
| **리팩토링 내성 우선** | Mock 과용은 테스트를 구현에 결합시킴 |
