## Multi-Language Errors with Golang

Friendly messages should be shown to users if any errors occur. We can achieve this at frontend level. But, maintaining error codes with their meaning is tedious, and backend system may mismatch with frontend. In this blog, I will show the way backend translates errors into multiple languages. This way can remove mismatch between backend and frontend and enable flexibility for adding info into translated messages.

Full code can be found at: https://github.com/liamhieuvu/go-multi-lang-errors

# Build a simple server

Imagine that we are building an API to create a user with some fields, and these fields should be validated by backend. We will use [gin](https://github.com/gin-gonic/gin) framework to build a server to handle user creation requests.

```bash
mkdir multilang
cd multilang

go mod init multilang
touch main.go
```

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type user struct {
	Name  string
	Age   uint8
	Email string
}

func main() {
	r := gin.Default()
	r.POST("/users", createUser)
	_ = r.Run()
}

func createUser(c *gin.Context) {
	var u user
	if err := c.ShouldBindJSON(&u); err != nil {
		c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{"message": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"message": "successful"})
}
```

For simplification, `createUser` function only returns successful message and does nothing. We will focus to input validation and its error translation to different languages. Now, we can call API like this:

```bash
curl --request POST 'localhost:8080/users' \
--header 'Content-Type: application/json' \
--data-raw '{ "name": "Liam" }'
# output: {"message":"successful"}
```

# Add input validation

We will use [validator](https://github.com/go-playground/validator) package to check input with `struct tags`. We want:

+ User must provide his name
+ User age must be between 10 and 90 years old
+  User must provide valid email

```go
import (
	...
	"github.com/go-playground/validator/v10"
)

type user struct {
	Name  string `validate:"required"`
	Age   uint8  `validate:"gte=10,lte=90"`
	Email string `validate:"required,email"`
}

var validate *validator.Validate

func main() {
	setup()
	...
}

func createUser(c *gin.Context) {
	var u user
	if err := c.ShouldBindJSON(&u); err != nil { ... }

	if err := validate.Struct(u); err != nil {
		c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{"rawMessage": err.Error()})
		return
	}
	...
}

func setup() {
	validate = validator.New()
}
```

If we try to create a user without email, we will receive a non-friendly message:

```bash
curl --request POST 'localhost:8080/users' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "Liam",
    "age": 30
}'
# output: {"rawMessage":"Key: 'user.Email' Error:Field validation for 'Email' failed on the 'required' tag"}
```

# Define error translation

We will use [universal-translator](github.com/go-playground/universal-translator) to translate the non-friendly error messages to English and Vietnamese. In [validator](https://github.com/go-playground/validator), the `Validate` struct has this method:

```go
func (v *Validate) RegisterTranslation(
	tag string,
	trans ut.Translator,
	registerFn RegisterTranslationsFunc,
	translationFn TranslationFunc,
) (err error) {
```

This method requires a `tag` (required, gte, email, etc. supported by [validator](https://github.com/go-playground/validator)) we want to translate, a translation engine `trans`, a dictionary `registerFn`, and the the way to translate `translationFn`. Next, we will define `translations` to translate sentence, `dicts` to translate field name, and `translationFunc` to define how to translate.

```go
// trans.go
import (
	ut "github.com/go-playground/universal-translator"
	"github.com/go-playground/validator/v10"
)

var translations = map[string]map[string]string{
	"en": {
		"required": "{0} is a required field.",
		"email":    "{0} is invalid.",
		"gte":      "{0} must be {1} or greater.",
		"lte":      "{0} must be {1} or smaller.",
	},
	"vi": {
		"required": "{0} là trường bắt buộc.",
		"email":    "{0} không hợp lệ.",
		"gte":      "{0} phải bằng hoặc lớn hơn {1}.",
		"lte":      "{0} phải bằng hoặc nhỏ hơn {1}.",
	},
}

var dicts = map[string]map[string]string{
	"vi": {
		"Name": "Tên",
		"Age":  "Tuổi",
	},
}

func translationFunc(t ut.Translator, fe validator.FieldError) string {
	field, err := t.T(fe.Field())
	if err != nil {
		field = fe.Field()
	}
	msg, err := t.T(fe.Tag(), field, fe.Param())
	if err != nil {
		return fe.Error()
	}
	return msg
}
```

In the `translationFunc` function, we first try to translate the field name, then we translate the sentence with this field name as `{0}` and `fe.Param()` as `{1}` (if existed, e.g. the param is 90 if validating with lte=90). Our work is simple, register `translations` and `dicts` to the engine and `translationFunc` to our validator:

```go
// main.go
import (
	...
	"github.com/go-playground/locales/en"
	"github.com/go-playground/locales/vi"
	ut "github.com/go-playground/universal-translator"
)
...
func setup() {
	validate = validator.New()
	enLocale := en.New()
	utrans = ut.New(enLocale, enLocale, vi.New())

	for locale, dict := range dicts {
		engine, _ := utrans.FindTranslator(locale)
		for key, trans := range dict {
			_ = engine.Add(key, trans, false)
		}
	}

	for locale, translation := range translations {
		engine, _ := utrans.FindTranslator(locale)
		for tag, trans := range translation {
			_ = validate.RegisterTranslation(tag, engine, func(t ut.Translator) error {
				return t.Add(tag, trans, false)
			}, translationFunc)
		}
	}
}
```

# Apply error translation

We will apply the translation to the user creation request. We expect to see a friendly message if there are any errors.

```go
// main.go
...
func createUser(c *gin.Context) {
	var u user
	if err := c.ShouldBindJSON(&u); err != nil { ... }

	if err := validate.Struct(u); err != nil {
		transErrs := err.(validator.ValidationErrors).Translate(getTransFromParam(c)) // 1
		c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{
			"message":    getErrMsg(transErrs), // 2
			"rawMessage": err.Error(),
		})
		return
	}
	...
}
```

If error, we will translate the error according to `locale` param, which is `en` or `vi`. Then, we merge translated errors to 1 message.

```go
// trans.go
func getTransFromParam(c *gin.Context) ut.Translator {
	t, found := utrans.GetTranslator(c.Query("locale"))
	if !found {
		t, _ = utrans.GetTranslator("en")
	}
	return t
}

func getErrMsg(errs validator.ValidationErrorsTranslations) string {
	messages := make([]string, 0, len(errs))
	for _, v := range errs {
		messages = append(messages, v)
	}
	return strings.Join(messages, " ")
}
```

It's done. Let's check the result:

```bash
curl --request POST 'localhost:8080/users?locale=vi' \
--header 'Content-Type: application/json' \
--data-raw '{
    "age": 100,
    "email": "liam@gmailcom"
}'
# {"message":"Tên là trường bắt buộc. Tuổi phải bằng hoặc nhỏ hơn 90. Email không hợp lệ."}

curl --request POST 'localhost:8080/trans/users?locale=en' \
--header 'Content-Type: application/json' \
--data-raw '{
    "age": 100,
    "email": "liam@gmailcom"
}'
# {"message":"Name is a required field. Age must be 90 or smaller. Email is invalid."}
```

Thank you for reading my blog.

