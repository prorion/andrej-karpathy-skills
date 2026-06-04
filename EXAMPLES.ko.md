# 예시

네 가지 원칙을 보여주는 실제 코드 예시입니다. 각 예시는 LLM이 흔히 잘못하는 것과 그것을 고치는 방법을 보여줍니다.

> [English](./EXAMPLES.md) | 한국어

---

## 1. 코딩 전에 생각하기

### 예시 1: 숨겨진 가정

**사용자 요청:** "사용자 데이터를 내보내는 기능을 추가해줘"

**❌ LLM이 하는 것 (잘못된 가정)**

```python
def export_users(format='json'):
    """Export all users to a file."""
    users = User.query.all()
    
    if format == 'json':
        with open('users.json', 'w') as f:
            json.dump([u.to_dict() for u in users], f)
    elif format == 'csv':
        with open('users.csv', 'w') as f:
            writer = csv.DictWriter(f, fieldnames=['id', 'email', 'name'])
            writer.writeheader()
            writer.writerows([u.to_dict() for u in users])
    
    return f"Exported {len(users)} users"
```

**문제점:**
- 모든 사용자를 내보내야 한다고 가정함 (페이지네이션? 개인정보는?)
- 묻지도 않고 파일 위치를 가정함
- 어떤 필드를 포함할지 가정함
- 실제 데이터 구조를 확인하지 않고 CSV 필드명을 가정함

**✅ 일어나야 하는 것 (가정을 드러내기)**

```
구현하기 전에, 다음을 명확히 해야 합니다:

1. **범위**: 모든 사용자를 내보낼까요, 아니면 필터링된 일부만? (개인정보 영향)
2. **형식**: "내보내기"라고 하셨는데, 다음 중 무엇을 의미하나요:
   - 브라우저에서 파일 다운로드?
   - 이메일 알림을 동반한 백그라운드 작업?
   - 데이터를 반환하는 API 엔드포인트?
3. **필드**: 어떤 사용자 필드를 포함하나요? (일부는 민감할 수 있음)
4. **규모**: 보통 사용자 수가 몇 명인가요? (접근 방식에 영향)

가장 단순한 접근: 페이지네이션된 JSON을 반환하는 API 엔드포인트 추가.
파일 기반 내보내기는 추가 정보가 필요합니다.

어떤 것을 선호하시나요?
```

### 예시 2: 여러 해석

**사용자 요청:** "검색을 더 빠르게 만들어줘"

**❌ LLM이 하는 것 (조용히 하나만 선택)**

```python
# 어느 "빠름"이 중요한지 묻지 않고
# 캐싱, 데이터베이스 인덱스, 비동기 처리를 추가함
@lru_cache(maxsize=1000)
async def search(query: str) -> List[Result]:
    # ... 200줄의 최적화 코드
```

**✅ 일어나야 하는 것**

```
"검색을 더 빠르게"는 다음을 의미할 수 있습니다:

1. **응답 시간 단축** (500ms 대신 < 100ms)
   - 데이터베이스 인덱스 추가
   - 자주 쓰는 쿼리 캐싱
   - 예상 소요: 2시간

2. **더 많은 동시 검색 처리** (처리량)
   - async/await 추가
   - 커넥션 풀링
   - 예상 소요: 4시간

3. **체감 속도 향상** (UX)
   - 부분 결과를 즉시 표시
   - 점진적 로딩
   - 예상 소요: 3시간

현재 검색은 일반 쿼리 기준 ~500ms 걸립니다.
당신의 사용 사례에는 어떤 측면이 가장 중요한가요?
```

---

## 2. 단순함 우선

### 예시 1: 과도한 추상화

**사용자 요청:** "할인을 계산하는 함수를 추가해줘"

**❌ LLM이 하는 것 (과잉 설계)**

