# 📦 Parsel 전수조사 & 활용 전략 리포트 (한국어)

> 이 문서는 Parsel 저장소를 전체 분석하고, 설치·사용법·정체성·수익화 전략까지
> 정리한 종합 리포트입니다.

---

## 🔗 참고 링크

| 항목 | 주소 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/parsel |
| **원본 저장소 (upstream)** | https://github.com/shipfastlabs/parsel |
| Packagist (Composer) | https://packagist.org/packages/shipfastlabs/parsel |
| LiteParse (LlamaIndex) | https://github.com/run-llama/liteparse |
| AnyDoc (Firecrawl) | https://github.com/firecrawl/anydoc |
| 메인테이너 | https://github.com/pushpak1300 (Shipfastlabs) |

### 원본 저장소 스탯 (2026-09-29 기준)

| 지표 | 값 |
|---|---|
| ⭐ Stars | **363** |
| 🍴 Forks | 10 |
| 📅 생성일 | 2026-05-29 (약 4개월) |
| 🔄 최근 푸시 | 2026-09-19 |
| 🏷️ 토픽 | docs, ocr, pdf, text-extraction |
| 📜 라이선스 | MIT |
| 💻 언어 | PHP |

> 4개월 만에 363스타 = 월평균 약 90스타. PHP 니치 라이브러리 기준 매우 빠른 성장세.

---

## 1. 한 줄 정의

> **Parsel은 PDF·DOCX·XLSX 같은 문서를 AI가 읽을 수 있는 텍스트(Markdown)로 바꿔주는 PHP 라이브러리다.**

정확히는 **직접 파싱하지 않고, 외부에 설치된 고성능 파싱 CLI 도구를 PHP에서 호출하는 "리모컨"** 역할을 한다.

```
PHP 코드  →  Parsel (통역/리모컨)  →  lit / anydoc (실제 파서)  →  깔끔한 텍스트
```

---

## 2. 폴더 구조 전체 지도

```
parsel/
├── README.md / UPGRADE.md / CHANGELOG.md   # 문서 3종
├── CLAUDE.md                                # 프로젝트 페르소나 설정 (Parsel 본체와 무관)
├── composer.json                            # PHP 8.4+, 의존성 3개
├── bin/
│   ├── parsel-install                       # 파서 바이너리 자동 설치기
│   └── parsel-install-lit                   # 구버전 호환 별칭
├── src/                                     # 본체 35개 파일
│   ├── Parsel.php                           # 정적 진입점 (Facade)
│   ├── ParselManager.php                    # 드라이버 공장 (Manager 패턴)
│   ├── Parser.php                           # 드라이버 + 소스 연결
│   ├── PendingParse.php                     # 체이닝 빌더 (핵심)
│   ├── Source.php                           # 입력 추상화 (경로 / 바이트)
│   ├── ParseRequest.php                     # 실행 직전 DTO
│   ├── Contracts/   (8)                     # 인터페이스 = 확장 포인트
│   ├── Drivers/     (2)                     # LiteParseDriver, AnyDocDriver
│   ├── Data/        (4)                     # Document → Page → TextItem, Cast
│   ├── Options/     (2)                     # LiteParseOptions, AnyDocOptions
│   ├── Enums/       (2)                     # ImageMode, OutputFormat
│   ├── Exceptions/  (11)                    # 세분화된 예외
│   └── Support/     (7)                     # 프로세스 실행, 바이너리 탐색, FS
├── tests/    (24개)                          # Pest 기반, 커버리지 100% 강제
├── examples/ (5개 스크립트 + 샘플문서 4개)
└── .github/workflows/                       # CI 2종 (Tests, Formats)
```

---

## 3. 동작 원리

```php
$markdown = Parsel::file('report.pdf')->markdown();
```

이 한 줄의 내부 흐름:

```
① Parsel::file()
   └→ ParselManager가 'liteparse' 드라이버 생성 (지연 로딩 + 캐싱)
② ->file('report.pdf')
   └→ Source::fromPath()로 입력 포장
③ ->markdown()
   └→ ParseRequest 생성 → LiteParseDriver::markdown()
④ LiteParseDriver 내부
   ├→ BinaryResolver: 'lit' 실행파일 탐색
   ├→ CliArguments: 명령어 배열 조립
   │    ['lit', 'parse', 'report.pdf', '--format', 'markdown', '-q', '--no-ocr']
   └→ CliProcess → Symfony Process로 실제 실행
⑤ stdout을 trim해서 반환
```

---

## 4. 드라이버 아키텍처

Laravel의 `Storage::disk('s3')` / `Cache::store('redis')` 패턴과 동일.

### 드라이버 능력 비교

