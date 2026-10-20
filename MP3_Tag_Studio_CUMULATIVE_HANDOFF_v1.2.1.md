# MP3 Tag Studio — Cumulative Handoff (v1.2.1)

- 날짜: 2026-10-20
- 형식: Single HTML, ID3v2.3 Writer / MP3 audio payload preserve
- 목적: 영문으로 잘못 붙은 국내 가요 MP3의 곡명/가수/앨범/표지/가사 교정

## 구현 경과

### v1.0.0
- 여러 곡 MP3 로드 및 기존 ID3/ID3v1 파싱
- 음원 정보 수동 편집, 자동 정보 검색, 가사/동기화 가사, 표지
- ID3v2.3 저장, ZIP 저장

### v1.1.0
- Apple Music JSONP 우선 검색, MusicBrainz 검색 실패 독립 처리
- 검색 연결 진단
- 쓰기 가능한 `FileSystemFileHandle`을 이용한 원본 덮어쓰기 및 기존 파일 내용 비교

### v1.2.0
문제 제보: 덮어쓰기 클릭해도 반응 없음, 플레이어에 앨범아트 없음.

원인 분석:
- `updateOverwriteUI()`에서 `!canUseFilePicker()`면 버튼이 disabled 상태가 됨. 파일/브라우저 제한 시 사용자 반응이 없음.
- 커스텀 confirm()이 원본 picker 호출보다 앞에 있어 transient user activation을 잃을 가능성 존재.
- 검색 결과 이미지가 화면에 렌더링되더라도 CORS/네트워크 오류로 바이너리 Blob을 가져오지 못하면 MP3에 삽입되지 않음.
- HTML `<audio controls>`는 커버를 표시하는 UI가 아니며 현재 음원은 저장 이전 원본임.

해결:
- 원본 덮어쓰기 버튼을 파일 선택 이후 항상 클릭 가능하게 변경. 지원 안 되는 곳에선 '같은 이름으로 저장' 다운로드 대체 (원본 직접 수정이라고 주장하지 않음).
- 파일 picker 호출 이전의 blocking `confirm()` 제거, 쓰기 권한/원본 일치 검사/쓰기 전 confirm으로 순서 변경.
- 쓰기 관련 SecurityError/NotAllowedError 발생 시 별도 오류를 명시하고 같은 이름 다운로드로 폴백.
- 앨범아트 JPEG 변환 및 1000px 크기 제한, ID3v2.3 `APIC` image/jpeg 타입 3(front-cover)으로 통일.
- `verifyBuiltMp3()`가 결과 Blob의 실제 ID3 프레임을 다시 파싱해 곡명/가수/앨범/가사, APIC JPEG 원시 데이터 검증.
- 직접 저장한 경우 `getFile()`로 디스크 파일을 다시 열어 재검증.
- `setCoverStatus()`, `setSaveFeedback()`으로 표지 준비 여부/저장 결과/실패 이유를 지속 표시.
- 자동 이미지 수신 실패 시 원본 이미지 페이지를 여는 버튼 제공. 수동 이미지 업로드 안내.
- 브라우저 내장 오디오바와 태그 저장의 차이 안내.

### v1.2.1 (2026-10-20)
문제 제보: `원본 저장 실패: An operation that depends on state cached in an interface object was made but the state had changed since it was read from disk.`

**원인**: `buildMp3(song)`의 `new Blob([header, song.file.slice(...)])`는 전체 원본 오디오를 즉시 메모리로 복사한다고 보장하지 않음. 동일 원본 핸들에 `createWritable()` 후 `write(blob)`가 진행될 때, 뒤늦게 source File snapshot을 읽으면 원본이 변경/교체되어 Chrome이 과거 `File` 객체 읽기를 거부함. 파일시스템 API의 `getFile()` 문서상 디스크 파일 변경 시 반환한 `File`이 unreadable할 수 있음.