```python
from abc import ABC, abstractmethod
from enum import Enum
from typing import Protocol, Union
from dataclasses import dataclass

class DiscountStrategy(ABC):
    @abstractmethod
    def calculate(self, amount: float) -> float:
        pass

class PercentageDiscount(DiscountStrategy):
    def __init__(self, percentage: float):
        self.percentage = percentage
    
    def calculate(self, amount: float) -> float:
        return amount * (self.percentage / 100)

class FixedDiscount(DiscountStrategy):
    def __init__(self, fixed_amount: float):
        self.fixed_amount = fixed_amount
    
    def calculate(self, amount: float) -> float:
        return min(self.fixed_amount, amount)

@dataclass
class DiscountConfig:
    strategy: DiscountStrategy
    min_purchase: float = 0.0
    max_discount: float = float('inf')

class DiscountCalculator:
    def __init__(self, config: DiscountConfig):
        self.config = config
    
    def apply_discount(self, amount: float) -> float:
        if amount < self.config.min_purchase:
            return 0
        discount = self.config.strategy.calculate(amount)
        return min(discount, self.config.max_discount)

# 단순한 계산 하나에 30줄 이상의 설정이 필요함
```

**✅ 일어나야 하는 것 (단순)**

```python
def calculate_discount(amount: float, percent: float) -> float:
    """Calculate discount amount. percent should be 0-100."""
    return amount * (percent / 100)

# 사용
discount = calculate_discount(100.0, 10.0)  # $10 off
```

**복잡성을 더해야 할 때:** 실제로 여러 할인 유형이 필요할 때만. 그 요구가 나중에 생기면, 그때 리팩토링하라.

### 예시 2: 투기적 기능

**사용자 요청:** "사용자 환경설정을 데이터베이스에 저장해줘"

**❌ LLM이 하는 것 (불필요한 기능)**

```python
class PreferenceManager:
    def __init__(self, db, cache=None, validator=None):
        self.db = db
        self.cache = cache or InMemoryCache()
        self.validator = validator or DefaultValidator()
    
    def save(self, user_id: int, prefs: dict, 
             merge: bool = True, 
             validate: bool = True,
             notify: bool = False) -> bool:
        """Save preferences with optional merging, validation, and notifications."""
        
        if validate:
            errors = self.validator.validate(prefs)
            if errors:
                raise ValidationError(errors)
        
        existing = self.get(user_id) if merge else {}
        
        if merge:
            existing.update(prefs)
            prefs = existing
        
        self.db.execute(
            "UPDATE users SET preferences = ? WHERE id = ?",
            (json.dumps(prefs), user_id)
        )
        
        if self.cache:
            self.cache.set(f"prefs:{user_id}", prefs)
        
        if notify:
            self.notify_preference_change(user_id, prefs)
        
        return True
    
    def notify_preference_change(self, user_id: int, prefs: dict):
        # 아무도 요청하지 않은 기능에 또 30줄
        pass
```

**✅ 일어나야 하는 것 (요청받은 것만)**

```python
def save_preferences(db, user_id: int, preferences: dict):
    """Save user preferences to database."""
    db.execute(
        "UPDATE users SET preferences = ? WHERE id = ?",
        (json.dumps(preferences), user_id)
    )
```

**필요하면 나중에 추가:** 캐싱(성능이 중요해질 때), 유효성 검사(잘못된 데이터가 나타날 때), 병합(요구가 생길 때).

---

## 3. 외과적 변경

### 예시 1: 지나가다 하는 리팩토링

**사용자 요청:** "빈 이메일이 검증기를 크래시시키는 버그를 고쳐줘"

**❌ LLM이 하는 것 (지나치게 많이)**

```diff
  def validate_user(user_data):
-     # Check email format
-     if not user_data.get('email'):
+     """Validate user data."""
+     email = user_data.get('email', '').strip()
+     
+     # Validate email
+     if not email:
          raise ValueError("Email required")
-     
-     # Basic email validation
-     if '@' not in user_data['email']:
+     if '@' not in email or '.' not in email.split('@')[1]:
          raise ValueError("Invalid email")
      
-     # Check username
-     if not user_data.get('username'):
+     # Validate username
+     username = user_data.get('username', '').strip()
+     if not username:
          raise ValueError("Username required")
+     if len(username) < 3:
+         raise ValueError("Username too short")
+     if not username.isalnum():
+         raise ValueError("Username must be alphanumeric")
      
      return True
```

**문제점:**
- 버그 수정을 넘어 이메일 검증을 "개선"함
- 아무도 요청하지 않은 사용자명 검증을 추가함
- 주석을 변경함
- docstring을 추가함

**✅ 일어나야 하는 것 (외과적)**

