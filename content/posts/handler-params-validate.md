---
title: "Custom API Calls params validator with go validator"
draft: false
date: "2020-11-27"
---

 ## In this article, we are going to explore how to make custom validator with `go validator`

 *This custom validator is made with my best friend and coding buddy Gligor - link to his github account https://github.com/Gligor23*


### 1. Custom ValidationError Structure file.
- First we need to create our custom ValidationError structure 

```Go
// ValidationError represents custom validation error structure
// using this error when validating request body
type ValidationError struct {
	Message          string                     `json:"message"`
	Errors           map[string]string          `json:"errors,omitempty"`
	ValidationErrors validator.ValidationErrors `json:"-"`
}
```
**Care we need to use `go validator v10` - github.com/go-playground/validator/v10**


- Constructor-like factory function for creating our NewValidationError


```Go
// NewValidationError creates new ValidationError
func NewValidationError(ve validator.ValidationErrors) *ValidationError {
	validationError := &ValidationError{
		Message: "some generic message",
	}

	if ve != nil {
		validationError.ValidationErrors = ve
		validationError.FormatErrors()
	}

	return validationError
}
```


- FormatErrors function creates key and message for each validation error

```Go
// FormatErrors creates key and message for each validation error
// be careful when using err.Param() only use it on tags with param value (ex: max=1)
func (q *ValidationError) FormatErrors() {
	q.Errors = make(map[string]string)

	for _, err := range q.ValidationErrors {
		switch err.ActualTag() {
		case "email":
			q.Errors[err.Field()] = "Invalid email format"
		case "required":
			q.Errors[err.Field()] = "This field is required"
		case "max":
			q.Errors[err.Field()] = "Max length allowed is " + err.Param()
		case "min":
			q.Errors[err.Field()] = "Must be at least " + err.Param() + " character long"
		case "alphanum":
			q.Errors[err.Field()] = "Only alphanumeric characters allowed"
		case "oneof":
			q.Errors[err.Field()] = "Must be one of: " + err.Param()
		case "html":
			q.Errors[err.Field()] = "Content must be html"
		default:
			q.Errors[err.Field()] = "Validation failed on condition: " + err.ActualTag()
		}
	}
}
```


### 2. We are all setup to register our custom validator 

- We can make global var for our validator


```Go
// MyCustomValidator global validator
var MyCustomValidator *validator.Validate
```

- Constructor-like function

```Go
// Validator is a constructor for our validator
// if our validator is once created it returns it else it creates it
func Validator() *validator.Validate {
	sync.Once.Do(func() {
		initValidator()
	})

	return MBValidator
}
```

- initValidator() initializes validator

```Go
// Using "validate" in structure to validate param and use name of param from form:"bla"
func initValidator() {
	MBValidator = validator.New()
	MBValidator.SetTagName("validate")
	MBValidator.RegisterTagNameFunc(func(fld reflect.StructField) string {
		name := strings.SplitN(fld.Tag.Get("form"), ",", 2)[0]
		return name
	})
}
```

- We can  create our own tagName for validating our own params 

Let's create custom tag validator which will validate if our param is made from AlphaNumericHypen characters.

* First we need to add custom error message in `FormatErrors()` function 

```Go
case "alphanumhyphen":
q.Errors[err.Field()] = "Must consist only of alphanumeric and hyphen characters"
```

* Second we need to create  function which will validate our custom tag.

```Go
func(fl validator.FieldLevel) ValidateAlphaHypenNumTag bool {
	matched, _ := regexp.MatchString("^[\\w-]*$", fl.Field().String())
		return matched
}
```

* Now we can register our custom tag validator in our custom validator `initValidator()`

```Go
// Using "validate" in structure to validate param and use name of param from form:"bla"
func initValidator() {
	MBValidator = validator.New()
	MBValidator.SetTagName("validate")
	MBValidator.RegisterTagNameFunc(func(fld reflect.StructField) string {
		name := strings.SplitN(fld.Tag.Get("form"), ",", 2)[0]
		return name
	})

	// RegisterValidation needs tag name and function which will do the validation.
	err := MBValidator.RegisterValidation("alphanumhyphen", b ValidateAlphaHypenNumTag) {
		return b
	})
	if err != nil {
		// if register validation fails panic
		panic(err)
	}
}
```


### 3. Let's finally use our custom validator

- Create param structure for PostAccount() handler.

```Go
// PostAccount represents request body for POST /api/account
type PostAccount struct {
	Name         string            `form:"name" 		        validate:"required,max=191"`
	Email 		 string            `form:"email"                validate:"required,email,max=191"`
	Password	 string            `form:"password" 	        validate:"required,min=8"`
	Metadata     map[string]string `form:"metadata"             validate:"omitempty,dive,keys,required,alphanumhyphen,endkeys,required"`

}
```

- Create PostAccount() Handler.

```Go
func PostAccount() {
	bodyParams := &params.PostAccount{}
	// Fil bodyParams from form body or whatever you are using for binding.

	// use our custom validator to validate body params.
	if err := validator.Validate(body); err != nil {
		// return response to end user, and the error
		return
	}
}
```

* Error which will be shown to end user will be descriptive and easy to understand.
-- if he fail `Password` lenght or incorrect `email` he will recieve error message with both of errors:
```JSON
	"code": 400
	"message" : "Invalid parameters, please try again"
	"errors": {
			   "email": "Invalid email format", 
			   "password": "Must be at least 8 character long"
			   })
```

*Feel free to contact me if you need any help.*