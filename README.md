# 🛒 쇼핑 리스트

Supabase를 백엔드로 쓰는 단일 파일 쇼핑 리스트 웹 앱. 빌드 도구도, 프레임워크도, 로컬 서버도 필요 없습니다.

## 실행

[`index.html`](index.html)을 브라우저로 열기만 하면 됩니다. 인터넷 연결이 필요합니다.

```bash
git clone https://github.com/a2632308-afk/shopping-listapp.git
cd shopping-listapp
start index.html      # Windows
open index.html       # macOS
```

처음 열면 **게스트로 바로 시작**됩니다. 로그인 없이도 쓸 수 있지만, 게스트 목록은 그 브라우저에만 남습니다.

여러 브라우저나 기기에서 같은 목록을 보려면 헤더의 **로그인 / 가입**을 눌러 이메일 계정을 만드세요. 게스트 상태에서 가입하면 **지금 담아둔 항목이 그대로 계정으로 옮겨집니다.**

## 기능

- **추가** — 입력창에 타이핑 후 Enter 또는 "추가" 버튼.
- **체크** — 체크박스나 항목 이름 아무 곳이나 클릭하면 토글됩니다. 완료 항목은 회색 취소선으로 표시됩니다.
- **삭제** — 각 항목 우측의 `×` 버튼.
- **완료 항목 비우기** — 체크된 항목이 있을 때만 하단에 나타나며, 완료된 항목만 한 번에 제거합니다.
- **요약** — 헤더에 `전체 / 담음 / 남음` 개수가 실시간 표시됩니다.
- **로그인 / 가입** — 이메일과 비밀번호로 계정을 만들면 어느 브라우저·기기에서든 같은 목록을 봅니다. 게스트로 담아둔 항목은 가입 시 계정으로 승계됩니다.
- **자동 이전** — 예전 localStorage 버전을 쓰던 브라우저라면, 첫 실행 때 기존 항목을 한 번만 데이터베이스로 옮깁니다.
- **다크 모드** — OS 설정을 따라 자동 전환됩니다.

모든 변경은 화면에 먼저 반영한 뒤 서버에 저장하며, 저장이 실패하면 이전 상태로 되돌리고 상단에 오류를 표시합니다.

## 데이터 구조

항목은 Supabase의 `shopping_items` 테이블에 저장됩니다.

| 컬럼 | 타입 | 비고 |
|---|---|---|
| `id` | `uuid` | 기본키, 자동 생성 |
| `user_id` | `uuid` | `auth.users` 참조, 기본값 `auth.uid()`, 사용자 삭제 시 함께 삭제 |
| `name` | `text` | 1–100자 제약 |
| `done` | `boolean` | 기본값 `false` |
| `created_at` | `timestamptz` | 정렬 기준 |

## 접근 제어

테이블에 **RLS(Row Level Security)가 켜져 있고**, 조회·삽입·수정·삭제 네 정책 모두 `auth.uid() = user_id` 조건을 겁니다. 즉 **각 사용자는 자기 항목만 읽고 쓸 수 있습니다.**

`index.html`에 들어 있는 `sb_publishable_...` 키는 브라우저에 공개되도록 설계된 값이라 저장소에 올려도 안전합니다. RLS가 실제 접근을 막습니다. **`service_role` 키나 `sb_secret_...` 키는 절대 클라이언트 코드에 넣으면 안 됩니다** — 이 키들은 RLS를 우회합니다.

### 게스트와 계정의 차이

| | 게스트 | 이메일 계정 |
|---|---|---|
| 로그인 | 불필요 | 이메일 + 비밀번호 |
| 다른 브라우저·기기에서 보기 | ❌ 각각 별개 목록 | ✅ 같은 목록 |
| 브라우저 데이터 삭제 시 | 목록에 접근 불가 | 로그인하면 복원 |

게스트는 Supabase의 익명 계정을 쓰며 브라우저 세션에 묶여 있습니다. 가입하면 그 익명 계정이 **영구 계정으로 승격**되므로 `user_id`가 유지되고 담아둔 항목도 그대로 남습니다.

