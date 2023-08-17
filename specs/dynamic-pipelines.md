# Dynamically create N number of pipelines from Singer output

## Problem

Singer taps are designed to be used in a single pipeline, with a single configuration. However, there are cases where a single tap can be used to extract data from different instances of the same type of source. There are two ways to achieve this today:

1. Use a single plugin, and pass different configurations to it at runtime:

   ```sh
    # Set env vars for the tap
    export TAP_POSTGRES_HOST=...
    export TAP_POSTGRES_PORT=...
    export TAP_POSTGRES_USER=...
    export TAP_POSTGRES_PASSWORD=...
    export TAP_POSTGRES_DATABASE=...
    # Run the tap
    meltano run tap-postgres target-snowflake
    ```

   This process can be automated by using a Secret Ops provider like [Infisical](https://github.com/Infisical/infisical) or [chamber](https://github.com/segmentio/chamber) or writing a script that generates the required env vars for each configuration.

2. Use inheritance to create a new plugin for each configuration:

   ```yaml
   plugins:
    extractors:
    - name: tap-postgres
    - name: tap-postgres--tenant1
      inherit_from: tap-postgres
    - name: tap-postgres--tenant2
      inherit_from: tap-postgres
   ```

   This approach does not scales to 100s or 1000s of instances, as it requires a new plugin to be created for each configuration.

## `meltano.yml`

```yaml
plugins:
  extractors:
  # Third-party data source
  - name: tap-shopify

  # Configuration sources
  - name: tap-postgres--shopify_configs
    inherit_from: tap-postgres
    select: 
    - public-shopify_configs.*

  # Third-party destination
  loaders:
  - name: target-s3-parquet

schedules:
- name: sync-all-shopify
  interval: @hourly
  config_source:
    tap-shopify: tap-postgres--shopify_configs
  extractor: tap-shopify
  loader: target-s3-parquet  
```

Where each key in the `config_source` mapping is a plugin used in the schedule and each value is a **tap** name.

> [!NOTE]
> TBD: how well the above configuration spec plays with _jobs_ since a job name can be referenced in a schedule.

> [!NOTE]
> TBD: how to reference a _pipeline_ instead of a plain tap, in case a mapper is required, for example.

> [!NOTE]
> TBD: This design makes sense for extractors. Do we want to support config sources for loaders too? If so, how could we ensure both collections of configs have the same cardinality and order?

## Under the hood

By writing dynamic configurations at the _schedule_ level, `meltano run` would be able to invoke the respective plugin with each of the configurations, but it is TBD whether this is expected of `meltano run`, or if it's the responsibility of the orchestrator (e.g. Meltano Cloud).

## Alternatives

* [Annotations](https://docs.meltano.com/concepts/project/#annotations) could be used in the schedule definition:

  ```yaml
  plugins: ... # same as above
  schedules:
  - name: sync-all-shopify
    interval: @hourly
    annotations:
      config_source:
        tap-shopify: tap-postgres--shopify_configs
    extractor: tap-shopify
    loader: target-s3-parquet  
  ```

  This would clearly make it the responsibility of the orchestrator to generate _N_ pipelines.
