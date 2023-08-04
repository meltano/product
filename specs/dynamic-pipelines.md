# Dynamically create N number of pipelines from Singer output

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
> TBD: do we want to support config sources for loaders too? If so, how could we ensure bot collections of configs have the same cardinality and order?

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
