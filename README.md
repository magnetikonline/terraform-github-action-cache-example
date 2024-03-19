# Terraform GitHub Actions caching example

[![Terraform provider cache example](https://github.com/magnetikonline/terraform-github-action-cache-example/actions/workflows/example.yaml/badge.svg)](https://github.com/magnetikonline/terraform-github-action-cache-example/actions/workflows/example.yaml)

## Summary

An implementation of caching [Terraform providers](https://developer.hashicorp.com/terraform/language/providers) via [`actions/cache`](https://github.com/actions/cache) within a workflow run in an attempt to improve `terraform init|plan|apply` execution times.

Why?

- Providers are external to Terraform itself and require download during `terraform init` operations.
- Common use providers can often be very large in size. For example, the [AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) at time of writing weighs in around `360MB` uncompressed.
- By caching these provider binaries between GitHub Action runs, we hope to have required configuration providers available to `terraform` sooner!

## Example

See: [`.github/workflows/example.yaml`](.github/workflows/example.yaml)

Breakdown of the key workflow steps:

- Git source is fetched and Terraform setup via [`actions/checkout`](https://github.com/actions/checkout) and [`hashicorp/setup-terraform`](https://github.com/hashicorp/setup-terraform) respectively.
- Terraform plugin (provider) cache path is configured:
	- By default Terraform downloads providers to individual `.terraform/` directories alongside a configuration. By enabling a system wide cache, `terraform` downloads each provider _once_ to a central location and symlink back into each `.terraform/` directory - avoiding repeated downloads. [More details here](https://developer.hashicorp.com/terraform/cli/config/config-file#provider-plugin-cache).
	- The plugin cache path is set via the [`TF_PLUGIN_CACHE_DIR`](https://developer.hashicorp.com/terraform/cli/config/environment-variables#tf_plugin_cache_dir) environment variable.
	- With a global plugin cache location enabled, we've now got the perfect candidate for workflow run caching.
- Finally, [`actions/cache`](https://github.com/actions/cache) is setup to save/restore the plugin cache location we've set at `~/.terraform.d/plugin-cache`. The cache key is composed using a hash of dependency lock files (`.terraform.lock.hcl`), introduced in [Terraform 0.14](https://developer.hashicorp.com/terraform/language/files/dependency-lock).

	Configuration lock file can be created/updated via the following:

	```sh
	$ terraform providers lock -platform=darwin_amd64 -platform=darwin_arm64 -platform=linux_amd64
	```

- Finally a `terraform init` is run, which sets up a trivial Terraform configuration of [`main.tf`](main.tf).
