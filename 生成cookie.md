在调用第三方接口时，需要用到凭证，大多数接口使用token即可，那自然有小多数接口使用cookie等认证。此系列将围绕我的研发之旅进行。

获取cookie相对简单独立，直接上代码
```go
package main

import (
	"bytes"
	"fmt"
	"mime/multipart"
	"net/http"
	"time"
)

var (
	CookieName  string // cookie k
	CookieValue string // cookie v
	Time        int    // cookie 限期
)

// 初始化cookie所需信息
func cookieInit() {
	postData := make(map[string]string)
	postData["username"] = "axxxn"
	postData["password"] = "axxxxxt"
	url := "http://txxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxkens"
	postWithFormData("POST", url, &postData)

}

// PostWithFormData 获取cookie和过期时间
func postWithFormData(method, url string, postData *map[string]string) {
	body := new(bytes.Buffer)
	w := multipart.NewWriter(body)
	for k, v := range *postData {
		w.WriteField(k, v)
	}
	w.Close()
	req, _ := http.NewRequest(method, url, body)
	req.Header.Set("Content-Type", w.FormDataContentType())
	resp, err := http.DefaultClient.Do(req)
	if err != nil || resp.StatusCode > 300 {
		fmt.Println("err = ", err)
		fmt.Printf("%+v\n", resp.Cookies())
		return
	}
	defer resp.Body.Close()
	for _, cookie := range resp.Cookies() {
		if cookie.Name == "tke" {
			CookieName = cookie.Name
			CookieValue = cookie.Value
			nowtime := time.Now().Unix()
			Time = cookie.MaxAge + int(nowtime)
		}
	}
}

// GetCookie 判断cookie过期时间范围cookie
func GetCookie() (string, string) {
	nowtime := time.Now().Unix()
	if len(CookieValue) == 0 {
		cookieInit()
	} else {
		if int(nowtime) >= Time {
			cookieInit()
		}
	}
	return CookieName, CookieValue
}

func main()  {
	n,v := GetCookie()
	fmt.Println("n = ", n)
	fmt.Println("v = ", v)
}
// n = hubxxxxxxxen
// v = eyJhbGxxxxx9pZCI6MCwib3Blbl91c2VyX2lkIjowLCJxxxxiLCJhdmF0YXJfdXJsIjoiIiwic3BhY2VfaWQiOiIiLCJhcHBfa2V5xxxfaWQiOiIyZWNiMDkyNi03ZWYzLxxxxN1ZWRfYXQiOjE2MzA0ODxxxxiOIOfC0TJxxAcc=
```