# terraform-google-bigquery

This module allows you to create opinionated Google Cloud Platform BigQuery datasets and tables.
This will allow the user to programmatically create an empty table schema inside of a dataset, ready for loading.
Additional user accounts and permissions are necessary to begin querying the newly created table(s).

## Compatibility
This module is meant for use with Terraform 0.13+ and tested using Terraform 1.0+. If you find incompatibilities using Terraform >=0.13, please open an issue.
 If you haven't
[upgraded](https://www.terraform.io/upgrade-guides/0-13.html) and need a Terraform
0.12.x-compatible version of this module, the last released version
intended for Terraform 0.12.x is [v4.5.0](https://registry.terraform.io/modules/terraform-google-modules/-bigquery/google/v4.5.0).

## Upgrading

The current version is 4.X. The following guides are available to assist with upgrades:

- [3.0 -> 4.0](./docs/upgrading_to_bigquery_v4.0.md)
- [2.0 -> 3.0](./docs/upgrading_to_bigquery_v3.0.md)
- [1.0 -> 2.0](./docs/upgrading_to_bigquery_v2.0.md)
- [0.1 -> 1.0](./docs/upgrading_to_bigquery_v1.0.md)

## Usage

Basic usage of this module is as follows:

```hcl
module "bigquery" {
  source  = "terraform-google-modules/bigquery/google"
  version = "~> 10.1"

  dataset_id                  = "foo"
  dataset_name                = "foo"
  description                 = "some description"
  project_id                  = "<PROJECT ID>"
  location                    = "US"
  default_table_expiration_ms = 3600000
  resource_tags               = {"<PROJECT>/<TAG KEY>":"<TAG VALUE>"}

  tables = [
  {
    table_id           = "foo",
    schema             =  "<SCHEMA JSON DATA>",
    time_partitioning  = {
      type                     = "DAY",
      field                    = null,
      require_partition_filter = false,
      expiration_ms            = null,
    },
    table_constraints {

      primary_key { 
        columns = ["id_1", "id_2"] 
      }
      foreign_keys { 
        name = "foreign_key_1"
        referenced_table {
          project_id  = "<PROJECT ID>"
          dataset_id  = "foo"
          table_id    = "table_1"
        }
        column_references {
          referencing_column = "id_1"
          referenced_column = "id"
        }
      }
    },
    range_partitioning = null,
    expiration_time = null,
    clustering      = ["fullVisitorId", "visitId"],
    labels          = {
      env      = "dev"
      billable = "true"
      owner    = "joedoe"
    },
  },
  {
    table_id           = "bar",
    schema             =  "<SCHEMA JSON DATA>",
    time_partitioning  = null,
    range_partitioning = {
      field = "customer_id",
      range = {
        start    = "1"
        end      = "100",
        interval = "10",
      },
    },
    expiration_time    = 2524604400000, # 2050/01/01
    clustering         = [],
    labels = {
      env      = "devops"
      billable = "true"
      owner    = "joedoe"
    }
  }
  ],

  views = [
    {
      view_id    = "barview",
      use_legacy_sql = false,
      query          = <<EOF
      SELECT
       column_a,
       column_b,
      FROM
        `project_id.dataset_id.table_id`
      WHERE
        approved_user = SESSION_USER
      EOF,
      labels = {
        env      = "devops"
        billable = "true"
        owner    = "joedoe"
      }
    }
  ]
  dataset_labels = {
    env      = "dev"
    billable = "true"
  }
}
```

Functional examples are included in the
[examples](./examples/) directory.

### Variable `tables` detailed description

The `tables` variable should be provided as a list of object with the following keys:
```hcl
{
  table_id = "some_id"                        # Unique table id (will be used as ID for table).
  table_name = "Friendly Name"                # Optional friendly name for table. If not set, the "table_id" will be used by default.
  schema = file("path/to/schema.json")        # Schema as JSON string.
  time_partitioning = {                       # Set it to `null` to omit partitioning configuration for the table.
        type                     = "DAY",     # The only type supported is DAY, which will generate one partition per day based on data loading time.
        field                    = null,      # The field used to determine how to create a time-based partition. If time-based partitioning is enabled without this value, the table is partitioned based on the load time. Set it to `null` to omit configuration.
        require_partition_filter = false,     # If set to true, queries over this table require a partition filter that can be used for partition elimination to be specified. Set it to `null` to omit configuration.
        expiration_ms            = null,      # Number of milliseconds for which to keep the storage for a partition.
      },
  range_partitioning = {                      # Set it to `null` to omit partitioning configuration for the table.
    field = "integer_column",                 # The column used to create the integer range partitions.
    range = {
      start    = "1"                          # The start of range partitioning, inclusive.
      end      = "100",                       # The end of range partitioning, exclusive.
      interval = "10",                        # The width of each range within the partition.
        },
      },
  clustering = ["fullVisitorId", "visitId"]   # Specifies column names to use for data clustering. Up to four top-level columns are allowed, and should be specified in descending priority order. Partitioning should be configured in order to use clustering.
  expiration_time = 2524604400000             # The time when this table expires, in milliseconds since the epoch. If set to `null`, the table will persist indefinitely.
  deletion_protection = true                  # Optional. Configures deletion_protection for the table. If unset, module-level deletion_protection setting will be used.
  labels = {                                  # A mapping of labels to assign to the table.
      env      = "dev"
      billable = "true"
    }
}
```

### Variable `views` detailed description

The `views` variable should be provided as a list of object with the following keys:
```hcl
{
  view_id = "some_id"                                                # Unique view id. it will be set to friendly name as well
  query = "Select user_id, name from `project_id.dataset_id.table`"  # the Select query that will create the view. Tables should be created before.
  use_legacy_sql = false                                             # whether to use legacy sql or standard sql
  labels = {                                                         # A mapping of labels to assign to the view.
      env      = "dev"
      billable = "true"
  }
}
```

### Variable `routines` detailed description

The `routines` variable should be provided as a list of object with the following keys:
```hcl
{
  routine_id = "some_id"                     # The ID of the routine. The ID must contain only letters, numbers, or underscores. The maximum length is 256 characters.
  routine_type = "PROCEDURE"                 # The type of routine. Possible values are SCALAR_FUNCTION and PROCEDURE.
  language = "SQL"                           # The language of the routine. Possible values are SQL and JAVASCRIPT.
  definition_body = "CREATE FUNCTION test return x*y;"  # The body of the routine. For functions, this is the expression in the AS clause. If language=SQL, it is the substring inside (but excluding) the parentheses.
  return_type     = null                     # A JSON schema for the return type. Optional if language = "SQL"; required otherwise. If absent, the return type is inferred from definitionBody at query time in each query that references this routine. If present, then the evaluated result will be cast to the specified returned type at query time.
  description = "Description"               # The description of the routine if defined.
  arguments = [                             # Set it to `null` to omit arguments block configuration for the routine.
    {
      name      = "x",                      # The name of this argument. Can be absent for function return argument.
      data_type = null,                     # A JSON schema for the data type. Required unless argumentKind = ANY_TYPE.
      argument_kind = "ANY_TYPE"            # Defaults to FIXED_TYPE. Default value is FIXED_TYPE. Possible values are FIXED_TYPE and ANY_TYPE.
      mode = null                           # Specifies whether the argument is input or output. Can be set for procedures only. Possible values are IN, OUT, and INOUT.
    }
  ]
}
```
A detailed example with authorized views can be found [here](./examples/basic_view/main.tf).

## Features
This module provisions a dataset and a list of tables with associated JSON schemas and views from queries.

<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| access | An array of objects that define dataset access for one or more entities. | `any` | <pre>[<br>  {<br>    "role": "roles/bigquery.dataOwner",<br>    "special_group": "projectOwners"<br>  }<br>]</pre> | no |
| dataset\_id | Unique ID for the dataset being provisioned. | `string` | n/a | yes |
| dataset\_labels | Key value pairs in a map for dataset labels | `map(string)` | `{}` | no |
| dataset\_name | Friendly name for the dataset being provisioned. | `string` | `null` | no |
| default\_partition\_expiration\_ms | The default partition expiration for all partitioned tables in the dataset, in MS | `number` | `null` | no |
| default\_table\_expiration\_ms | TTL of tables using the dataset in MS | `number` | `null` | no |
| delete\_contents\_on\_destroy | (Optional) If set to true, delete all the tables in the dataset when destroying the resource; otherwise, destroying the resource will fail if tables are present. | `bool` | `null` | no |
| deletion\_protection | Whether or not to allow deletion of tables and external tables defined by this module. Can be overriden by table-level deletion\_protection configuration. | `bool` | `false` | no |
| description | Dataset description. | `string` | `null` | no |
| encryption\_key | Default encryption key to apply to the dataset. Defaults to null (Google-managed). | `string` | `null` | no |
| external\_tables | A list of objects which include table\_id, expiration\_time, external\_data\_configuration, and labels. | <pre>list(object({<br>    table_id              = string,<br>    description           = optional(string),<br>    autodetect            = bool,<br>    compression           = string,<br>    ignore_unknown_values = bool,<br>    max_bad_records       = number,<br>    schema                = string,<br>    source_format         = string,<br>    source_uris           = list(string),<br>    csv_options = object({<br>      quote                 = string,<br>      allow_jagged_rows     = bool,<br>      allow_quoted_newlines = bool,<br>      encoding              = string,<br>      field_delimiter       = string,<br>      skip_leading_rows     = number,<br>    }),<br>    google_sheets_options = object({<br>      range             = string,<br>      skip_leading_rows = number,<br>    }),<br>    hive_partitioning_options = object({<br>      mode              = string,<br>      source_uri_prefix = string,<br>    }),<br>    expiration_time     = optional(string, null),<br>    max_staleness       = optional(string),<br>    deletion_protection = optional(bool),<br>    labels              = optional(map(string), {}),<br>  }))</pre> | `[]` | no |
| location | The location of the dataset. For multi-region, US or EU can be provided. | `string` | `"US"` | no |
| materialized\_views | A list of objects which includes view\_id, view\_query, clustering, time\_partitioning, range\_partitioning, expiration\_time and labels | <pre>list(object({<br>    view_id             = string,<br>    description         = optional(string),<br>    query               = string,<br>    enable_refresh      = bool,<br>    refresh_interval_ms = string,<br>    clustering          = optional(list(string), []),<br>    time_partitioning = optional(object({<br>      expiration_ms            = string,<br>      field                    = string,<br>      type                     = string,<br>      require_partition_filter = bool,<br>    }), null),<br>    range_partitioning = optional(object({<br>      field = string,<br>      range = object({<br>        start    = string,<br>        end      = string,<br>        interval = string,<br>      }),<br>    }), null),<br>    expiration_time = optional(string, null),<br>    max_staleness   = optional(string),<br>    labels          = optional(map(string), {}),<br>  }))</pre> | `[]` | no |
| max\_time\_travel\_hours | Defines the time travel window in hours | `number` | `null` | no |
| project\_id | Project where the dataset and table are created | `string` | n/a | yes |
| resource\_tags | A map of resource tags to add to the dataset | `map(string)` | `{}` | no |
| routines | A list of objects which include routine\_id, routine\_type, routine\_language, definition\_body, return\_type, routine\_description and arguments. | <pre>list(object({<br>    routine_id      = string,<br>    routine_type    = string,<br>    language        = string,<br>    definition_body = string,<br>    return_type     = string,<br>    description     = string,<br>    arguments = optional(list(object({<br>      name          = string,<br>      data_type     = string,<br>      argument_kind = string,<br>      mode          = string,<br>    })), []),<br>  }))</pre> | `[]` | no |
| storage\_billing\_model | Specifies the storage billing model for the dataset. Set this flag value to LOGICAL to use logical bytes for storage billing, or to PHYSICAL to use physical bytes instead. LOGICAL is the default if this flag isn't specified. | `string` | `null` | no |
| tables | A list of objects which include table\_id, table\_name, schema, clustering, time\_partitioning, range\_partitioning, expiration\_time and labels. | <pre>list(object({<br>    table_id                 = string,<br>    description              = optional(string),<br>    table_name               = optional(string),<br>    schema                   = string,<br>    clustering               = optional(list(string), []),<br>    require_partition_filter = optional(bool),<br>    time_partitioning = optional(object({<br>      expiration_ms = string,<br>      field         = string,<br>      type          = string,<br>    }), null),<br>    range_partitioning = optional(object({<br>      field = string,<br>      range = object({<br>        start    = string,<br>        end      = string,<br>        interval = string,<br>      }),<br>    }), null),<br>    expiration_time     = optional(string, null),<br>    deletion_protection = optional(bool),<br>    labels              = optional(map(string), {}),<br>  }))</pre> | `[]` | no |
| views | A list of objects which include view\_id and view query | <pre>list(object({<br>    view_id        = string,<br>    description    = optional(string),<br>    query          = string,<br>    use_legacy_sql = bool,<br>    labels         = optional(map(string), {}),<br>  }))</pre> | `[]` | no |

## Outputs

| Name | Description |
|------|-------------|
| bigquery\_dataset | Bigquery dataset resource. |
| bigquery\_external\_tables | Map of BigQuery external table resources being provisioned. |
| bigquery\_tables | Map of bigquery table resources being provisioned. |
| bigquery\_views | Map of bigquery view resources being provisioned. |
| env\_vars | Exported environment variables |
| external\_table\_ids | Unique IDs for any external tables being provisioned |
| external\_table\_names | Friendly names for any external tables being provisioned |
| project | Project where the dataset and tables are created |
| routine\_ids | Unique IDs for any routine being provisioned |
| table\_fqns | Fully qualified names for the table with format projects/{{project}}/datasets/{{dataset}}/tables/{{name}} |
| table\_ids | Unique id for the table being provisioned |
| table\_names | Friendly name for the table being provisioned |
| view\_ids | Unique id for the view being provisioned |
| view\_names | friendlyname for the view being provisioned |

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->

## Requirements

These sections describe requirements for using this module.

### Software

The following dependencies must be available:

- [Terraform](https://www.terraform.io/downloads.html) >= 0.13.0
- [Terraform Provider for GCP][terraform-provider-gcp] plugin v3

### Service Account

A service account with the following roles must be used to provision
the resources of this module:

- BigQuery Data Owner: `roles/bigquery.dataOwner`

The [Project Factory module][project-factory-module] and the
[IAM module][iam-module] may be used in combination to provision a
service account with the necessary roles applied.

#### Script Helper
A helper script for configuring a Service Account is located at (./helpers/setup-sa.sh).

### APIs

A project with the following APIs enabled must be used to host the
resources of this module:

- BigQuery JSON API: `bigquery-json.googleapis.com`

The [Project Factory module][project-factory-module] can be used to
provision a project with the necessary APIs enabled.

## Contributing

Refer to the [contribution guidelines](./CONTRIBUTING.md) for
information on contributing to this module.

[iam-module]: https://registry.terraform.io/modules/terraform-google-modules/iam/google
[project-factory-module]: https://registry.terraform.io/modules/terraform-google-modules/project-factory/google
[terraform-provider-gcp]: https://www.terraform.io/docs/providers/google/index.html
[terraform]: https://www.terraform.io/downloads.html
