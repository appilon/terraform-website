---
layout: "extend"
page_title: "Home - Extending Terraform"
sidebar_current: "docs-extend-detecting-drift"
description: |-
  Extending Terraform is a section for content dedicated to developing Plugins
  to extend Terraform's core offering.
---


# Detecting Drift

One of the core challenges of infrastructure as code is keeping a consistent mapping of what you have deployed, terraform manages this in it’s state (which can be stored locally or remotely).

When developing a provider it’s important that whatever the provider API reports for a particular resource is consistently stored in terraform state, otherwise when terraform plan/apply is subsequently run it will detect change, this is what we refer to as Drift. When developing a new provider and resources, there are some common scenarios to handle to avoid Drift.

## Call READ after CREATE 

At this point you should be familiar with fact that provider plugins are developed as CRUD operations with a function for each operation. The one when a new resource is being created is obviously the `CREATE` method. Here is a hypotethical config.

```hcl
resource "simple_resource" "example" {
    name = "example"
    # omitted argument that the API defaults to some value if left unspecified
    # type = "simple"
}
```

The associated provider code might look like this

```go
func simpleResourceRead(d *schema.ResourceData, meta interface{}) error {
    client := meta.(*ProviderApi).client
    resource, _ := client.GetResource(d.Get("name").(string))
    d.Set("name", resource.Name)
    d.Set("type", resource.Type)
    return nil
}

func simpleResourceCreate(d *schema.ResourceData, meta interface{}) error {
    client := meta.(*ProviderApi).client
    client.CreateResource(d.Get("name").(string))
    return nil
}
```

The problem with this, is that our `CREATE` function does not synchronize the API state into the terraform state after creation. As mentioned in the config comment, the `type` is set to some default by the provider API if omitted on creation. Furthermore provider APIs can mutate values passed to them. Say the config contained capital letters in the values set, but the provider API converts all values to lowercase, again our terraform state is taking the user’s config as the source of truth, when in reality we should take the provider API as the source of truth. The common practice is to call `READ` at the end of `CREATE` or `UPDATE`, ensuring state is captured right away.

```go
func simpleResourceCreate(d *schema.ResourceData, meta interface{}) error {
    client := meta.(*ProviderApi).client
    client.CreateResource(d.Get("name").(string))
    return simpleResourceRead(d, meta)
}
```

## Capture all state in READ

It was hinted at above, but it is important that all attributes are set during the `READ` function and all set values have been defined in the schema.

 ## Error checking aggregate types

 So far we have described a very simple resource with very trivial types, often however the schema/configuration is setup to nest the same way the provider’s data structures nest for consistency, or the attribute is a list, for those we rely on `TypeList`, `TypeMap`, and `TypeSet`. Those types need to be handled with care when setting the state as they can error.

 ```go
 d.Set("tags", map[string]string{"name": "test"})
 ```

 Should be

 ```go
 if err := d.Set("tags", map[string]string{"name": "test"}); err != nil {
    return fmt.Errorf("error setting tags in state %s", err) 
 }
 ```

 Without error checking terraform will run successfully but with broken state. The same goes for error checking API calls.

## Resources modified externally

In production a common reason for drift is resources have been modified externally (say someone changed some values in a web dashboard). In this circumstance drift is expected, to reconcile this you could try importing the changed resources as a new resource, or updating your config to reflect the changes. Ultimately though it’s important that terraform be authoritative source of modifications and state.

## Handling Drift

As mentioned, there are some situations where drift is okay. For instance configuration that contains uppercase letters but the provider API downcases all data. This would cause constant a diff, a situation like is is easily resolved with the help of some of the [schema functions][0].

When operating infrastructure, to help keep terraform the source of truth on the state of provisioned resources, [this article][1] describes a way terraform itself can be used to automatically detect drift.

[0]: https://www.terraform.io/docs/extend/schemas/schema-behaviors.html#function-behaviors
[1]: https://medium.com/build-acl/state-drift-detection-using-terraform-d0383628d2ea