| 능력 | LiteParse (`lit`) | AnyDoc (`anydoc`) |
|---|---|---|
| Markdown | ✅ | ✅ |
| 순수 텍스트 | ✅ | ❌ |
| 구조화 + 좌표 | ✅ | ❌ |
| 페이지 스트리밍 (lazy) | ✅ | ❌ |
| 스크린샷 | ✅ | ❌ |
| OCR | ✅ | ❌ |
| 제작사 | LlamaIndex | Firecrawl |

### 능력별 인터페이스 분리 (ISP 원칙)

```php
interface Driver {                          // 최소 계약 = markdown만
    public function name(): string;
    public function validateOptions(array $options): void;
    public function markdown(ParseRequest $r): string;
}

interface TextDriver               extends Driver {}   // 선택
interface StructuredDocumentDriver extends Driver {}   // 선택
interface LazyPageDriver           extends Driver {}   // 선택
interface ScreenshotDriver         extends Driver {}   // 선택
```

지원하지 않는 기능 호출 시 **프로세스 실행 전에** 차단:

```php
Parsel::driver('anydoc')->file('doc.pdf')->text();
// 💥 UnsupportedCapabilityException
```

> README 명시: *"Parsel does not derive fake structured data or plain text from AnyDoc Markdown."*
> → 없으면 없다고 하지, 가짜로 만들어내지 않는다.

---

## 5. 주요 기능

### 입력 2가지

```php
Parsel::file('/path/report.pdf');        // 파일 경로
Parsel::bytes($uploadedBytes, 'pdf');    // 바이트 (확장자 필수)
```

바이트 입력은 임시파일로 떨궜다가 `finally`로 반드시 삭제된다.

### 출력 5가지

```php
->markdown()       // AI 입력용 최적
->text()           // 순수 텍스트 ("--- Page N ---" 구분자 자동 제거)
->parse()          // Document 객체 (페이지/좌표 전부)
->toArray()        // 배열
->save('out.md')   // 확장자 보고 포맷 자동 선택
```

### 구조화 데이터 (Document → Page → TextItem)

```php
$doc = Parsel::file('invoice.pdf')->parse();

foreach ($doc->pages as $page) {           // number, width, height, text, items
    foreach ($page->items as $item) {      // TextItem
        echo "{$item->text} @ ({$item->x}, {$item->y})";
        echo " font={$item->fontName} size={$item->fontSize} conf={$item->confidence}";
    }
}
```

**좌표가 나온다는 것이 킬러 기능.**
같은 y좌표 = 같은 줄 → "총액" 라벨 오른쪽 숫자 = 금액, 식의 **위치 기반 필드 추출**이 가능하다.
인보이스/영수증/계약서 자동화의 핵심.

### 대용량 스트리밍

```php
foreach (Parsel::file('1000p.pdf')->lazyPages() as $page) {
    echo $page->text;   // 한 페이지씩만 메모리에
}
```

내부에서 `halaxa/json-machine`으로 `/pages` 포인터를 따라 JSON 스트리밍 파싱.
1000페이지 JSON을 통째로 `json_decode` 하지 않으므로 메모리 안전.

### 타입세이프 옵션

```php
LiteParseOptions::make()
    ->pageRange(1, 5)->page(10)              // 누적됨 → "1-5,10"
    ->maxPages(20)
    ->withOcr(language: 'kor+eng', workers: 8)
    ->withDpi(300)
    ->preserveSmallText()
    ->withPassword('1234')
    ->withImages(ImageMode::Embed, '/img')
    ->withoutLinks()
    ->keepHeadersAndFooters()
    ->option('new-upstream-flag', 42);       // 탈출구
```

- 오타 → `InvalidProviderOptionsException`
- 드라이버 불일치 (liteparse 옵션을 anydoc에) → 예외

### 바이너리 탐색 3단계

```
1순위: ->withBinary('/usr/local/bin/lit')
2순위: PARSEL_LITEPARSE_BINARY 환경변수 (구 PARSEL_LIT_BINARY 폴백)
3순위: PATH에서 'lit' 검색
  ↓ 실패
💥 BinaryNotFoundException
```

### 테스트용 Fake

```php
$fake = Parsel::fake([
    '--format json' => file_get_contents('fixtures/output.json'),
    'anydoc'        => '# Converted document',
]);

Parsel::file('invoice.pdf')->parse();
expect($fake->ranCount())->toBe(1);
```

명령어 문자열 **부분매칭**으로 가짜 응답 반환 → 바이너리 없이도 CI에서 100% 커버리지 달성.

### 커스텀 드라이버 확장

```php
Parsel::extend('company-api', fn (ParselManager $m): Driver => new CompanyApiDriver);
Parsel::driver('company-api')->file('report.pdf')->markdown();
```

CLI뿐 아니라 **HTTP API 드라이버**도 가능 → Upstage, CLOVA, Google Document AI 등 연결 가능.

---

## 6. 품질 관리 수준

