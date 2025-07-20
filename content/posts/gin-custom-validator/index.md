+++
title = 'Gin Custom Validator'
date = 2025-05-01T15:19:59+08:00
draft = false
description = '分享在 Gin 自訂驗證器以及自訂錯誤訊息的寫法'
toc = true
tags = ['GO']
categories = ['GO']
+++

發現在 gin 框架寫自訂驗證器比我想像中困難，看來我被 Laravel 養太好了🤯

這篇分享自訂驗證器以及自訂錯誤訊息的寫法，紀錄一下以免以後忘記~

## 先看官方文件的範例吧
[Gin 官方文件](https://gin-gonic.com/en/docs/examples/custom-validators/)的範例是，有一支 API 需要輸入 check in 和 check out 的日期，並自訂一個驗證器 `bookabledate`，如果輸入時間在今天之後，則驗證通過
```go
package main

import (
  "net/http"
  "time"

  "github.com/gin-gonic/gin"
  "github.com/gin-gonic/gin/binding"
  "github.com/go-playground/validator/v10"
)

// Booking contains binded and validated data.
type Booking struct {
  CheckIn  time.Time `form:"check_in" binding:"required,bookabledate" time_format:"2006-01-02"`
  CheckOut time.Time `form:"check_out" binding:"required,gtfield=CheckIn,bookabledate" time_format:"2006-01-02"`
}

var bookableDate validator.Func = func(fl validator.FieldLevel) bool {
  date, ok := fl.Field().Interface().(time.Time)
  if ok {
    today := time.Now()
    if today.After(date) {
      return false
    }
  }
  return true
}

func main() {
  route := gin.Default()

  if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
    v.RegisterValidation("bookabledate", bookableDate)
  }

  route.GET("/bookable", getBookable)
  route.Run(":8085")
}

func getBookable(c *gin.Context) {
  var b Booking
  if err := c.ShouldBindWith(&b, binding.Query); err == nil {
    c.JSON(http.StatusOK, gin.H{"message": "Booking dates are valid!"})
  } else {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
  }
}
```
測試
```
$ curl "localhost:8085/bookable?check_in=2018-04-16&check_out=2018-04-15"
{"error":"Key: 'Booking.CheckIn' Error:Field validation for 'CheckIn' failed on the 'bookabledate' tag\nKey: 'Booking.CheckOut' Error:Field validation for 'CheckOut' failed on the 'gtfield' tag"}
```

## 翻譯錯誤訊息
來修改一下，用 field 作為 key，錯誤訊息作為 value 回傳，並且錯誤訊息翻譯為中文
```go
package main

import (
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/locales/en"
	"github.com/go-playground/locales/zh_Hant_TW"
	ut "github.com/go-playground/universal-translator"
	"github.com/go-playground/validator/v10"
	zh_tw_translations "github.com/go-playground/validator/v10/translations/zh_tw"
)

// Booking contains binded and validated data.
type Booking struct {
	CheckIn  time.Time `form:"check_in" binding:"required,bookabledate" time_format:"2006-01-02"`
	CheckOut time.Time `form:"check_out" binding:"required,gtfield=CheckIn,bookabledate" time_format:"2006-01-02"`
}

var bookableDate validator.Func = func(fl validator.FieldLevel) bool {
	date, ok := fl.Field().Interface().(time.Time)
	if ok {
		today := time.Now()
		if today.After(date) {
			return false
		}
	}
	return true
}

var trans ut.Translator

func main() {
	route := gin.Default()

	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
		v.RegisterValidation("bookabledate", bookableDate)

		tw := zh_Hant_TW.New()
		en := en.New()

		uni := ut.New(en, tw, en)

		trans, _ = uni.GetTranslator("zh_Hant_TW")
		zh_tw_translations.RegisterDefaultTranslations(v, trans)
	}

	route.GET("/bookable", getBookable)
	route.Run(":8085")
}

func getBookable(c *gin.Context) {
	var b Booking
	if err := c.ShouldBindWith(&b, binding.Query); err == nil {
		c.JSON(http.StatusOK, gin.H{"message": "Booking dates are valid!"})
	} else {
		errs := err.(validator.ValidationErrors)
		c.JSON(http.StatusBadRequest, gin.H{"error": errs.Translate(trans)})
	}
}
```
測試
```
$ curl "localhost:8085/bookable?check_in=2018-04-16&check_out=2018-04-15"
{
    "error":{
        "Booking.CheckIn":"Key: 'Booking.CheckIn' Error:Field validation for 'CheckIn' failed on the 'bookabledate' tag",
        "Booking.CheckOut":"CheckOut必須大於CheckIn"
    }
}
```

從 [validator 包的 errors.go](https://github.com/go-playground/validator/blob/master/errors.go) 可以看到，
`errs.Translate(trans)` 回傳的是 `ValidationErrorsTranslations`，它的型別是 `map[string]string`，key 為 `ns`，value 為翻譯後的錯誤訊息
```go
// 節錄
type ValidationErrorsTranslations map[string]string
type ValidationErrors []FieldError

func (ve ValidationErrors) Translate(ut ut.Translator) ValidationErrorsTranslations {

	trans := make(ValidationErrorsTranslations)

	var fe *fieldError

	for i := 0; i < len(ve); i++ {
		fe = ve[i].(*fieldError)

		// // in case an Anonymous struct was used, ensure that the key
		// // would be 'Username' instead of ".Username"
		// if len(fe.ns) > 0 && fe.ns[:1] == "." {
		// 	trans[fe.ns[1:]] = fe.Translate(ut)
		// 	continue
		// }

		trans[fe.ns] = fe.Translate(ut)
	}

	return trans
}
```

## 自訂錯誤訊息
現在來給驗證器 `bookabledate` 定義一個錯誤訊息

我們可以改寫上面的 `Translate()`，如果 struct tag 是我們自訂的驗證器，則回傳對應的錯誤訊息

複習一下 interface value，是由 value 和 concrete type 組成的 tuple，所以呼叫 interface 的 method，等於呼叫 concrete type 的同名 method

也就是說，我們可以呼叫 `FieldError` 的 `Namespace()` 和 `StructField()` 等 method 取得 `fieldError` 的 struct field
```go
func getBookable(c *gin.Context) {
	var b Booking
	if err := c.ShouldBindWith(&b, binding.Query); err == nil {
		c.JSON(http.StatusOK, gin.H{"message": "Booking dates are valid!"})
	} else {
		errs := err.(validator.ValidationErrors)
		c.JSON(http.StatusBadRequest, gin.H{"error": getErrorsTrans(errs)})
	}
}

func getErrorsTrans(ve validator.ValidationErrors) validator.ValidationErrorsTranslations {
	trans := make(validator.ValidationErrorsTranslations)

	for i := 0; i < len(ve); i++ {
		trans[ve[i].Namespace()] = getErrorMessage(ve[i])
	}

	return trans
}

func getErrorMessage(fe validator.FieldError) string {
	switch fe.Tag() {
	case "bookabledate":
		return fe.StructField() + "的時間必須大於今天"
	}
	return fe.Translate(trans)
}
```
測試
```
$ curl "localhost:8085/bookable?check_in=2018-04-16&check_out=2028-04-17"
{"error":{"Booking.CheckIn":"CheckIn的時間必須大於今天"}}
```


## 參考資料
- [validator库参数校验若干实用技巧](https://www.liwenzhou.com/posts/Go/validator-usages/)
- [go-playground/validator: :100:Go Struct and Field validation, including Cross Field, Cross Struct, Map, Slice and Array diving](https://github.com/go-playground/validator)
- [How can I define custom error message? · Issue #559 · go-playground/validator](https://github.com/go-playground/validator/issues/559#issuecomment-976459959)
