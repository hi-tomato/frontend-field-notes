# 보안 모듈에 폴백을 만들지 않은 이유

## 상황

비수탁 지갑에서 니모닉은 서버에 없다. 기기에서 지키지 못하면 끝이다.
PIN으로 니모닉을 봉투 암호화하기로 했다 — PIN에서 Argon2id로 키를 파생하고,
그 키로 니모닉을 AES-GCM 암호화하는 구조다.

문제는 Argon2id다. 메모리-하드 KDF를 JS로 돌리면 모바일에서 느리고, 무거운
연산이 JS 단일 스레드를 점유해 UI가 멈춘다. 그래서 Kotlin(Android)과
Swift(iOS)로 **네이티브 모듈을 직접 작성**했다.

그러자 다음 질문이 남았다. **네이티브 모듈을 쓸 수 없는 상황에서는?**

## 갈림길: 폴백을 만들 것인가

같은 앱에 네이티브 모듈이 하나 더 있었다. 니모닉 파생 **속도**를 위한
모듈인데, 이건 JS 폴백이 안전하다 — 결과가 동일하고 느릴 뿐이다.

보안 모듈은 다르다. 네이티브가 안 되니 SHA256 같은 약한 파생으로 내려간다면,
그것은 폴백이 아니라 **보안 강등**이다. 그리고 최악의 성질이 하나 있다:
아무도 모른다. 앱은 조용히 동작하고, 사용자는 자기 지갑이 약한 키로 보호되고
있다는 사실을 알 길이 없다.

가용성과 보안이 충돌할 때의 원칙을 정했다 — **fail-closed.** 안전하게 할 수
없으면, 하지 않는다.

## 구현: 4단계 게이트와 셀프체크

```ts
async function ensureSecureCryptoReady(): Promise<void> {
  if (!featureEnabled)      throw new SecureCryptoUnavailableError("flag");
  if (!isSupportedPlatform) throw new SecureCryptoUnavailableError("platform");
  if (nativeModule == null) throw new SecureCryptoUnavailableError("module");
  await selfCheck();  // 통과 못 하면 여기서도 throw
}
```

주목할 것은 마지막 단계다. **모듈이 로드됐다는 것과, 이 기기에서 올바르게
동작한다는 것은 다르다.** 셀프체크는 고정 벡터로 암호화→복호화 왕복을 돌리고,
**틀린 PIN이 거부되는지까지** 확인한다. 거부 확인이 없으면 "항상 성공하는
가짜 crypto"도 통과해 버린다.

## 미래의 나를 위한 설계: 파라미터를 데이터와 함께 저장

Argon2 파라미터(메모리·반복 횟수)는 기기 성능과 권장 기준에 따라 언젠가
올리게 된다. 그때 기존 사용자의 vault가 열리지 않으면 재앙이다.

그래서 파라미터를 **vault 안에 함께 저장**했다.

```ts
type WalletVault = {
  v: 2;
  salt: string;
  argon2: { memMiB: number; iterations: number; parallelism: number };
  iv: string;
  ct: string;
};
```

값을 올려도 새로 암호화되는 vault에만 적용되고, 기존 vault는 저장된
파라미터로 계속 복호화된다. **재튜닝이 비파괴적**이다.

## 배운 것

폴백은 공짜가 아니다 — "무엇으로 폴백하는가"가 시스템의 실질 하한선이 된다.
성능 폴백은 하한이 "느림"이라 안전하지만, 보안 폴백은 하한이 "약함"이라
그 자체로 취약점이다. 같은 패턴이라도 **실패의 방향**에 따라 정답이 뒤집힌다.