| 검사 | 명령 | 기준 |
|---|---|---|
| 코드 스타일 | `pint --test` | Laravel 스타일 강제 |
| 정적 분석 | `phpstan` | 최고 레벨 |
| 타입 커버리지 | `pest --type-coverage --min=100` | 100% 미만 실패 |
| 유닛 테스트 | `pest --coverage --exactly=100` | **정확히 100%** |
| 리팩터 검사 | `rector --dry-run` | 개선 여지 0 |

CI 매트릭스: **Ubuntu × macOS × Windows × (prefer-lowest / prefer-stable)** = 6조합.

### 코드에서 확인된 "잘 만든 티"

1. 전 파일 `declare(strict_types=1)`
2. `final readonly class` 적극 사용 (불변 객체)
3. PHP 8.4 신문법 (`private const array`, 생성자 프로퍼티 승격, `match(true)`)
4. 예외 11종 세분화 — 전부 `ParselException` 상속
5. `finally` 정리 철저 → 임시파일 잔존 없음
6. snake_case/camelCase 양쪽 대응 (`Cast::pick($raw, ['textItems','text_items'])`)
7. DI 친화적 — 정적 Facade와 `new ParselManager` 인스턴스 방식 모두 지원

---

## 7. 설치 및 사용법

### ⚠️ 설치는 2단계

```
1단계: PHP 패키지 설치 (Composer)   ← "리모컨"
2단계: 파서 바이너리 설치 (npm 등)  ← "TV"  ← 빼먹으면 BinaryNotFoundException
```

### 사전 준비물

| 항목 | 요구사항 | 확인 |
|---|---|---|
| PHP | **8.4 이상** | `php -v` |
| ext-json | 기본 포함 | `php -m \| grep json` |
| Composer | 최신 | `composer -V` |
| Node.js | 20+ (AnyDoc만) | `node -v` |
| 프로세스 실행 | `proc_open` 허용 | 공유호스팅 ❌ |

### 1단계 — Composer

```bash
composer require shipfastlabs/parsel
```

### 2단계 — 바이너리

```bash
vendor/bin/parsel-install                                  # LiteParse (기본)
vendor/bin/parsel-install --driver=anydoc                  # AnyDoc
vendor/bin/parsel-install --driver=all                     # 둘 다
vendor/bin/parsel-install --driver=liteparse --manager=cargo
vendor/bin/parsel-install --driver=liteparse --with-system-dependencies
```

**설치기 동작**
- 이미 설치돼 있으면 건너뜀
- `--manager` 생략 시 npm → pnpm → bun → pip → cargo 순 자동 탐색
- `--with-system-dependencies`는 OS 감지 후 LibreOffice + ImageMagick 설치
  - macOS: `brew install --cask libreoffice` / `brew install imagemagick`
  - Ubuntu: `sudo apt-get install -y libreoffice imagemagick`
  - Windows: `choco install libreoffice-fresh imagemagick.app`

**드라이버별 지원 매니저**

| | npm | pnpm | bun | pip | cargo |
|---|---|---|---|---|---|
| LiteParse (`@llamaindex/liteparse`) | ✅ | ✅ | ✅ | ✅ | ✅ |
| AnyDoc (`@firecrawl/anydoc`) | ✅ | ✅ | ✅ | ❌ | ❌ |

### 3단계 — 확인

```bash
lit --version
anydoc --version
```

### 사용법 레벨별

**Level 1 — 기본**

```php
require 'vendor/autoload.php';
use Shipfastlabs\Parsel;

echo Parsel::file('report.pdf')->markdown();
echo Parsel::file('report.pdf')->text();
```

**Level 2 — 옵션**

```php
use Shipfastlabs\Parsel\Options\LiteParseOptions;
use Shipfastlabs\Parsel\Enums\ImageMode;

$md = Parsel::file('scan.pdf')
    ->withProviderOptions(
        LiteParseOptions::make()
            ->pageRange(1, 10)->page(15)
            ->withOcr(language: 'kor+eng', workers: 8)
            ->withDpi(300)
            ->preserveSmallText()
            ->withImages(ImageMode::Embed, '/img')
    )
    ->withTimeout(180)
    ->markdown();
```

**Level 3 — 웹 업로드 (Laravel)**

```php
public function upload(Request $request)
{
    $file = $request->file('document');

    return response()->json([
        'markdown' => Parsel::bytes(
            $file->get(),
            $file->getClientOriginalExtension()
        )->markdown(),
    ]);
}
```

**Level 4 — 좌표 추출 + 에러 처리**

