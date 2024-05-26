---
title: "js 비동기 함수 동시 실행 제한 라이브러리 - async-sema"
description: "async-sema 라이브러리 소개"
date: "2024.05.26"
---

유용한 라이브러리를 찾아서 기록차 글을 남긴다. 이 라이브러리는 비동기 함수 동시 실행 수를 제한한다.

[async-sema github](https://github.com/vercel/async-sema)

사용 방법은 아래와 같다.

```ts
import { Sema } from "async-sema";
import { sleep } from "radash";

const s = new Sema(5);

async function main() {
  for (let i = 0; i < 10; i++) func(i);
}

async function func(index: number) {
  try {
    await s.acquire();
    console.log("작업시작 :", index);
    await sleep(1000);
    console.log("작업완료 :", index);
  } finally {
    await s.release();
  }
}

main();
```

Sema를 생성할 때 동시 실행을 최대 5회로 제한했다. func의 s.acquire() 호출시 현재 동시 실행 수가 5 이하라면 이후 코드의 실행 권한을 얻는다. 5 이상이라면 다른 작업이 완료되어 s.relase()를 호출 할 때까지 기다린다.

결과

```
작업시작 : 0
작업시작 : 1
작업시작 : 2
작업시작 : 3
작업시작 : 4
작업완료 : 0
작업시작 : 5
작업완료 : 1
작업시작 : 6
작업완료 : 2
작업시작 : 7
작업완료 : 3
작업시작 : 8
작업완료 : 4
작업시작 : 9
작업완료 : 5
작업완료 : 6
작업완료 : 7
작업완료 : 8
작업완료 : 9
```

작업이 하나 완료될 때 마다 대기하던 i+5번 작업이 즉시 실행된다.

고차함수를 만들어서 쓰면 좀 더 사용이 쉽다.

```ts
// createSema.ts
import { Sema } from "async-sema";
export function createSema<T>(n: number) {
  const s = new Sema(n);
  const withLock = async (func: () => T) => {
    try {
      await s.acquire();
      return await func();
    } finally {
      s.release();
    }
  };

  return {
    s,
    withLock,
  };
}

// main.ts
import { sleep } from "radash";
import { createSema } from "./sema";

const { withLock } = createSema(5);

async function main() {
  for (let i = 0; i < 10; i++) {
    withLock(() => func(i));
  }
}

async function func(index: number) {
  console.log("작업시작 :", index);
  await sleep(1000);
  console.log("작업완료 :", index);
}

main();
```
