# Unit Testing

Testing plugin code in small, isolated units is distinct from Acceptance Tests, and does not require network connections. Unit tests are commonly used for testing helper methods that expand or flatten API responses into data structures for storage into state by Terraform. This section covers the specifics of writing Unit Tests for Terraform Plugin code.

The procedure for writing unit tests for Terraform follows the same setup and conventions of writing any Go unit tests. We recommend naming tests to follow the same convention as our acceptance tests, `Test<Provider>_`. For more information on Go tests, see the [official docs][0]. Here is an example

```go
package example

import (
 "fmt"
 "reflect"
 "testing"
)

func TestExampleResource_flattens(t *testing.T) {
 input := &ProviderResponse{
   Settings: &Settings{
     Active: true,
   },
   LogLevel: &LogLevel{
     Level: 42,
   },
 }

 expected := map[string]interface{} {
   "settings_active": true,
   "log_level": 42,
 }

 result := flattenResponse(input)

 if !reflect.DeepEqual(expected, result) {
   t.Fatalf("expected: %v, got: %v", expected, result)
 }
}
```

[0]: https://golang.org/pkg/testing/
