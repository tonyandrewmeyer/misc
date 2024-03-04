# Versions

## Juju Version

The Juju version uses [semantic versioning](https://semver.org/), so is a string in the format
major.minor.patch, where each section is a positive integer (which may be higher than 9).

### Agent

You can see the version of Juju used in a model in `juju status`, in the top section under
"Version" in the tabular view, or `model.version` in JSON or YAML output. You can also use the
`juju model-config -m controller agent-version` command.

To upgrade the patch version of Juju (e.g. 3.3.0 to 3.3.1), you run
[`juju upgrade-controller`](https://juju.is/docs/juju/juju-upgrade-controller), specifying the
patch version. To upgrade minor or major versions, you need to create a new controller and
[migrate](https://juju.is/docs/juju/juju-migrate) the models to it.

### CLI

You can see the version of the Juju CLI tool using `juju --version`. When bootstrapping a new
controller, this is the version of the agent that will be used, unless the `--agent-version` option
is provided.

To upgrade your local Juju version, use the same tool you used to install it. For example, if you
installed Juju with snap, you can run `snap refresh juju --channel=3.3/stable` to upgrade to
Juju 3.3.

## Charms

To upgrade the version or revision of an installed charm,
[use `juju refresh`](https://juju.is/docs/juju/manage-charms#heading--update-a-charm) for both
local and Charmhub charms.

### Charm Version

Charms can optionally specify a version of the charm itself. You can find this information with
`juju status` in the YAML or JSON formats in the `applications.<app name>.charm-version` field. If
there is no version, the key will not be present in the status output.

The version is an arbitrary string, so could be a major.minor.patch style version, a version control
hash identifier, or anything else.

To set the version, charms include a file called `version` (no extension) at the top level of the
charm (the same folder as the `charmcraft.yaml` file), where the content of the file is the version
string.

In the past, the version would be set automatically if the charm was deployed from a known version
control system, but that is no longer the case. The intention is that charmers set this version to
allow tracking the installed version of the charm back to the source tree that it was built from.

### Charm Revision

All charms have a revision, which is a simple positive integer. You can see the revisions available
of a charm published to Charmhub with `charmcraft revisions <charm name>`, and in the drop-down near
the top of a charm's page in Charmhub. You can see the installed revision of a charm in
`juju status` in the application table in the "Rev" column in the tabular view, or in the
`applications.<app name>.charm-rev` field in JSON or YAML format.

The revision is automatically set when uploading the charm using `charmcraft upload`. Note that
revisions may be unreleased, or may be released only for specific
[channels](https://juju.is/docs/sdk/charmcraft-release).

When a charm is deployed locally, Juju will use an initial revision of 1 and then increment the
revision each time a new version of the same charm is deployed or refreshed.

### Libraries

Charm libs have both an "API version" and a "patch version". Any changes to the lib API that are not
backwards compatible will be made in a new API version, while new patch versions include compatible
improvements including new features, general improvements, and bug fixes.

The available versions for a charm lib can be found on Charmhub in the "Libraries" tab. These can
also be found using `charmcraft list-lib <charm name>`.

The API and patch versions of a charm lib are contained within the lib source file itself. Search
for `LIBAPI` (the API version) and `LIBPATCH` (the patch version) within the source.

It is not possible to see which version of a lib was used in a deployed charm, other than by using
the `charm-version` to examine the source tree from which the charm was built or retrieving the
deployed charm source from a unit. In general, the lib version is a charm author concern, rather
than one of an admin.

Authors publish new versions of charm libraries using the
[charmcraft publish-lib command](https://juju.is/docs/sdk/create-and-publish-a-charm-library#heading--update-a-charm-library),
setting the [metadata](https://juju.is/docs/sdk/library#heading--structure) appropriately - in
particular, the `LIBAPI` value for the API version (which is also in the path) and the `LIBPATCH`
value for the patch version.

To upgrade the revision of a lib used by a charm, the same `charmcraft fetch-lib` command is used
as when originally installing the lib. To upgrade the version of the lib, use `fetch-lib` but
change the version number in the path, for example from `charms.demo.v0.demo` to
`charms.demo.v1.demo`. See the
[documentation](https://juju.is/docs/sdk/find-and-use-a-charm-library#heading--update-a-charm-library)
for more details.

### Interfaces

When integrating two charms together, it's important that both charms understand what the other
side of the relation requires and provides. In the charm metadata this is declared using the
[interface field of the relation](https://juju.is/docs/sdk/charmcraft-yaml#heading--peers-provides-requires).

Interfaces are more meaningful when they are
[registered](https://juju.is/docs/sdk/register-an-interface), which involves defining the data
structures that each side of the relation will use in the relation databag, and creating a suite of
tests that ensure that charms that require or provide the interface align with the specification.

Interface versions are defined through the folder structure in the
[charm-relation-interfaces repository](https://github.com/canonical/charm-relation-interfaces/blob/main/interfaces/)
and are updated by authors using pull requests.

## Workload / Application

Most applications modelled by charms will also have a version - for example, the version of
[Wordpress](https://charmhub.io/wordpress-k8s), [PostgreSQL](https://charmhub.io/postgresql-k8s), or
[Jenkins](https://charmhub.io/jenkins).

Each application will have its own versioning scheme, and
its own way of accessing that information. To make things easier for admins, the charm can expose
the workload version through Juju. You can find the workload (application) version with
`juju status` in the application table in the "Version" column in the tabular view, or in the
`applications.<app name>.version` field in JSON or YAML format. If the charm has not set the
workload version, then the field will not be present in JSON or YAML format, and if the version
string is too long or contains particular characters then it will not be displayed in the tabular
format.

To set the workload version, charms call
[`Unit.set_workload_version()`](https://ops.readthedocs.io/en/latest/#ops.Unit.set_workload_version)
providing the appropriate string (which is generally retrieved from the workload application
itself).

Upgrading the workload version is handled by the charm - typically the charm has been built and
tested against a specific version of the application, so the charm revision will be tied to that
version of the application.