```php
use Shipfastlabs\Parsel\Exceptions\{ParselException, BinaryNotFoundException, UnsupportedCapabilityException};

try {
    $doc = Parsel::file('invoice.pdf')->parse();

    foreach ($doc->pages[0]->items as $item) {
        if (str_contains($item->text, '합계')) {
            $amount = array_filter($doc->pages[0]->items,
                fn($i) => abs($i->y - $item->y) < 3 && $i->x > $item->x);
            echo reset($amount)->text;
        }
    }
} catch (BinaryNotFoundException $e) {
    // lit 미설치
} catch (UnsupportedCapabilityException $e) {
    // 드라이버 미지원 기능
} catch (ParselException $e) {
    // 그 외 전부
}
```

**Level 5 — DI (Octane/Swoole 등 상주형 앱 권장)**

```php
$this->app->singleton(ParselManager::class, fn() => new ParselManager(
    default: config('parsel.driver', 'liteparse'),
    timeout: 120.0,
));
```

### Docker 예시

```dockerfile
FROM php:8.4-cli

RUN apt-get update && apt-get install -y \
    nodejs npm libreoffice imagemagick \
 && npm install -g @llamaindex/liteparse @firecrawl/anydoc \
 && rm -rf /var/lib/apt/lists/*

ENV PARSEL_LITEPARSE_BINARY=/usr/local/bin/lit
ENV PARSEL_ANYDOC_BINARY=/usr/local/bin/anydoc

COPY . /app
WORKDIR /app
RUN composer install --no-dev --optimize-autoloader
```

---

## 8. 정체성 — 플러그인? 스킬? MCP?

### 정답: **셋 다 아님. Composer 라이브러리(PHP 패키지)다.**

| 종류 | 무엇 | 실행 위치 | 설치 | Parsel? |
|---|---|---|---|---|
| **Composer 라이브러리** | PHP 코드 모음 | PHP 앱 내부 | `composer require` | ✅ **이것** |
| Claude 플러그인 | Claude Code 기능 확장 | Claude Code | `/plugin` | ❌ |
| Claude 스킬 | Claude용 전문 지식 문서 | Claude Code | `.claude/skills/` | ❌ |
| MCP 서버 | AI ↔ 외부도구 표준 연결 | 별도 프로세스 | MCP 설정 | ❌ |
| CLI 도구 | 터미널 명령어 | 셸 | npm/brew | ⚠️ `lit`/`anydoc`이 해당 |

```
┌──────────────────────────────────────┐
│      PHP 웹 애플리케이션              │
│  ┌────────────────────────────────┐  │
│  │  Parsel (Composer 라이브러리)   │  │  ← Parsel의 위치
│  └──────────────┬─────────────────┘  │
└─────────────────┼────────────────────┘
                  │ proc_open
                  ▼
         ┌─────────────────┐
         │  lit / anydoc   │  ← 외부 CLI 프로그램
         └─────────────────┘
```

> 참고: 저장소 루트의 `CLAUDE.md`는 이 포크에서 추가한 페르소나 설정 파일이며,
> Parsel 라이브러리 기능과는 무관하다.

**다만** — Parsel을 감싸 MCP 서버로 만들면 Claude가 직접 PDF를 읽게 할 수 있다.
(수익화 아이디어 3번 참고)

---

## 9. API 토큰 필요 여부

### 정답: **불필요. 완전 무료, 100% 로컬.**

`src/` 전체에 HTTP 클라이언트 코드가 **단 한 줄도 없음**. 의존성은 3개뿐:

- `ext-json` (PHP 기본)
- `halaxa/json-machine` (JSON 스트리밍)
- `symfony/process` (프로세스 실행)

→ Guzzle/curl 없음 = **네트워크 미사용**.

### 로컬 vs 클라우드 API 비교

| 항목 | Parsel (로컬) | 클라우드 API |
|---|---|---|
| 비용 | **0원** | 페이지당 3~30원 |
| 월 10만 페이지 | **0원** | 30만~300만원 |
| 속도 | 네트워크 지연 없음 | 왕복 지연 |
| 호출 제한 | 없음 | Rate limit |
| 데이터 유출 위험 | **없음** | 외부 전송됨 |
| 오프라인 | 가능 | 불가 |
| 정확도 | 도구 성능 의존 | 대체로 더 높음 |

### 특히 유리한 분야

- **의료** — 진료기록 외부 전송 불가
- **법률** — 계약서 기밀 유지
- **금융** — 개인정보보호법 / 망분리 규제
- **공공기관** — 망분리 환경

### 토큰이 필요해지는 경우

1. OCR 서버를 따로 둘 때 (`withOcr(serverUrl: ...)`) — 보통 내부망이라 불필요
2. 직접 원격 드라이버를 만들 때 — 사용자 선택 사항

---

## 10. GitHub에서 유명한 이유

