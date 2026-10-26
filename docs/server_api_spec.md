# DigitalTok 서버 API 명세 (코드에서 추출)

- 출처: [DigitalTok/digitaltok-server](https://github.com/DigitalTok/digitaltok-server)
- 스택: Spring Boot, JPA, Spring Security + JWT, AWS S3, EC2(Docker), Swagger
- Base: `https://www.diring.site`

## 공통 응답 형식

```json
{ "isSuccess": true, "code": "...", "message": "...", "result": { } }
```
`code`, `message`, `result`는 값이 null이면 응답에서 빠집니다.

## 인증 규칙 (SecurityConfig)

| 경로 | 권한 |
|---|---|
| `/api/v1/auth/**`, `/health`, Swagger | 누구나 |
| `/api/v1/templates/subway/generate` | **ADMIN 전용** |
| 그 외 전부 | `Authorization: Bearer {accessToken}` 필요 |

---

## 1. Auth: `/api/v1/auth`

| Method | Path | Request Body | Result |
|---|---|---|---|
| POST | `/signup` | `email`*, `password`*(6~20자), `phoneNumber`(`010-xxxx-xxxx`, 선택) | `userId, email, nickname, accessToken, refreshToken` |
| POST | `/login` | `email`*, `password`* | `grantType, accessToken, refreshToken, accessTokenExpiresIn` |
| DELETE | `/logout` | `refreshToken`* | String |
| POST | `/duplicate-check` | `email`* | String |
| POST | `/password/reset` | `email`* | String |

## 2. User: `/api/v1/users`

| Method | Path | Request Body | Result |
|---|---|---|---|
| GET | `/me` | - | `userId, nickname, email` |
| DELETE | `/me` | - (회원 탈퇴) | String |
| PATCH | `/password` | `oldPassword`*, `newPassword`* | String |
| PATCH | `/email` | `password`*(본인 확인), `newEmail`* | String |

## 3. Device: `/api/v1/devices`

| Method | Path | Request | Result |
|---|---|---|---|
| POST | `` | body `nfcUid`* | `deviceId, registeredAt, unregisteredAt, status(ACTIVE/INACTIVE)` |
| GET | `/{nfcUid}` | - | 위와 같음 |
| DELETE | `/{nfcUid}` | - | 위와 같음 |

## 4. Image: `/api/v1/images`

| Method | Path | Request | Result |
|---|---|---|---|
| POST | `` (multipart) | part `file`, query `imageName` | `image{imageId, originalUrl, previewUrl, einkDataUrl, imageName, createdAt, deletedAt}`, `imageMapping{userImageId, userId, imageId, isFavorite, savedAt, lastUsedAt}` |
| GET | `/recent` | - | `count, items[{imageId, previewUrl, imageName, isFavorite, lastUsedAt}]` |
| GET | `/{imageId}/preview` | - | `imageId, previewUrl, updatedAt` |
| GET | `/{imageId}/binary` | - | `imageId, einkDataUrl, lastUsedAt, meta{...}` |
| PATCH | `/{imageId}/favorite` | body `isFavorite`* | `userId, imageId, isFavorite, favoriteCount, updatedAt` |

`meta`는 항상 고정값입니다: `width 200, height 200, bpp 2, palette "Black=0, White=1, Yellow=2, Red=3", packing "MSB-first", scan "row-major", payloadBytes 10000, hasHeader false`

> [!NOTE]
> 업로드는 **비동기**로 처리됩니다. 응답은 바로 오지만(`previewUrl`/`einkDataUrl` = null), 실제 S3 업로드와 변환은 `@Async` 리스너가 나중에 수행하고, 이미지 상태가 `PROCESSING → COMPLETED/FAILED`로 바뀝니다.

## 5. Template: `/api/v1/templates`

| Method | Path | Request | Result |
|---|---|---|---|
| GET | `/priority` | - | `count, items[{templateId, priorityType, templateImageUrl}]` |
| GET | `/priority/{templateId}` | - | `templateId, priority_type`⚠️snake_case, `templateImageUrl, templateDataUrl` |
| GET | `/subway` | - | `count, items[{templateId, stationName, lineName, templateImageUrl}]` |
| GET | `/subway/{templateId}` | - | `templateId, stationName, stationNameEng, lineName, templateImageUrl, templateDataUrl` |
| GET | `/subway/search` | query `keyword`(선택) | 목록과 같음 |
| POST | `/subway/generate` 🔒ADMIN | `stationName`(10자 이하, '역' 붙이지 않음), `stationNameEng`(영문 20자 이하), `lineName`(10자 이하, '호선' 붙이지 않음) | String |

지하철 템플릿은 **관리자가 미리 만들어 두는 데이터**입니다. Pretendard-Bold 폰트로 역 이름과 노선 이미지를 그린 뒤 4색 변환(디더링 없음)해서 S3에 올립니다.

---

## 6. 실제 E-ink 변환 로직 (`image/service/processing`)

```
원본 → [Preprocessor] 센터 정사각형 크롭 → 200×200 Bicubic 리사이즈
     → [Quantizer] 파란색 계열 보정 → 최근접 팔레트(B/W/Y/R)
                   (사용자 업로드: Floyd–Steinberg 디더링 / 지하철 템플릿: 디더링 OFF)
     → [PreviewRenderer] 4색 결과를 PNG preview로 렌더링
     → [BinaryEncoder] MSB-first 2bpp 패킹 → 10,000 bytes 검증 → S3
```

> [!WARNING]
> **실제 코드는 명세 문서와 스캔 방향이 다릅니다.**
> 명세에는 `(0,0)→(199,0)`처럼 왼쪽에서 오른쪽으로 읽는다고 되어 있지만, `EinkBinaryEncoder`는 각 행을 **오른쪽에서 왼쪽으로(x=199→0)** 읽습니다.
> 코드 주석: *"좌우 반전 보정: x를 오른쪽→왼쪽으로 스캔, 패킹 밀림 현상 해결"*
> 실제 기기로 테스트하면서 맞춘 값으로 보이므로, Firebase 버전은 **코드 동작을 기준으로** 구현해야 합니다.

---

## 7. 앱 ↔ 서버 불일치 (Firebase 이전 시 함께 정리)

| 항목 | 서버 | 앱 |
|---|---|---|
| 기기 해제 경로 | `DELETE /devices/{nfcUid}` | `DELETE /devices/nfc/{nfcUid}` ❌ |
| 비밀번호 변경 | `PATCH /users/password` 있음 | 앱에 API 정의 없음 |
| 업로드 응답 | `category`, `subwayTemplateId` 주석 처리됨 | 앱 DTO에서 필수(non-null)로 선언됨 |
| Mock meta | 200×200 / 2bpp / 10000 | 250×122 / 1bpp / 4000 ❌ |
