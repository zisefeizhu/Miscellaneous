```go
package middleware

import (
	"containerplatform/pkg/errno"
	"containerplatform/pkg/getcookie"
	"crypto/tls"
	"fmt"
	"io/ioutil"
	"net"
	"net/http"
	"strings"

	"errors"
	"time"
)

// request.Header.Set 与request.Header.Add 区别：https://www.cnblogs.com/Dong-Ge/articles/11269898.html

type HTTPClient interface {
	Do(req *http.Request) (*http.Response, error)
}

var (
	Client HTTPClient
)

func init() {
	Client = &http.Client{
		Timeout: 3 * time.Second,
		//忽略https证书
		Transport: &http.Transport{
			TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
			Proxy:           http.ProxyFromEnvironment,
			DialContext: (&net.Dialer{
				Timeout:   1 * time.Second,
				KeepAlive: 30 * time.Second, // HTTp keeplive
				DualStack: true,
			}).DialContext,
			MaxIdleConns:        100,
			MaxIdleConnsPerHost: 8,
			IdleConnTimeout:     120 * time.Second,
		},
	}
}

// CheckRespStatus 状态检查
func CheckRespStatus(resp *http.Response) ([]byte, error) {
	bodyBytes, _ := ioutil.ReadAll(resp.Body)

	if resp.StatusCode >= 200 && resp.StatusCode < 400 {
		return bodyBytes, nil
	}
	return bodyBytes, errno.New(errno.InternalServerError, errors.New(string(bodyBytes)))
}

// GetRequest 建立Get请求
func GetRequest(url string, headerSet map[string]string) (respStatusCode int, respBytes []byte, err error) {
	// token requestMethod url 参数是有顺序的
	// 提交请求
	request, err := http.NewRequest("GET", url, nil)
	if err != nil {
		err = errno.New(errno.InternalServerError, err)
		return
	}

	// 添加token
	// request.Header.Set("Authorization", "Bearer " + token)

	// 添加cookie
	cookieName, cookieValue := getcookie.GetCookie()
	cookie := &http.Cookie{Name: cookieName, Value: cookieValue, HttpOnly: true}
	request.AddCookie(cookie)

	// header 添加字段
	if headerSet != nil {
		for k, v := range headerSet {
			// request.Header.Set 与request.Header.Add 区别：https://www.cnblogs.com/Dong-Ge/articles/11269898.html
			request.Header.Set(k, v)
		}
	}

	// 处理返回结果
	resp, err := Client.Do(request)
	if err != nil {
		err = errno.New(errno.InternalServerError, err)
		return 0, nil, err
	}
	defer resp.Body.Close()
	// 返回的状态码
	respBytes, err = CheckRespStatus(resp)
	respStatusCode = resp.StatusCode
	return
}

// PostRequest 建立Post请求
func PostRequest(url, body string, headerSet map[string]string) (respStatusCode int, respBytes []byte, err error) {
	request, err := http.NewRequest("POST", url, strings.NewReader(body))
	if err != nil {
		err = errno.New(errno.InternalServerError, err)
		return
	}
	cookieName, cookieValue := getcookie.GetCookie() // 拿到cookie
	cookie := &http.Cookie{Name: cookieName, Value: cookieValue, HttpOnly: true}
	request.AddCookie(cookie)
	// header 添加字段
	if headerSet != nil {
		for k, v := range headerSet {
			// request.Header.Set 与request.Header.Add 区别：https://www.cnblogs.com/Dong-Ge/articles/11269898.html
			request.Header.Set(k, v)
		}
	}
	resp, err := Client.Do(request)
	if err != nil {
		err = errno.New(errno.InternalServerError, err)
		return 0, nil, err
	}
	defer resp.Body.Close()
	// 返回的状态码
	respBytes, err = CheckRespStatus(resp)
	respStatusCode = resp.StatusCode
	return
}


// PutRequest 建立Put请求
func PutRequest(url, body string, headerSet map[string]string)  (respStatusCode int, respBytes []byte, err error) {
	request, err := http.NewRequest("PUT", url, strings.NewReader(body)); if err != nil {
		err = errno.New(errno.InternalServerError, err)
		return
	}

	cookieName, cookieValue := getcookie.GetCookie() // 拿到cookie
	cookie := &http.Cookie{Name: cookieName, Value: cookieValue, HttpOnly: true}
	request.AddCookie(cookie)

	// header 添加字段
	if headerSet != nil {
		for k, v := range headerSet {
			request.Header.Set(k, v)
		}
	}

	resp, err := Client.Do(request)
	if err != nil {
		fmt.Println(err.Error())
		err = errno.New(errno.InternalServerError, err)
		return 0, nil, err
	}
	defer resp.Body.Close()
	//返回的状态码
	respBytes, err = CheckRespStatus(resp)
	respStatusCode = resp.StatusCode
	return
}

// PatchRequest 建立Patch请求
func PatchRequest(url, body string, headerSet map[string]string) (respStatusCode int, respBytes []byte, err error) {
	request, err := http.NewRequest("PATCH", url, strings.NewReader(body))
	if err != nil {
		err = errno.New(errno.InternalServerError, err)
		return
	}
	cookieName, cookieValue := getcookie.GetCookie() // 拿到cookie
	cookie := &http.Cookie{Name: cookieName, Value: cookieValue, HttpOnly: true}
	request.AddCookie(cookie)
	// header 添加字段
	if headerSet != nil {
		for k, v := range headerSet {
			request.Header.Set(k, v)
		}
	}
	resp, err := Client.Do(request)
	if err != nil {
		err = errno.New(errno.InternalServerError, err)
		return 0, nil, err
	}
	defer resp.Body.Close()

	//返回的状态码
	respBytes, err = CheckRespStatus(resp)

	respStatusCode = resp.StatusCode
	return
}

// DeleteRequest 建立Delete请求
func DeleteRequest(url, body string, headerSet map[string]string) (respStatusCode int, respBytes []byte, err error) {
	request, err := http.NewRequest("DELETE", url, strings.NewReader(body))
	if err != nil {
		err = errno.New(errno.InternalServerError, err)
		return
	}

	cookiename, cookievalue := getcookie.GetCookie() // 拿到cookie
	cookie := &http.Cookie{Name: cookiename, Value: cookievalue, HttpOnly: true}
	request.AddCookie(cookie)
	// header 添加字段
	if headerSet != nil {
		for k, v := range headerSet {
			// request.Header.Set 与request.Header.Add 区别：https://www.cnblogs.com/Dong-Ge/articles/11269898.html
			request.Header.Set(k, v)
		}
	}
	resp, err := Client.Do(request)
	if err != nil {
		err = errno.New(errno.InternalServerError, err)
		return 0, nil, err
	}
	defer resp.Body.Close()
	//返回的状态码
	respBytes, err = CheckRespStatus(resp)
	respStatusCode = resp.StatusCode
	return
}
```
- 注：上述五类http请求实质还可以写到一个函数中，这里为了调用清晰特地按类分开。