| # | 이유 | 설명 | 기여도 |
|---|---|---|---|
| 1 | **타이밍** | AI/RAG 붐 + PHP 생태계의 명백한 공백 | ⭐⭐⭐⭐⭐ |
| 2 | **Laravel 친화 API** | `Storage::disk()`, `Http::fake()`와 동일 감성 → 학습비용 0 | ⭐⭐⭐⭐⭐ |
| 3 | **코드 품질** | 커버리지 100% 강제, 6조합 CI, 배지 3종 | ⭐⭐⭐⭐⭐ |
| 4 | **브랜딩** | "Parsel" = 해리포터 파셀텅(뱀의 언어) = 못 읽는 언어를 읽음 | ⭐⭐⭐⭐ |
| 5 | **후광 효과** | LlamaIndex(LiteParse) + Firecrawl(AnyDoc) 브랜드 연결 | ⭐⭐⭐⭐ |
| 6 | **문서화** | 복붙 가능 예제 30개+, UPGRADE.md, examples/ 5종 | ⭐⭐⭐⭐ |
| 7 | **좁고 명확한 범위** | "PHP에서 PDF 읽기" 하나 → 이해 빠름, 도입 장벽 낮음 | ⭐⭐⭐⭐ |

---

## 11. 로컬 AI 에이전트 구축에 도움 되는가

### 결론: **매우 도움됨. 단, 전체 파이프라인의 "한 조각".**

```
🤖 로컬 AI 에이전트 파이프라인
 1️⃣ 문서 수집
 2️⃣ 파싱        ← ⭐ Parsel 담당
 3️⃣ 청킹
 4️⃣ 임베딩       ← Ollama
 5️⃣ 벡터 저장    ← pgvector / Qdrant
 6️⃣ 유사도 검색
 7️⃣ LLM 응답    ← Ollama / llama.cpp
```

> **Garbage In, Garbage Out** — 2번에서 텍스트가 깨지면 3~7번이 아무리 좋아도 소용없다.
> RAG 프로젝트 실패 원인 1위가 파싱 품질 문제다.

### 로컬 에이전트에 특히 좋은 이유

1. **완전 오프라인** — Parsel + Ollama + pgvector 전부 로컬 → 외부 통신 0
2. **좌표 정보** — 폰트 크기로 제목/본문 구분 → **제목 단위 청킹** → 답변 정확도 상승
   - 출처 하이라이팅("12페이지 상단")까지 가능
3. **lazyPages** — 수천 페이지 배치 인덱싱을 메모리 안전하게
4. **screenshots** — 표/그래프/도면을 이미지로 → Vision 모델(LLaVA)에 전달 (멀티모달)
5. **extend()** — 평소 무료 로컬, 중요 문서만 유료 API로 폴백 → 비용 최적화

### 샘플 아키텍처

```php
class LocalAgent
{
    public function ingest(string $path): void
    {
        foreach (Parsel::file($path)->lazyPages() as $page) {        // 1) 로컬 파싱
            foreach ($this->smartChunk($page) as $chunk) {           // 2) 제목 기준 청킹
                $vector = $this->ollama->embeddings('nomic-embed-text', $chunk);  // 3) 로컬 임베딩
                DB::table('chunks')->insert([                        // 4) pgvector
                    'text' => $chunk, 'embedding' => $vector,
                    'page' => $page->number, 'source' => $path,
                ]);
            }
        }
    }

    public function ask(string $question): string
    {
        $qVec = $this->ollama->embeddings('nomic-embed-text', $question);
        $ctx  = implode("\n---\n", array_column($this->searchSimilar($qVec, 5), 'text'));

        return $this->ollama->generate('llama3',
            "다음 문서를 참고해서 답해줘:\n{$ctx}\n\n질문: {$question}");
    }
}
```

### 한계

| 한계 | 대응 |
|---|---|
| 한글 OCR 정확도가 상용 대비 낮을 수 있음 | `kor` tessdata 설치, DPI 300+ |
| 복잡한 표는 Markdown 변환 시 깨질 수 있음 | 표만 스크린샷 → Vision 모델 |
| 임베딩/LLM은 별도 구축 필요 | Ollama 조합 |
| PHP 벡터DB 라이브러리 부족 | pgvector(SQL 직접) / Qdrant REST |

---

## 12. React / PHP로 만들 수 있는가

### 🐘 PHP — ✅ 가능 (Parsel 자체가 PHP)

직접 만든다면 이 구조를 따르면 된다:

```php
interface ProcessRunner { public function run(array $cmd, ?string $in, ?float $t): ProcessResult; }
interface Driver        { public function name(): string; public function markdown(ParseRequest $r): string; }
class MyManager         { public function driver(?string $n = null): Parser; public function extend(string $n, Closure $f): self; }
```

> 다만 바퀴를 다시 만들기보다 **Parsel을 `extend()`로 확장**하는 편이 훨씬 효율적이다.
> MIT 라이선스라 상업적 사용도 자유롭다.

