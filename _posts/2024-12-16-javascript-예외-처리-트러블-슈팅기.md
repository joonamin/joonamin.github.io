---
layout: post
title: Javascript 예외 처리 트러블 슈팅기
date: 2024-12-16 02:12 +0900
pin: false
math: true
mermaid: true
description: 클래스 A를 상속받은 자식 클래스 B에서, 클래스 A에 대한 instanceof 연산이 false가 되는 문제가 발생했습니다.
categories: [Dev, Troubleshooting]
tags: [prototype, error middleware]

---

## 서론

authorization header로 전달되는 jwt를 verify하여 이를 request body에 포함시키는 로직을 작성하는 도중 아래와 같은 문제가 발생하였습니다.


* Authorization 필드가 누락된 경우, Exception을 던져 `ErrorMiddleware` 에서 처리할 수 있도록 위임합니다.
* `ErrorMiddleware` 에서는 예측 가능한 custom exception의 인스턴스인지를 판단하고 `예상 가능했던 예외`인지 판단하고 response에 이를 포함시킵니다.
* 하지만, CustomException을 throw 하였음에도, `500` internal server error가 발생하였습니다.



## 문제의 Exception 클래스들

예상 가능한 예외 상황을 `APIError` 클래스를 base로 하는 Exception으로 정의하였습니다.


```javascript
export class APIError extends Error {
    statusCode: number
    errorCode: number
    message: string
    cause: Error | string
	... 생략 ...
}
```

{: file="src/types/errors/error.ts"}


```javascript
export class NotFoundTokenException extends APIError {
    constructor(cause: Error | string = null) {
        super(403, 4030, 'No Token provided', cause)
        Object.setPrototypeOf(this, NotFoundTokenException)
        Error.captureStackTrace(this, NotFoundTokenException)
    }
}
```

{: file="src/types/errors/auth.ts"}

`throw new NotFoundTokenException` 으로 던져진 예외가, error를 처리하는 middleware로 전달되었지만 알 수 없는 이유로 `error instanceof APIError` 가 false로 평가되었습니다.

```javascript
export const verifyToken = (req: Request, res: Response, next: NextFunction) => {
    const token = req.headers['authorization']?.split(' ')[1]
    if (!token) {
        throw new NotFoundTokenException(new Error('check the "authorization" header field'))
    }
```

{: file="src/middlewares/auth.ts"}



## 첫번째 시도

> The order of middleware loading is important: middleware functions that are loaded first are also executed first.  <br/>
> [출처](https://expressjs.com/en/guide/writing-middleware.html)
> {: .prompt-info}

처음에는 throw된 exception이 error middleware에 전달되지 않아 발생한 문제점으로 인지하였습니다.  이를 먼저 점검해보았고, 문제의 소지가 없어보였습니다.

express middleware는 미들웨어를 처리하는 도중, 에러가 발생할 경우 가장 가까운 error handling middleware를 찾아 에러를 전달합니다. 

<img src="https://raw.githubusercontent.com/joonamin/UpicImageRepo/master/uPic/image-20241220023829317.png" alt="image-20241220023829317" style="zoom:67%;" />

이러한 플로우에 따라서 앱이 동작하는지 디버거를 통해서 확인하여 error middleware가 정상적으로 동작한다는 것을 보장할 수 있었습니다.



## 두번째 시도

문제의 원인은 `error` 를 `APIError`의 instance로 인식하지 못하는 것이었습니다.

```javascript
const errorMiddleware = (error: APIError, req: Request, res: Response, next: NextFunction) => {
    try {
        if (!(error instanceof APIError)) {
            error = new InternalServerError(error)
        }
        res.meta = res.meta ? { ...res.meta, error } : { error }
        res.status(error.statusCode).json({
            message: error.message,
            code: error.errorCode,
        })
    } catch (err) {
        logger.error('fail in error middleware', { original: error, new: err })
        res.status(500).json({ message: 'internal server error', code: 500 })
    }
}
```

3-4번 라인에서 `error`가 APIError의 인스턴스임에도 `false`로 평가하여 `InternalServerError` 로 처리된 것이었습니다.

자바스크립트에서는 `상속`을 프로토타입으로 구현합니다. `APIError`를 상속받은 `NotFoundTokenException` 의 프로토타입을 확인해야했습니다. [참고](https://ko.javascript.info/class)

너무나도 당연하게 `Object.setPrototypeOf(this, NotFoundTokenException.prototype)` 으로 설정되어 있어야하는 값이 `NotFoundTokenException` 으로 지정되어있었습니다. 이에 따라, prototype chain을 순회하면서 `error`가 `APIError`의 인스턴스인지 확인하는 과정에서 `false`로 평가되었던 것이었습니다.



## 이런 문제가 다시는 발생하지 않도록..

해당 문제는 결국 개발 과정에서 발생한 실수 때문이었습니다. 

조금 더 신경을 써서 작업을 한다면 실수를 줄일 수 있겠지만, 원천적으로 예방할 수는 없다고 생각했습니다.

`instanceof`를 자바스크립트에서는 어떤 방식으로 내부적으로 처리하는지 찾아보던 도중, 문제 지점을 줄일 수 있었고 빠르게 수정할 수 있었습니다. 

이처럼, 문제가 발생했을 때 문제가 발생한 지점을 빠르게 좁혀나갈 수 있도록 많은 경험을 기록하며 축적하는 것이 최선이라 생각합니다.
