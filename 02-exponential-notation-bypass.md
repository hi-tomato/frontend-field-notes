# "1e-7"이 금액 검증을 뚫던 문제

## 상황

송금 화면에는 "보유량을 초과해 전송할 수 없다"는 규칙이 있다. 검증은 금액
문자열을 소수점 기준으로 쪼개 정수부와 소수부의 자릿수를 세는 방식이었다.

```ts
const [int, frac = ""] = amount.split(".");
if (frac.length > decimals) reject();
```

## 문제

JS의 `Number.prototype.toString()`은 1e-7 미만, 1e21 이상에서 **지수 표기**를
내놓는다.

```ts
(0.0000001).toString();  // "1e-7"
"1e-7".split(".");       // ["1e-7"] — 소수부가 없다?
```

소수점이 없으니 소수부는 빈 문자열이 되고, 자릿수 검증을 그대로 통과한다.
아주 작은 금액과 아주 큰 금액에서 검증이 무력화된다.

## toFixed는 왜 답이 아닌가

`toFixed()`로 십진 문자열을 만들면 될 것 같지만, 여기서 **반올림**이 끼어든다.

절삭해야 할 값이 올림되면 보유량보다 큰 금액이 검증을 통과할 수 있다.
금액 처리의 안전 방향은 "모자라게"이지 "넘치게"가 아니다 — 반올림은 안전
불변식을 반대 방향으로 깬다.

BigNumber 라이브러리 도입도 검토했지만, 계산 자체는 이미 `bigint` 기반으로
정수 처리되고 있었다. 필요한 것은 **표시·검증 직전의 문자열 정규화** 하나였고,
그것 때문에 의존성을 추가하는 것은 과했다.

## 선택: 가수부를 지수만큼 시프트한다

지수 표기를 정규식으로 분해하고, 소수점 위치를 지수만큼 옮겨 십진 문자열로
전개한다. 숫자로 되돌리지 않으므로 반올림이 개입할 곳이 없다.

```ts
function expandExponential(amount: string): string {
  const m = amount.match(/^([+-]?)(\d+)(?:\.(\d*))?[eE]([+-]?\d+)$/);
  if (!m) return amount;                       // 지수 표기가 아니면 그대로
  const [, sign, int, frac = "", exp] = m;
  const digits = int + frac;
  const point = int.length + Number(exp);      // 새 소수점 위치
  if (point >= digits.length)
    return sign + digits + "0".repeat(point - digits.length);
  if (point > 0)
    return `${sign}${digits.slice(0, point)}.${digits.slice(point)}`;
  return `${sign}0.${"0".repeat(-point)}${digits}`;
}
```

경계값(1e-7, 1.2e+21, 음수)을 단위 테스트로 고정했다.

같은 작업의 연장으로, 토큰마다 다른 `decimals`(USDT는 6, ETH는 18)를 금액
유틸 전반에 도입해 "18자리 고정" 가정으로 어긋나던 표시·입력·최대치 경로를
함께 정리했다.

## 배운 것

경계는 **표현이 바뀌는 지점**에서 뚫린다. 값은 같아도 `0.0000001`과 `"1e-7"`은
문자열 검증 입장에서 다른 입력이다. 금액처럼 안전 방향이 있는 도메인에서는
"정확한 변환"보다 **"어느 쪽으로 틀릴 것인가"**를 먼저 정해야 한다.
