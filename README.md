

![Pipeline](https://github.com/pb33f/libopenapi-validator/workflows/Build/badge.svg)
[![codecov](https://codecov.io/gh/pb33f/libopenapi-validator/branch/main/graph/badge.svg?)](https://codecov.io/gh/pb33f/libopenapi-validator)
[![discord](https://img.shields.io/discord/923258363540815912)](https://discord.gg/x7VACVuEGP)
[![Docs](https://img.shields.io/badge/godoc-reference-5fafd7)](https://pkg.go.dev/github.com/pb33f/libopenapi)


# libopenapi-validator

Full Documentation coming shortly, now ready for early testing.

## Examples

### Validating a document

```go
// 1. Load the OpenAPI 3+ spec into a byte array
petstore, err := os.ReadFile("test_specs/invalid_31.yaml")

if err != nil {
    panic(err)
}

// 2. Create a new OpenAPI document using libopenapi
document, docErrs := libopenapi.NewDocument(petstore)

if docErrs != nil {
    panic(docErrs)
}

// 3. Create a new validator
docValidator, validatorErrs := NewValidator(document)

if validatorErrs != nil {
    panic(validatorErrs)
}

// 4. Validate!
valid, validationErrs := docValidator.ValidateDocument()

if !valid {
    for i, e := range validationErrs {
        // 5. Handle the error
        fmt.Printf("%d: Type: %s, Failure: %s\n", i, e.ValidationType, e.Message)
        fmt.Printf("Fix: %s\n\n", e.HowToFix)
    }
}
// Output: 0: Type: schema, Failure: Document does not pass validation
//Fix: Ensure that the object being submitted, matches the schema correctly
```


### Validating `*http.Request` and `*http.Response`

```go
// 1. Load the OpenAPI 3+ spec into a byte array
petstore, err := os.ReadFile("test_specs/petstorev3.json")

if err != nil {
    panic(err)
}

// 2. Create a new OpenAPI document using libopenapi
document, docErrs := libopenapi.NewDocument(petstore)

if docErrs != nil {
    panic(docErrs)
}

// 3. Create a new validator
docValidator, validatorErrs := NewValidator(document)

if validatorErrs != nil {
    panic(validatorErrs)
}

// 6. Create a new *http.Request (normally, this would be where the host application will pass in the request)
request, _ := http.NewRequest(http.MethodGet, "/pet/findByStatus?status=sold", nil)

// 7. Simulate a request/response, in this case the contract returns a 200 with an array of pets.
// Normally, this would be where the host application would pass in the response.
recorder := httptest.NewRecorder()
handler := func(w http.ResponseWriter, r *http.Request) {

    // set return content type.
    w.Header().Set(helpers.ContentTypeHeader, helpers.JSONContentType)
    w.WriteHeader(http.StatusOK)

    // create a Pet
    body := map[string]interface{}{
        "id":   123,
        "name": "cotton",
        "category": map[string]interface{}{
            "id":   "NotAValidPetId", // this will fail, it should be an integer.
            "name": "dogs",
        },
        "photoUrls": []string{"https://pb33f.io"},
    }

    // marshal the request body into bytes.
    responseBodyBytes, _ := json.Marshal([]interface{}{body}) // operation returns an array of pets
    // return the response.
    _, _ = w.Write(responseBodyBytes)
}

// simulate request/response
handler(recorder, request)

// 8. Validate!
valid, validationErrs := docValidator.ValidateHttpRequestResponse(request, recorder.Result())

if !valid {
    for _, e := range validationErrs {
        // 5. Handle the error
        fmt.Printf("Type: %s, Failure: %s\n", e.ValidationType, e.Message)
        fmt.Printf("Schema Error: %s, Line: %d, Col: %d\n",
            e.SchemaValidationErrors[0].Reason,
            e.SchemaValidationErrors[0].Line,
            e.SchemaValidationErrors[0].Column)
    }
}
```
Will print: 

```
Type: response, Failure: 200 response body for '/pet/findByStatus' failed to validate schema
Schema Error: expected integer, but got string, Line: 19, Col: 27
```
