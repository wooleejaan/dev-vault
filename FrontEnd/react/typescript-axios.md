# 기본적인 TypeScript와 Axios 사용법

axios를 사용할 때, 추상화를 통해 훅을 만들어 놓으면 사용할 때 편하다.<br>
더불어 개발모드일 때 로그를 찍으면 언제 api를 요청하고, 얼마나 요청하는지 쉽게 확인할 수 있다.

### Instance 와 Common 함수 생성하기

아래와 같이 인스턴스를 생성해주고 url은 env로 관리한다.

```js
// instance.api.ts

export const instance: CustomAxiosInterface = axios.create({
  baseURL: process.env.NEXT_PUBLIC_BASE_URL,
  headers: {
    "Content-Type": "application/json",
    Accept: "*/*",
  },
  timeout: 30000,
});
```

여기에 request/response interceptor를 연결해준다.

### request interceptors

권한이 필요한 요청이라면, 아래와 같이 interceptors에서 Access Token을 주입해주는 식이다.<br>
Refresh Token도 존재한다면, 이 내부에서 확인 로직을 덧붙이면 된다.

```js
// instance.api.ts

(생략...)
instance.interceptors.request.use(
  (config: InternalAxiosRequestConfig): InternalAxiosRequestConfig => {
    /**
     * request 직전 공통으로 진행할 작업
     */
    if (config && config.headers) {
      const token = getCookie('token')
      // 인증할 때 받은 토큰을 쿠키에 저장했다면 가져옵니다.
      if (token) {
        config.headers.Authorization = `Bearer ${token}`
        config.headers['Content-Type'] = 'application/json'
      }
    }
    if (process.env.NODE_ENV === 'development') {
      const { method, url } = config
      logOnDev(`🚀 [API] ${method?.toUpperCase()} ${url} | Request`)
    }
    return config
  },
  (error: AxiosError | Error): Promise<AxiosError> =>
    /**
     * request 에러 시 작업
     */
    Promise.reject(error),
)
```

위에서 사용한 logOnDev 함수는 아래와 같이 개발 모드일 때 콘솔로 로그를 찍어주는 단순한 함수다.

```js
// logOnDev.util.ts

const logOnDev = (message: string) => {
  if (process.env.NODE_ENV === "development") {
    console.log(message);
  }
};

export { logOnDev };
```

### response interceptors

응답의 경우 아래와 같이 Status에 맞게 메시지를 인터셉터 단에서 연결해줄 수도 있다.

```ts
// instance.api.ts

(생략...)
instance.interceptors.response.use(
	(response: AxiosResponse): AxiosResponse => {
		/**
		* http status가 20X이고, http response가 then으로 넘어가기 직전 호출
		*/

		return response
	},
	(error: AxiosError | Error): Promise<AxiosError> => {
		/**
		* http status가 20X가 아니고, http response가 catch로 넘어가기 직전 호출
		*/
		if (process.env.NODE_ENV === 'development') {
		if (axios.isAxiosError(error)) {
			const { message } = error
			const { method, url } = error.config as InternalAxiosRequestConfig
			const { status, statusText } = error.response as AxiosResponse
			logOnDev(
				`🚨 [API] ${method?.toUpperCase()} ${url} | Error ${status} ${statusText} | ${message}`,
		)
		switch (status) {
			case 401: {
			// 로그인 필요 메시지 연결
			break
		}
			case 403: {
			// 권한 필요 메시지 연결
			break
		}
			case 404: {
			// 잘못된 요청 메시지 연결
			break
		}
			case 500: {
			// 서버 문제 발생 메시지 연결
			break
		}
			default: {
			// 알 수 없는 오류 발생 메시지 연결
			break
		}
	}
		} else {
			logOnDev(`🚨 [API] | Error ${error.message}`)
		}
	}
		return Promise.reject(error)
	},
)
```

### Common CRUD 함수 생성하기

위에서 만든 instance에 crud를 연결한 요청 함수를 작성할 수 있다.

이렇게 만든 common 함수의 경우 제네릭으로 외부에서 데이터 타입을 정의할 수 있도록 해서 확장성을 높일 수 있다.

```ts
// common.api.ts

/* eslint-disable @typescript-eslint/no-explicit-any */
import { AxiosRequestConfig, InternalAxiosRequestConfig } from 'axios'
import { CommonResponse } from '@/libs/shared/api/types/type-instance'
import { instance } from './instance'
/* get 요청 */
export const getRequest = async <T>(
	url: string,
	config?: AxiosRequestConfig,
): Promise<T> => {
	const response = await instance.get<CommonResponse<T>>(
		url,
		config as InternalAxiosRequestConfig,
	)
	return response.data
}

/* post 요청 */
export const postRequest = async <T>(
	url: string,
	data: any,
	config?: AxiosRequestConfig,
): Promise<T> => {
	const response = await instance.post<CommonResponse<T>>(
		url,
		data,
		config as InternalAxiosRequestConfig,
	)
	return response.data
}
/* delete 요청 */
...
/* put 요청 */
...
/* patch 요청 */
...
```

이제 common request 함수를 사용해서 아래와 같이 페이지에 필요한 api 함수들을 하나씩 제작하면 된다.<br>
사용하는 쪽에서는 자연스럽게 url을 고려하지 않고 파라미터로 넣을 값들만 고려하면 되므로 사용이 쉽다.<br>
물론 파라미터에 대해 알아야 하므로 jsdoc을 적극적으로 활용하면 좋다.

```ts
// alertsRequest.api.ts

const clearAlerts = async (
  userId: string,
  alertId: string,
  params?: {
    token?: string;
  }
): Promise<UserAlertsListData | string | Error> => {
  try {
    const response = await putRequest<UserAlertsListData>(
      `/users/${userId}/alerts/${alertId}`,
      {},
      {
        headers: {
          Authorization: `Bearer ${params?.token}`,
        },
      }
    );
    return response;
  } catch (error) {
    if (axios.isAxiosError(error)) {
      if (!error.response?.data.message) {
        return error as Error;
      }
      return error.response?.data.message;
    }
    return error as Error;
  }
};
```

### instance, common handler에 필요한 TypeScript 작성하기

CustomAxiosResponse, CommonResponse의 경우 백엔드 api에 맞게 커스텀하면 된다.

```ts
// axios.type.ts

/* instance */
import {
  AxiosInstance,
  AxiosInterceptorManager,
  AxiosResponse,
  InternalAxiosRequestConfig,
} from "axios";
type CustomAxiosResponse<T = any> = {
  response?: T;
  refreshToken?: string;
};
interface CustomAxiosInterface extends AxiosInstance {
  interceptors: {
    request: AxiosInterceptorManager<InternalAxiosRequestConfig>;
    response: AxiosInterceptorManager<AxiosResponse<CustomAxiosResponse>>;
  };
  get<T>(url: string, config?: InternalAxiosRequestConfig): Promise<T>;
  delete<T>(url: string, config?: InternalAxiosRequestConfig): Promise<T>;
  post<T>(
    url: string,
    data?: any,
    config?: InternalAxiosRequestConfig
  ): Promise<T>;
  put<T>(
    url: string,
    data?: any,
    config?: InternalAxiosRequestConfig
  ): Promise<T>;
  patch<T>(
    url: string,
    data?: any,
    config?: InternalAxiosRequestConfig
  ): Promise<T>;
}
/* common */
interface CommonResponse<T> {
  data: T;
  status: number;
  statusText: string;
}
export type { CustomAxiosInterface, CommonResponse };
```