**수정**:
- `detachFileBytes(file)`는 원본을 8MiB 단위로 **쓰기 시작 전** 읽어 Uint8Array 구성 후 새 Blob 생성 (768MB 초과는 안전하게 차단).
- `buildMp3(song, sourceFile=song.file)`가 독립 Blob을 소스로 받아 MP3 audio payload를 구성할 수 있도록 확장. 기존 `saveSelected()` 및 ZIP 경로는 기본값으로 호환.
- `overwriteSelected()`는 ID3 기록 예정 Blob을 완전히 준비하여 검증하고 최신 파일 크기/수정일을 재검사한 뒤에만 `createWritable()` 실행.
- 쓰기 직전 플레이어 정지/URL 해제, 쓰기 성공 뒤 최신 `getFile()` 기준으로 태그/커버/데이터 갱신, 연속 덮어쓰기 대응.
- `refreshSelectedFromDisk()` 원본 재연결 버튼 추가 (편집값 유지). 쓰기 커밋 후 검증 실패는 별도 경고하여 중복쓰기 방지.
- 보안 거절/파일 변경/스트림 쓰기 오류 별도 상태 표시.

**테스트**: `node --check` PASS, Node 모의 writable handle 첫 저장 및 연속 두 번 저장 PASS, 디스크 외부 변경 차단 PASS, 원본 재연결 PASS, 외부 Mutagen에서 TIT2(한글/이모지), TPE1, JPEG APIC 읽기 PASS. 자동 브라우저는 실행 환경 접근 정책으로 테스트 불가. 실제 사용자 OS/브라우저 파일시스템에서의 복제 확인은 미검증.

## 주요 함수
- `loadFile` / `parseTag`: ID3 태그 읽기. 기존 APIC 이미지 파싱.
- `buildMp3`: ID3v2.3 프레임 생성, 이미지 JPEG 정규화, 오디오 데이터 원본 유지.
- `normalizeImage`: JPEG 유효성 검사, PNG/WebP/AVIF 등을 캔버스 JPEG로 변환.
- `verifyBuiltMp3`: 저장 Blob에서 메타데이터와 APIC 프레임 다시 검사.
- `overwriteSelected`: 지원 환경에서 원본 선택 → 권한 → 동일성 확인 → 쓰기 → 재검증.
- `saveSelected`: `_tagged.mp3` 다운로드, 내부 재검증.
- `setCoverFromURL`: fetch 결과를 이미지 Blob으로 저장, 실패 시 안내.

## 테스트
- `node --check` JS 문법 검증: PASS
- Node JS 단위 테스트: ID3 JPEG APIC, 원본 핸들 덮어쓰기, 대체 다운로드, 다른 원본 차단: PASS
- Python Mutagen 실제 output MP3: `TIT2` 한글/이모지와 `APIC`(JPEG) 및 MP3 길이 읽기: PASS
- 외부 검색 서비스 실시간 응답 / 브라우저 로컬 UI / 사용자 음악 앱별 표시: 미검증

## 안전/제약
- 사용자 명시적 권한 없이 브라우저가 로컬 원본 파일에 직접 덮어쓰는 것은 불가능함. Firefox/Safari/모바일에서는 일반적으로 대체 다운로드 동작.
- 일부 MP3 플레이어는 내장 커버를 읽지 않거나 이전 메타데이터를 캐시함. 변경 파일을 다시 추가하거나 라이브러리 캐시를 갱신할 필요가 있음.
- 다운로드 브라우저 기능은 실제 다운로드 완료 자체를 JS가 강제로 확인할 수 없음. 내부 Blob의 ID3 검증만 보장.
- 음원 파일이 이미 변경되었거나 다른 파일을 덮어쓰려 하면 거부함.

## 후속 개선 후보
- 가사 제공처 다변화, 한영 곡명 유사도 개선
- MP3 태그 내장 이미지의 픽셀 디코딩 및 크기·품질 진단 패널
- 원본 디렉터리 읽기/쓰기 권한 및 여러 파일 배치 덮어쓰기 (사용자 명시 승인 필수)