### ⚛️ React — ⚠️ 절반만 가능

**불가능**: 브라우저는 보안 샌드박스 때문에 `proc_open`, 파일시스템 접근, 바이너리 실행이 모두 막혀 있다.

**가능한 3가지 방법**

**방법 1 — React(프론트) + PHP(백엔드) ⭐ 권장**

```jsx
const upload = async (file) => {
  const fd = new FormData();
  fd.append('document', file);
  const res = await fetch('/api/parse', { method: 'POST', body: fd });
  setMd((await res.json()).markdown);
};
```

```php
Route::post('/api/parse', fn (Request $r) => response()->json([
    'markdown' => Parsel::bytes($r->file('document')->get(),
                                $r->file('document')->getClientOriginalExtension())->markdown(),
]));
```

**방법 2 — 브라우저 직접 처리 (JS 라이브러리)**

| 라이브러리 | 용도 |
|---|---|
| `pdf.js` | PDF 텍스트 + 좌표 (Mozilla) |
| `mammoth.js` | DOCX → HTML |
| `xlsx` (SheetJS) | 엑셀 |
| `tesseract.js` | OCR (WASM, 느림) |

장점: 서버 비용 0, 파일이 서버로 안 감 / 단점: OCR 느림, 대용량 불리, 기능 제한

**방법 3 — Node.js 백엔드 (`parsel-js` 직접 제작)**

```javascript
class LiteParseDriver {
  name() { return 'liteparse'; }
  async markdown({ file }) {
    const { stdout } = await execFileAsync('lit',
      ['parse', file, '--format', 'markdown', '-q', '--no-ocr']);
    return stdout.trim();
  }
}
```

> PHP판이 4개월에 363스타인 점을 고려하면, npm 생태계에서의 `parsel-js`는 좋은 오픈소스 아이템이 될 수 있다.

### 권장 스택

```
React (Inertia) + Laravel + Parsel + Ollama + pgvector
= 완전 로컬 문서 AI 서비스
```

---

## 13. 수익화 아이디어

### 시장 지도

```
문서 파싱 시장 = 성장 중
 ├─ AI/RAG 붐 → 모든 기업이 사내 문서 챗봇 수요
 ├─ 기존 강자: Python (LangChain, Unstructured)
 ├─ 빈틈: PHP/Laravel 생태계  ← Parsel의 자리
 └─ 더 큰 빈틈: 🇰🇷 한국 시장 (한글 OCR, HWP, 국내 양식)
```

전략의 핵심 = **"PHP × 한국 × AI" 삼중 빈틈 공략**

---

### 🟢 티어 1 — 오픈소스 (비용 0, 신뢰 구축)

#### 아이디어 1. `parsel-laravel` — Laravel 통합 패키지

Parsel은 순수 PHP라 ServiceProvider, Facade, 설정파일, 큐 잡이 없다. → 명백한 빈틈.

```php
// config/parsel.php + Facade + 큐잡 + 이벤트 + Eloquent 트레이트 + Artisan 명령
class Contract extends Model {
    use HasParsedContent;
    protected $parsable = 'file_path';   // 자동 파싱 + 캐싱
}
$contract->parsedContent;

php artisan parsel:parse storage/docs/report.pdf --format=md
php artisan parsel:check
```

**수익 경로**: 오픈소스 → 인지도 → 컨설팅(시간당 10~20만원) / 기업 커스터마이징(500~3000만원)
/ GitHub Sponsors / 강의·책 / 취업·이직 가치 상승

| 기간 | 난이도 | 초기비용 | 직접수익 | 간접가치 |
|---|---|---|---|---|
| 2~4주 | ⭐⭐ | 0원 | 낮음 | ⭐⭐⭐⭐⭐ |

---

#### 아이디어 2. `parsel-korean` — 한국형 드라이버 팩 🇰🇷

LiteParse/AnyDoc은 영어권 최적화라 한글 OCR이 약하다. 한국 기업 문서(HWP, 한글 스캔본,
국세청 양식, 등기부등본)를 PHP로 다루는 도구가 사실상 없다.

```php
Parsel::driver('upstage')->file('계약서.pdf')->markdown();    // 한글 최강, 표 인식
Parsel::driver('clova')->file('영수증.jpg')->markdown();       // 영수증/신분증 특화
Parsel::driver('google-docai')->file('form.pdf')->parse();
Parsel::driver('azure-form')->file('invoice.pdf')->parse();
Parsel::driver('hwp')->file('공문.hwp')->markdown();          // 공공기관 필수
```

**킬러 기능 — 자동 폴백 + 비용 최적화**

```
1. 무료 로컬(lit)로 먼저 시도
2. 텍스트 추출량이 적으면 = 스캔 이미지로 판단
3. → 자동으로 Upstage OCR 폴백
4. 결과 캐싱
→ 비용 70~90% 절감
```

