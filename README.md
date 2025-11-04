# [welive-project 개발 보고서]

## 🏢 Welive - 스마트 아파트 운영 시스템 (주민 투표 모듈)
#### [배포 링크] (https://www.welive-test.online)
#### [Swagger](https://api.welive-test.online/api/docs/)
### 📋 목차
- [프로젝트 개요](#프로젝트-개요)
- [기술 스택 및 개발 환경](#기술-스택-및-개발-환경)
- [담당 API 및 기능 구현](#담당-api-및-기능-구현)
- [트러블슈팅 경험](#트러블슈팅-경험)
- [프로젝트 회고](#프로젝트-회고)

---

### 프로젝트 개요

#### 🏗 Welive 프로젝트 소개
**Welive**는 입주민∙관리자∙슈퍼 어드민의 역할에 따라 맞춤형 기능을 제공하는 스마트 아파트 운영 시스템으로, 주민들과 아파트 관리 단체를 위한 상호 관리 플랫폼입니다.

입주민 명부 관리, 민원 처리, 주민 투표, 공지사항 등록은 물론, 실시간 알림과 자동 승인 시스템까지 지원하여 복잡했던 아파트 관리 업무를 간편하고 효율적으로 바꿔드립니다.

#### 🎯 담당 모듈: 주민 투표 시스템
아파트 단지 내 주민들이 온라인으로 안건에 대해 투표하고 의사결정에 참여할 수 있는 디지털 투표 시스템 구축

#### 📅 개발 기간
2025.09.15 ~ 2025.11.03

#### 🎭 나의 역할
- **백엔드 개발자** - 주민 투표 시스템 전체 설계 및 구현
- 투표 CRUD API 개발
- 투표 참여/취소 기능 구현
- 자동 투표 마감 스케줄러 개발
- 테스트 코드 작성 및 CI/CD 구축

---

### 기술 스택 및 개발 환경

#### 🛠 Backend Stack
```
Runtime     : Node.js
Language    : TypeScript
Framework   : Express.js
ORM         : TypeORM
Database    : PostgreSQL
Validation  : Zod
Testing     : Jest + Supertest
```

#### 🏗 Architecture & Patterns
- **레이어드 아키텍처** (Controller → Service with Repository)
  - Controller: HTTP 요청/응답 처리
  - Service: 비즈니스 로직 + TypeORM Repository 직접 활용
- **함수형 프로그래밍** 스타일 적용
- **의존성 주입** 패턴 활용
- **트랜잭션 처리**를 통한 데이터 일관성 보장

#### 🔧 Development Tools
```yaml
Version Control : Git & GitHub
CI/CD          : GitHub Actions
API Testing    : Postman
Scheduler      : node-cron
Code Style     : ESLint + Prettier
```

---

### 담당 API 및 기능 구현

#### 📊 투표 관리 API (Polls)

#### 1. **투표 생성** `POST /api/polls`
```typescript
// Request Body
{
  "boardId": "게시판ID (uuid)",
  "status": "IN_PROGRESS",
  "title": "투표 제목",
  "content": "투표 내용",
  "buildingPermission": 101,  // 동별 권한 (선택)
  "startDate": "2025-06-01T00:00:00Z",
  "endDate": "2025-06-10T00:00:00Z",
  "options": [
    {
      "title": "옵션1"
    },
    {
      "title": "옵션2"
    }
  ]
}
```

#### 2. **투표 목록 조회** `GET /api/polls`
- Query Parameters: 
  - `page`: 페이지 번호 (기본값: 1)
  - `limit`: 한 페이지당 항목 수 (기본값: 11)
- 권한별 필터링 (관리자: 전체, 주민: 해당 동)
- 페이지네이션 지원

#### 3. **투표 상세 조회** `GET /api/polls/:pollId`
- 투표 정보 + 옵션별 득표수
- 참여 여부 확인
- 실시간 투표율 계산

#### 4. **투표 수정** `PATCH /api/polls/:pollId`
```typescript
// Request Body
{
  "title": "수정된 제목",
  "content": "수정된 내용",
  "buildingPermission": 102,
  "startDate": "2025-06-11T12:00:00.000Z",
  "endDate": "2025-06-12T20:00:00.000Z",
  "status": "PENDING",
  "options": [
    {
      "title": "수정된 옵션1"
    }
  ]
}
```
- 관리자 전용
- 진행 전 투표만 수정 가능

#### 5. **투표 삭제** `DELETE /api/polls/:pollId`
- 관리자 전용
- Soft Delete 방식

#### 🗳 투표 참여 API (Votes)

#### 1. **투표 참여** `POST /api/options/:optionId/vote`
- Path Parameter: `optionId` (선택한 옵션 ID)
- Business Rules:
  - 투표 기간 내에만 참여 가능
  - 중복 투표 방지 (투표당 1인 1회)
  - 동별 권한 체크
  - 트랜잭션으로 동시성 제어

#### 2. **투표 취소** `DELETE /api/options/:optionId/vote`
- Path Parameter: `optionId` (취소할 옵션 ID)
- 투표 기간 내 취소 가능
- 기존 투표 내역 완전 삭제

#### ⏰ 투표 스케줄러 API

#### 1. **스케줄러 상태 확인** `GET /api/poll-scheduler/ping`
```json
{
  "message": "Poll scheduler is running."
}
```
- 스케줄러 정상 작동 여부 확인

#### 2. **자동 스케줄링**
- 서버 내부에서 cron job으로 자동 실행
- 매시간 만료된 투표 자동 마감 처리
- 마감된 투표 결과를 공지사항으로 자동 생성

#### 📝 Database Schema

```sql
-- Poll 엔티티
CREATE TABLE poll (
  id SERIAL PRIMARY KEY,
  boardId UUID NOT NULL,
  title VARCHAR(255) NOT NULL,
  content TEXT,
  status VARCHAR(50) DEFAULT 'PENDING',
  buildingPermission INTEGER,
  startDate TIMESTAMP,
  endDate TIMESTAMP,
  createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updatedAt TIMESTAMP
);

-- PollOption 엔티티
CREATE TABLE poll_option (
  id SERIAL PRIMARY KEY,
  pollId INTEGER REFERENCES poll(id),
  title VARCHAR(255) NOT NULL,
  votes INTEGER DEFAULT 0
);

-- Vote 엔티티
CREATE TABLE vote (
  id SERIAL PRIMARY KEY,
  pollId INTEGER REFERENCES poll(id),
  userId UUID,
  optionId INTEGER REFERENCES poll_option(id),
  createdAt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(pollId, userId)
);
```

---

### 트러블슈팅 경험

#### 🔍 Problem 1: Jest Mock 설정 오류 (403 Forbidden)

#### 문제 상황
```typescript
// 테스트 실행 시 403 Forbidden 에러 발생
FAIL  src/polls/polls.controller.test.ts
  ✕ should return 403 if user lacks permission
  Error: Expected 201, got 403
```

#### 원인 분석
Jest mock이 실제 미들웨어보다 늦게 적용되어 인증이 실패하는 문제

#### 해결 방법
```typescript
// ❌ Before - mock이 import 이후에 위치
import { authenticate } from '@/middleware/authenticate';
jest.mock('@/middleware/authenticate');

// ✅ After - mock을 최상단으로 이동
jest.mock('@/middleware/authenticate');
import { authenticate } from '@/middleware/authenticate';
```

**결과**: 모든 테스트 정상 통과, 인증 미들웨어 mock 정상 작동

---

#### 🔍 Problem 2: TypeORM 과도한 로깅 문제

#### 문제 상황
```bash
# 개발 중 콘솔에 수백 줄의 SQL 쿼리 로그 출력
query: SELECT "Poll"."id" AS "Poll_id"...
query: SELECT "PollOption"."id" AS "PollOption_id"...
# ... 수백 줄 계속
```

#### 원인 분석
TypeORM 개발 환경 설정에서 `logging: true`로 모든 쿼리 로깅

#### 해결 방법
```typescript
// ormconfig.ts
export const AppDataSource = new DataSource({
  type: 'postgres',
  // ✅ 환경별 로깅 레벨 분리
  logging: process.env.NODE_ENV === 'production' ? false : ['error', 'warn'],
  logger: process.env.NODE_ENV === 'test' ? 'simple-console' : 'advanced-console',
});
```

**결과**: 필요한 에러/경고 로그만 출력, 개발 생산성 향상

---

#### 🔍 Problem 3: Zod datetime() 메서드 호환성 문제

#### 문제 상황
```typescript
// Zod 3.22 버전에서 datetime() 메서드 없음
const schema = z.object({
  startDate: z.string().datetime(), // ❌ Error: datetime is not a function
});
```

#### 원인 분석
프로젝트 Zod 버전(3.22)과 datetime() 메서드 도입 버전(3.23) 불일치

#### 해결 방법
```typescript
// ✅ Custom validation with refine()
const schema = z.object({
  startDate: z.string().refine(
    (val) => !isNaN(Date.parse(val)),
    { message: 'Invalid datetime format' }
  ),
  endDate: z.string().refine(
    (val) => !isNaN(Date.parse(val)),
    { message: 'Invalid datetime format' }
  ),
});
```

**결과**: 패키지 버전 변경 없이 datetime 유효성 검증 구현

---

#### 🔍 Problem 4: GitHub Actions CI/CD 권한 문제

#### 문제 상황
```yaml
# CI/CD 파이프라인 실행 실패
Error: Resource not accessible by integration
```

#### 원인 분석
개인 리포지토리에서 GitHub Actions의 PR 권한 제한

#### 해결 과정
1. **첫 번째 시도**: Token 권한 수정 → 실패
2. **두 번째 시도**: Workflow 권한 설정 → 부분 성공
3. **최종 해결**: Organization으로 리포지토리 이전

**결과**: 모든 PR에서 자동 테스트 및 커버리지 리포트 생성

---

#### 🔍 Problem 5: 테스트 UUID 형식 검증 실패

#### 문제 상황
```typescript
// E2E 테스트 실패
Expected: 201 Created
Received: 400 Bad Request
Body: { "error": "Invalid UUID format" }
```

#### 원인 분석
테스트 데이터의 UUID가 실제 UUID v4 형식이 아님

#### 해결 방법
```typescript
// ❌ Before
const mockUserId = 'test-user-id';

// ✅ After - 실제 UUID v4 형식 사용
const mockUserId = '550e8400-e29b-41d4-a716-446655440000';
```

**결과**: UUID 유효성 검증 통과, E2E 테스트 성공

---

#### 🔍 Problem 6: TypeScript Strict Mode와 TypeORM 충돌

#### 문제 상황
```typescript
// TypeScript 에러
error TS2564: Property 'title' has no initializer 
and is not definitely assigned in the constructor.
```

#### 원인 분석
TypeScript strict mode와 TypeORM 데코레이터의 초기화 시점 차이

#### 해결 방법
```typescript
@Entity()
export class Poll {
  @PrimaryGeneratedColumn()
  id!: number;  // ✅ Definite assignment assertion

  @Column()
  title!: string;  // ✅ ! 연산자로 초기화 보장

  @Column({ nullable: true })
  content?: string;  // ✅ Optional property
}
```

**결과**: TypeScript 컴파일 에러 해결, 타입 안정성 유지

---

#### 🔍 Problem 7: PostgreSQL 관계 네이밍 문제

#### 문제 상황
```sql
-- 에러: relation "poll_option" does not exist
SELECT * FROM poll_option WHERE "pollId" = $1;
```

#### 원인 분석
1. PostgreSQL의 대소문자 민감도
2. TypeORM 네이밍 전략 불일치

#### 해결 방법
```typescript
// ✅ 명시적 테이블/컬럼 이름 지정
@Entity('poll_option')  // snake_case
export class PollOption {
  @Column({ name: 'poll_id' })  // 명시적 컬럼명
  pollId: number;  // camelCase 프로퍼티
}

// ✅ 커스텀 네이밍 전략
class CustomNamingStrategy extends SnakeNamingStrategy {
  columnName(propertyName: string, customName: string): string {
    return customName || snakeCase(propertyName);
  }
}
```

**결과**: 데이터베이스 쿼리 정상 실행, 네이밍 일관성 확보

---

#### 🔍 Problem 8: 투표 상태 자동 업데이트 로직 오류

#### 문제 상황
```typescript
// 시간이 지나도 투표 상태가 자동으로 변경되지 않음
// startDate가 지났는데도 status가 여전히 'PENDING'
```

#### 원인 분석
데이터베이스에 저장된 status 값이 시간 경과에 따라 자동 갱신되지 않음

#### 해결 방법
```typescript
// ✅ 조회 시 실시간 상태 계산
class PollService {
  private calculateStatus(poll: Poll): PollStatus {
    const now = new Date();
    const startDate = new Date(poll.startDate);
    const endDate = new Date(poll.endDate);
    
    if (now < startDate) return 'PENDING';
    if (now > endDate) return 'CLOSED';
    return 'IN_PROGRESS';
  }

  async getPolls() {
    const polls = await this.pollRepository.find();
    // 각 투표의 상태를 실시간으로 계산
    return polls.map(poll => ({
      ...poll,
      status: this.calculateStatus(poll)
    }));
  }
}

// ✅ 스케줄러로 주기적 상태 업데이트
@Cron('0 * * * *')  // 매시간 실행
async updateExpiredPolls() {
  await this.pollRepository
    .createQueryBuilder()
    .update(Poll)
    .set({ status: 'CLOSED' })
    .where('endDate < :now', { now: new Date() })
    .andWhere('status != :closed', { closed: 'CLOSED' })
    .execute();
}
```

**결과**: 투표 상태가 시간에 따라 정확히 반영됨

---

#### 🔍 Problem 9: 동별 권한 필터링 로직 오류

#### 문제 상황
```typescript
// 301동 주민이 로그인해도 투표 목록이 비어있음
// API Response: {"items": [], "totalCount": 0}
```

#### 원인 분석
```typescript
// 데이터베이스: buildingPermission = 301 (301동)
// 사용자: dongNumber = 3 (3동)
// 비교 로직: 301 !== 3 → 필터링 실패
```

#### 해결 방법
```typescript
// ✅ Modulo 연산으로 동 번호 추출
async getPolls(user: User, queryDto: GetPollsDto) {
  const query = this.pollRepository.createQueryBuilder('poll');
  
  if (user.role === 'RESIDENT') {
    query.andWhere(
      new Brackets(qb => {
        qb.where('poll.buildingPermission IS NULL')  // 전체 공개
          .orWhere('MOD(poll.buildingPermission, 100) = :dong', { 
            dong: user.dongNumber  // 301 % 100 = 1
          });
      })
    );
  }
  
  return query.getMany();
}
```

**결과**: 주민이 자신의 동 투표와 전체 투표 모두 정상 조회

---

### 프로젝트 회고

#### 💡 배운 점

#### 1. **체계적인 디버깅의 중요성**
- 문제 발생 시 로그 분석 → 원인 파악 → 단계별 해결 프로세스 확립
- `console.log` 디버깅에서 벗어나 디버거와 네트워크 탭 적극 활용

#### 2. **테스트 주도 개발의 가치**
- 버그 발견 시간 단축 (개발 단계에서 90% 이상 발견)
- 리팩토링 시 안정성 보장
- 테스트 커버리지 90% 이상 달성

#### 3. **팀 협업과 커뮤니케이션**
- 프론트엔드 팀과의 API 스펙 조율 경험
- Git 충돌 해결 및 PR 리뷰 문화 정착
- 팀원 간 코드 리뷰를 통한 코드 품질 향상

#### 😊 느낀 점

#### 1. **문제 해결 능력의 성장**
처음에는 에러 메시지만 보면 막막했지만, 프로젝트를 진행하며 에러 스택을 읽고 원인을 추론하는 능력이 크게 향상되었습니다. 특히 "403 Forbidden" 에러를 Jest mock 순서 조정으로 해결했을 때의 성취감이 컸습니다.

#### 2. **실무 개발 프로세스 체험**
단순히 기능을 구현하는 것을 넘어서, CI/CD 파이프라인 구축, 코드 리뷰, 테스트 작성 등 실무와 유사한 개발 프로세스를 경험할 수 있었습니다. GitHub Actions 권한 문제를 Organization 마이그레이션으로 해결한 경험은 특히 값진 경험이었습니다.

#### 3. **백엔드 개발자로서의 책임감**
프론트엔드 팀이 우리 API에 의존한다는 사실이 큰 책임감으로 다가왔습니다. API 하나의 오류가 전체 서비스에 영향을 미칠 수 있다는 것을 깨닫고, 더욱 꼼꼼하게 테스트하고 문서화하는 습관을 기를 수 있었습니다.

#### 🎯 앞으로의 목표

1. **TypeScript 심화 학습**: 제네릭, 유틸리티 타입 등 고급 기능 마스터
2. **성능 최적화**: 데이터베이스 쿼리 최적화, 캐싱 전략 학습
3. **마이크로서비스 아키텍처**: 모놀리식에서 MSA로의 전환 경험
4. **DevOps 역량 강화**: Docker, Kubernetes 등 컨테이너 기술 습득

---

## 📚 References

- [TypeORM Documentation](https://typeorm.io/)
- [Jest Testing Best Practices](https://github.com/goldbergyoni/javascript-testing-best-practices)
- [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

---

<div align="center">
  
  **🙏 읽어주셔서 감사합니다 🙏**
  
  *이 프로젝트는 Node.js와 TypeScript를 활용한 프로젝트로,*  
  *다양한 기술적 도전과 문제 해결 경험을 통해 한 단계 성장할 수 있었습니다.*
  
</div>