```diff
  def validate_user(user_data):
      # Check email format
-     if not user_data.get('email'):
+     email = user_data.get('email', '')
+     if not email or not email.strip():
          raise ValueError("Email required")
      
      # Basic email validation
-     if '@' not in user_data['email']:
+     if '@' not in email:
          raise ValueError("Invalid email")
      
      # Check username
      if not user_data.get('username'):
          raise ValueError("Username required")
      
      return True
```

**변경한 것은 오직:** 빈 이메일 처리를 고치는 특정 줄들뿐.

### 예시 2: 스타일 드리프트

**사용자 요청:** "업로드 함수에 로깅을 추가해줘"

**❌ LLM이 하는 것 (전부 재포맷)**

```diff
- def upload_file(file_path, destination):
+ def upload_file(file_path: str, destination: str) -> bool:
+     """Upload file to destination with logging."""
+     logger.info(f"Uploading {file_path} to {destination}")
+     
      try:
-         with open(file_path, 'rb') as f:
-             data = f.read()
-         
-         response = requests.post(destination, files={'file': data})
-         
-         if response.status_code == 200:
-             return True
-         else:
-             return False
+         with open(file_path, "rb") as f:
+             data = f.read()
+         response = requests.post(destination, files={"file": data})
+         success = response.status_code == 200
+         
+         if success:
+             logger.info(f"Upload successful: {file_path}")
+         else:
+             logger.error(f"Upload failed: {file_path}, status={response.status_code}")
+         
+         return success
      except Exception as e:
-         print(f"Error: {e}")
+         logger.exception(f"Upload error for {file_path}: {e}")
          return False
```

**문제점:**
- 따옴표 스타일 변경 ('' → "")
- 아무도 요청하지 않은 타입 힌트 추가
- docstring 추가
- 공백 재포맷
- 불리언 반환 로직 변경

**✅ 일어나야 하는 것 (기존 스타일에 맞추기)**

```diff
+ import logging
+ 
+ logger = logging.getLogger(__name__)
+ 
  def upload_file(file_path, destination):
+     logger.info(f'Starting upload: {file_path}')
      try:
          with open(file_path, 'rb') as f:
              data = f.read()
          
          response = requests.post(destination, files={'file': data})
          
          if response.status_code == 200:
+             logger.info(f'Upload successful: {file_path}')
              return True
          else:
+             logger.error(f'Upload failed: {file_path}, status={response.status_code}')
              return False
      except Exception as e:
-         print(f"Error: {e}")
+         logger.exception(f'Upload error: {file_path}')
          return False
```

**맞춘 것:** 작은따옴표, 타입 힌트 없음, 기존 불리언 패턴, 간격 스타일.

---

## 4. 목표 주도 실행

### 예시 1: 모호함 vs 검증 가능함

**사용자 요청:** "인증 시스템을 고쳐줘"

**❌ LLM이 하는 것 (모호한 접근)**

```
인증 시스템을 다음과 같이 고치겠습니다:
1. 코드 검토
2. 문제 식별
3. 개선 수행
4. 변경 테스트

[명확한 성공 기준 없이 변경을 진행함]
```

**✅ 일어나야 하는 것 (검증 가능한 목표)**

```
"인증을 고치려면" 성공 기준을 정의해야 합니다.
구체적으로 어떤 문제를 해결하나요?

문제가 "비밀번호 변경 후에도 사용자가 로그인 상태로 남는다"라면:

계획:
1. 테스트 작성: 비밀번호 변경 → 기존 세션 무효화 확인
   검증: 테스트 실패 (버그 재현)

2. 구현: 비밀번호 변경 시 세션 무효화
   검증: 테스트 통과

3. 엣지 케이스 확인: 다중 활성 세션, 동시 변경
   검증: 추가 테스트 통과

4. 회귀 없음 확인: 기존 인증 테스트가 여전히 통과
   검증: 전체 테스트 스위트 통과

현재 인증 테스트 커버리지: [커버리지 표시]
어떤 구체적 인증 문제를 겪고 계신가요?
```

### 예시 2: 검증을 동반한 다단계

**사용자 요청:** "API에 레이트 리미팅을 추가해줘"

**❌ LLM이 하는 것 (한꺼번에)**