**수익 모델**

| 모델 | 가격 |
|---|---|
| 오픈소스 코어 | 무료 |
| Pro 라이선스 (HWP + 스마트 폴백 + 우선지원) | $99~299/년 |
| 기업 라이선스 (무제한 + SLA) | $999~2999/년 |
| 구축 컨설팅 | 300~1000만원 |

| 기간 | 난이도 | 초기비용 | 차별성 |
|---|---|---|---|
| 1~2개월 | ⭐⭐⭐ | ~10만원 | ⭐⭐⭐⭐⭐ |

---

#### 아이디어 3. `parsel-mcp` — MCP 서버

```
Claude Desktop / Cursor  ↔ MCP ↔  parsel-mcp  ↔  Parsel  ↔  lit / anydoc
```

제공 도구: `parse_document`, `extract_pages`, `ocr_scan`, `extract_tables`,
`document_info`, `screenshot_page`, `search_in_docs`

> 예: *"~/Documents/계약서 폴더에서 위약금 조항 있는 것만 찾아줘"*

**수익 모델**: 무료(기본) / Pro $9.9월(OCR·표추출·배치) / Team $49월 / 화이트라벨

| 기간 | 난이도 | 타이밍 |
|---|---|---|
| 2~3주 | ⭐⭐ | ⭐⭐⭐⭐⭐ (MCP 생태계 초기 = 선점) |

---

### 🟡 티어 2 — SaaS

#### 아이디어 4. 문서 변환 API 서비스

```bash
curl -X POST https://api.example.com/v1/parse \
  -H "Authorization: Bearer sk_live_xxx" \
  -F "file=@report.pdf" -F "format=markdown"
```

직접 구축하면 2주(서버+PHP8.4+Node+lit+LibreOffice+OCR튜닝+큐+모니터링).
API를 쓰면 5분. → **"귀찮음"을 판다.**

| 플랜 | 가격/월 | 페이지 | 기능 |
|---|---|---|---|
| Free | 0 | 100p | 기본, 워터마크 |
| Starter | $19 | 2,000p | 전체 포맷, OCR |
| Pro | $79 | 10,000p | 웹훅, 배치, 우선처리 |
| Business | $299 | 50,000p | 전용큐, SLA 99.9% |
| Enterprise | 문의 | 무제한 | 온프레미스, 전담지원 |

**수익 시뮬레이션**

```
6개월차  : 월 $1,202 (약 165만원) → 순익 약 138만원
1년차    : 월 $5,765 (약 790만원) → 순익 약 710만원
2년차목표: 월 $20,000+ (약 2,700만원)
```

**스택**: Next.js + Laravel/Parsel + Redis·Horizon + S3(24h 자동삭제) + PostgreSQL + Stripe + Docker/ECS

**리스크 대응**

| 리스크 | 대응 |
|---|---|
| 경쟁자(CloudConvert 등) | 한글/HWP 특화로 차별화 |
| 서버비 폭증 | 해시 기반 캐싱, 오토스케일, 크레딧 제한 |
| 악성 파일 | 격리 컨테이너, 크기 제한, 백신 |
| 개인정보 이슈 | 24h 자동 삭제 + 로그 미저장 명시 |

---

#### 아이디어 5. 사내 문서 챗봇 셀프호스팅 솔루션

```
"ChatGPT 좋은데 우리 계약서를 외부에 보낼 순 없다"
→ Parsel(로컬 파싱) + Ollama(로컬 LLM) + pgvector(로컬 DB) = 완전 폐쇄망 AI
```

`docker compose up -d` 한 번으로 app / worker / ollama / postgres / redis 기동.

기능: 폴더 드래그 자동 인덱싱, 출처 하이라이팅 챗 UI, 부서별 권한 관리,
관리자 대시보드, 하이브리드 검색(벡터+키워드)

| 규모 | 가격 | 연 유지보수 |
|---|---|---|
| 소기업 (~20명) | 300만원 | 60만원 |
| 중기업 (~100명) | 1,000만원 | 200만원 |
| 대기업 | 3,000만원~ | 600만원~ |

```
1년차: 3,500만원 + 유지보수 700만원
2년차: 신규 6,000만원 + 유지보수 2,500만원 = 8,500만원
```

---

### 🔴 티어 3 — 버티컬 솔루션 (고단가)

> "모두를 위한 도구"보다 "한 업종을 위한 완결 솔루션"이 훨씬 비싸게 팔린다.

