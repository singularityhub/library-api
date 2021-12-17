# Library API

This is a specification for the Library API that was originally implemented
by Sylabs in the [singularity](https://github.com/sylabs/singularity) project,
and now is present in the following container registry implementations:

 - [Singularity Registry Server](https://github.com/singularityhub/sregistry)
 - [Hinkskalle Registry](https://github.com/csf-ngs/hinkskalle)
 - [Sylabs Cloud Library](https://cloud.sylabs.io/library)
 - [Singularity Enterprise](https://sylabs.io/singularity-enterprise/)
 
The specification here is a community effort to document and maintain this application
programming interface. To support this effort, Sylabs has contributed the
[OpenAPI specification](https://github.com/sylabs/scs-library-client/blob/master/api/server/openapi.yml)
on which their services are based.

## Who is this intended for?

The library API is a flexible application programming interface that describes interactions
between a client and a registry entity. While we generally encourage implementations to use
the [open-containers distribution spec](https://github.com/opencontainers/distribution-spec), 
this application programming interface can be desired if you need endpoints with extended
metadata, or want to support the existing tools that use it.

## Specification

The [specification](spec/) describes in more detail the endpoints provided by the
application programming interface. 

## Implementations

### Registry

The following non-commercial implementations of the spec exist:

 - [Singularity Registry Server](https://github.com/singularityhub/sregistry) and [documentation for the client interaction](https://singularityhub.github.io/sregistry/docs/client)
 - [Hinkskalle Registry](https://github.com/csf-ngs/hinkskalle)
 - [Sylabs Cloud Library](https://cloud.sylabs.io/library)

The following commercial implementations exist:

 - [Singularity Enterprise](https://sylabs.io/singularity-enterprise/)

### Clients

An open source Go client is available, which an be used to interact with `library://` endpoints:

- [scs-library-client](https://pkg.go.dev/github.com/sylabs/scs-library-client/client)

The `scs-library-client` module is used in both versions of the original Singularity software:

 - [sylabs/singularity](https://github.com/sylabs/singularity) 
 - [apptainer/singularity](https://github.com/apptainer/singularity) 
 
If you know of other implementations of registries or clients (the writer of this documentation
seems to recall some that she cannot remember) please [let us know](https://github.com/singularityhub/library-api/issues)
or [open a pull request](https://github.com/singularityhub/library-api) to add it.

## Community standard

Since this is a community standard, anyone in the community is invited to participate!
Suggestions for improvements are welcome through
[github issues](https://github.com/singularityhub/library-api/issues)
or [pull requests](https://github.com/singularityhub/library-api/pulls).
Changes to the specification will be reviewed and approved by people from the community
as described in the [community charter](community_charter.md).