```python
# Redis, 여러 전략, 구성 시스템, 모니터링을 포함한
# 완전한 레이트 리미팅을 하나의 300줄 커밋으로 구현
# 명확한 검증 단계 없음
```

**✅ 일어나야 하는 것 (검증을 동반한 점진적 진행)**

```
레이트 리미팅 계획:

1. 기본 인메모리 레이트 리미팅 추가 (단일 엔드포인트)
   검증:
   - 테스트: 100개 요청 → 처음 10개 성공, 나머지는 429
   - 수동: 엔드포인트에 curl 11번, 레이트 리밋 오류 확인

2. 미들웨어로 추출 (모든 엔드포인트에 적용)
   검증:
   - 테스트: /users와 /posts에 레이트 리밋 적용
   - 기존 엔드포인트 테스트가 여전히 통과

3. Redis 백엔드 추가 (다중 서버용)
   검증:
   - 테스트: 앱 재시작 후에도 레이트 리밋 유지
   - 테스트: 두 앱 인스턴스가 레이트 리밋 카운터 공유

4. 구성 추가 (엔드포인트별 비율)
   검증:
   - 테스트: /search는 10/분, /users는 100/분 허용
   - 구성 파일이 올바르게 파싱됨

각 단계는 독립적으로 검증 가능하고 배포 가능합니다.
1단계부터 시작할까요?
```

### 예시 3: 테스트 우선 검증

**사용자 요청:** "점수가 중복되면 정렬이 깨져"

**❌ LLM이 하는 것 (재현 없이 수정)**

```python
# 버그를 확인하지 않고 즉시 정렬 로직을 변경
def sort_scores(scores):
    return sorted(scores, key=lambda x: (-x['score'], x['name']))
```

**✅ 일어나야 하는 것 (먼저 재현)**

```python
# 1. 먼저, 문제를 재현하는 테스트를 작성
def test_sort_with_duplicate_scores():
    """Test sorting when multiple items have same score."""
    scores = [
        {'name': 'Alice', 'score': 100},
        {'name': 'Bob', 'score': 100},
        {'name': 'Charlie', 'score': 90},
    ]
    
    result = sort_scores(scores)
    
    # 버그: 중복에 대해 순서가 비결정적임
    # 이 테스트를 여러 번 실행해도 일관적이어야 함
    assert result[0]['score'] == 100
    assert result[1]['score'] == 100
    assert result[2]['score'] == 90

# 검증: 테스트를 10번 실행 → 일관되지 않은 순서로 실패

# 2. 이제 안정 정렬로 수정
def sort_scores(scores):
    """Sort by score descending, then name ascending for ties."""
    return sorted(scores, key=lambda x: (-x['score'], x['name']))

# 검증: 테스트가 일관되게 통과
```

---

## 안티패턴 요약

| 원칙 | 안티패턴 | 해결 |
|-----------|-------------|-----|
| 코딩 전에 생각하기 | 파일 형식, 필드, 범위를 조용히 가정 | 가정을 명시적으로 나열하고, 명확화 요청 |
| 단순함 우선 | 단일 할인 계산에 전략 패턴 | 복잡성이 실제로 필요해질 때까지 함수 하나로 |
| 외과적 변경 | 버그 수정 중 따옴표 재포맷, 타입 힌트 추가 | 보고된 문제를 고치는 줄만 변경 |
| 목표 주도 | "코드를 검토하고 개선하겠습니다" | "버그 X에 대한 테스트 작성 → 통과시키기 → 회귀 없음 확인" |

## 핵심 통찰

"과도하게 복잡한" 예시들은 명백히 틀린 것이 아닙니다 — 디자인 패턴과 모범 사례를 따릅니다. 문제는 **타이밍**입니다: 필요해지기 전에 복잡성을 더하며, 이는:

- 코드를 이해하기 어렵게 만들고
- 더 많은 버그를 유발하며
- 구현에 더 오래 걸리고
- 테스트하기 더 어렵습니다

"단순한" 버전은:
- 이해하기 쉽고
- 구현이 빠르고
- 테스트하기 쉽고
- 복잡성이 실제로 필요해질 때 나중에 리팩토링할 수 있습니다

**좋은 코드란 내일의 문제를 미리 푸는 것이 아니라, 오늘의 문제를 단순하게 푸는 코드입니다.**