| 업종 | 내용 | 가격 | 난이도 | 추천 |
|---|---|---|---|---|
| 🧾 세무/회계 | 영수증 사진 → 자동 분개 → 더존/이카운트 전송 | 월 5~30만원/사무소 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| ⚖️ 법무 | 계약서 독소조항·자동갱신·관할법원 위험 탐지 | 건당 5~20만원 / 월 50~200만원 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 🏥 의료 | 진료기록 요약 (반드시 로컬) | 1,000~5,000만원 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ (규제↑) |
| 🏗 건설/제조 | 도면·사양서 검색 (screenshots + Vision) | 2,000만원~ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| 🏠 부동산 | 등기부등본 자동 분석 (근저당/가압류/위험도) | 건당 1,000원 / 월 10~50만원 | ⭐⭐ | ⭐⭐⭐⭐⭐ |

```php
// 세무 예시 — 좌표 기반 필드 추출
$doc = Parsel::driver('clova')->file('영수증.jpg')->parse();
$data = $this->extractByCoordinates($doc, [
    '상호'       => ['anchor' => '상호', 'direction' => 'right'],
    '사업자번호' => ['pattern' => '/\d{3}-\d{2}-\d{5}/'],
    '금액'       => ['anchor' => '합계', 'direction' => 'right'],
    '날짜'       => ['pattern' => '/\d{4}[-.]\d{2}[-.]\d{2}/'],
]);
```

---

### 🟣 티어 4 — 콘텐츠·부가 수익

| 상품 | 가격 | 예상 수익 |
|---|---|---|
| 전자책 "PHP로 만드는 로컬 AI 문서 시스템" | 3~5만원 | 월 50~200만원 |
| 온라인 강의 (인프런/유데미) | 8~15만원 | 월 100~500만원 |
| 기업 출강 | 일 100~300만원 | 월 300~900만원 |
| 유튜브/블로그 | 광고+제휴 | 월 30~200만원 |
| 스타터킷/보일러플레이트 판매 | $99~299 | 월 30~100만원 |

---

### 실행 로드맵

```
0~1개월   씨 뿌리기    parsel-laravel + 블로그 + 홍보        수익 0원        목표 스타 100
1~3개월   차별화       parsel-korean + parsel-mcp + Pro판매  월 20~100만원
3~6개월   SaaS 런칭    변환 API MVP + Free플랜 + PH 런칭      월 100~500만원
6~12개월  B2B 확장     버티컬 1개 집중 + 셀프호스팅 패키징     월 500~2,000만원
1년 이후  스케일       팀빌딩 / 해외진출 / 강의·책            월 2,000만원+
```

### 최종 추천 TOP 3

| 순위 | 아이디어 | 이유 | 기간 | 비용 |
|---|---|---|---|---|
| 🥇 | `parsel-korean` + Laravel 통합 | 경쟁자 없음, 모든 사업의 기반 | 1~2개월 | ~0원 |
| 🥈 | `parsel-mcp` 서버 | MCP 생태계 초기 = 선점 효과 최대 | 2~3주 | 0원 |
| 🥉 | 부동산 or 세무 버티컬 SaaS | 1·2순위로 쌓은 실력·인지도로 확장 | 3개월 | 중 |

> 수익화의 핵심은 **기술**이 아니라 **누구의 어떤 고통을 없애주는가**다.
> "세무사님, 영수증 입력 3시간을 3분으로 줄여드립니다" — 이것이 돈이 된다.
> 작게 → 빠르게 → 피드백 → 개선.

---

## 14. 제약사항 정리

| 제약 | 내용 |
|---|---|
| PHP 8.4+ | 비교적 최신 버전 필요 |
| 외부 바이너리 필수 | `lit` / `anydoc` 미설치 시 동작 불가 |
| 공유호스팅 불가 | `proc_open`/`shell_exec` 차단 환경에서 사용 불가 |
| AnyDoc 기능 제한 | Markdown만 지원 |
| v1.0.0 | 신생 버전, API 변동 가능성 |
| Laravel 통합 없음 | 순수 PHP 패키지 (ServiceProvider 미제공) |

---

## 15. 종합 평가

| 항목 | 점수 | 근거 |
|---|---|---|
| 설계 품질 | ⭐⭐⭐⭐⭐ | Manager/Driver 패턴, 능력별 인터페이스 분리 |
| 코드 품질 | ⭐⭐⭐⭐⭐ | 커버리지 100% 강제, PHP 8.4 모던 문법 |
| 문서화 | ⭐⭐⭐⭐☆ | README + UPGRADE + 예제 5종 |
| 확장성 | ⭐⭐⭐⭐⭐ | `extend()`로 어떤 프로바이더든 연결 |
| 실용성 | ⭐⭐⭐⭐☆ | 바이너리 의존이 유일한 허들 |

**총평**: "PHP + AI 문서처리"라는 비어 있던 자리를, 높은 완성도로 선점하려는 프로젝트.
잘 만든 PHP 패키지의 표본으로서 **설계 학습 교재**로도 가치가 크다.

---

*작성: Claude Code 세션 분석 결과 · 2026-09-29*