이 프로젝트는 가입 시 확인 메일을 보내지 않도록 `mailer_autoconfirm`을 켜 두었습니다. Supabase 기본 SMTP는 프로젝트 멤버에게만, 시간당 소수만 발송할 수 있어 실사용에 맞지 않기 때문입니다. **따라서 이메일 주소는 검증되지 않습니다** — 실제 서비스로 쓰려면 외부 SMTP를 연결하고 이 옵션을 끄세요.

## 직접 배포하려면

포크해서 쓰려면 본인 Supabase 프로젝트가 필요합니다.

1. [Supabase](https://supabase.com)에서 프로젝트를 만듭니다.
2. SQL Editor에서 테이블과 정책을 만듭니다.

   ```sql
   create table public.shopping_items (
     id         uuid        primary key default gen_random_uuid(),
     user_id    uuid        not null default auth.uid() references auth.users(id) on delete cascade,
     name       text        not null check (char_length(name) between 1 and 100),
     done       boolean     not null default false,
     created_at timestamptz not null default now()
   );

   alter table public.shopping_items enable row level security;

   create policy "read own items"   on public.shopping_items for select using (auth.uid() = user_id);
   create policy "insert own items" on public.shopping_items for insert with check (auth.uid() = user_id);
   create policy "update own items" on public.shopping_items for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
   create policy "delete own items" on public.shopping_items for delete using (auth.uid() = user_id);

   create index shopping_items_user_created_idx on public.shopping_items (user_id, created_at);
   ```

3. Authentication → Providers에서 **Anonymous sign-ins**와 **Email**을 켭니다.
4. 확인 메일 없이 가입시키려면 Authentication → Providers → Email에서 **Confirm email**을 끕니다.
5. `index.html` 상단의 `SUPABASE_URL`과 `SUPABASE_PUBLISHABLE_KEY`를 본인 프로젝트 값으로 바꿉니다.

## 테스트

실제 Chrome을 자동 조작하는 Selenium 통합 테스트로 **51개 항목**(기능 31 + 인증 20)을 검증했으며 전부 통과했습니다. UI 동작뿐 아니라 **데이터베이스에 실제로 반영됐는지**까지 함께 확인합니다.

| 영역 | 검증 내용 |
|---|---|
| 로그인 / 가입 | 게스트 시작, 가입 시 목록 승계, 로그아웃, 재로그인 복원, 잘못된 비밀번호 거부 |
| 기기 간 동기화 | 세션을 완전히 지운 뒤 로그인하면 같은 목록이 보이는지 |
| 익명 인증 | 세션 발급, `is_anonymous` 확인, 초기 로드 |
| 추가 | Enter·버튼 양쪽 동작, 순서 유지, DB 저장 확인 |
| 빈 입력 방어 | 빈 값·공백만 입력 시 미추가 |
| 체크 | 체크박스·이름 클릭, 재클릭 해제, DB의 `done` 값 일치 |
| 삭제 | 지정 항목만 삭제, DB에서도 삭제 확인 |
| 완료 비우기 | 체크된 항목만 제거, 이후 버튼 자동 숨김 |
| 새로고침 | DB 재조회로 목록 복원, 동일 익명 사용자 유지 |
| RLS 격리 | 세션 교체 시 새 사용자 발급, 남의 항목 비노출 |
| 자동 이전 | localStorage 항목 이전, 완료 상태 유지, 중복 이전 방지 |
| HTML 주입 방어 | 태그 입력 시 텍스트로만 표시 |

사용자 입력은 `textContent`로 렌더링되므로 태그를 입력해도 문자 그대로 표시되고 DOM 요소가 생성되지 않습니다.

## 브라우저 지원

Chrome, Edge, Firefox, Safari 최신 버전에서 동작합니다. 다크 모드 테마에 `color-mix()`를 사용하므로 2023년 이전 브라우저에서는 일부 색상이 다르게 보일 수 있습니다.

## 라이선스

MIT
