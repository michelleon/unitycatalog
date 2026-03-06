# Unity Catalog External Tables Specification

> **Status**: Proposed
> **Version**: 0.1.0
> **Last Updated**: 2026-03-02

## Table of Contents

- [Introduction](#introduction)
- [Terminology](#terminology)
- [APIs](#apis)
  - [Create a Table](#create-a-table)
  - [Get a Table](#get-a-table)
  - [List Tables](#list-tables)
  - [Delete a Table](#delete-a-table)
  - [Create an External Location](#create-an-external-location)
  - [Get an External Location](#get-an-external-location)
  - [List External Locations](#list-external-locations)
  - [Update an External Location](#update-an-external-location)
  - [Delete an External Location](#delete-an-external-location)

---

## Introduction

This specification defines how Unity Catalog manages external tables and external locations.
An external table is a table whose data resides at a user-specified storage location outside of Unity Catalog's
direct control. Unity Catalog stores only the metadata for the table (name, schema, location, format) and does
not manage the lifecycle of the underlying data files.

This specification defines the APIs that enable the following operations between the server (Unity Catalog) and clients:
- Registering an external table by pointing Unity Catalog to a storage location that the user controls.
- Retrieving and listing external tables.
- Removing an external table from Unity Catalog without affecting its underlying data.
- Managing external locations, which are pre-registered storage paths paired with credentials that govern
  access to groups of external tables.

### What is a Unity Catalog External Table?

A Unity Catalog external table is a table whose storage location and data lifecycle are controlled entirely by the
user. Unity Catalog stores the table's metadata — name, catalog, schema, column definitions, format, storage
location, and properties — but does not allocate storage for the table and does not delete the underlying data
files when the table is dropped.

External tables support multiple data source formats, including Delta Lake, Parquet, CSV, JSON, Avro, ORC,
and plain text. This makes external tables useful for registering existing datasets in cloud storage without
migrating or reformatting the data.

Comparison with Managed Tables:

Feature | Managed Table | External Table
-|-|-
Storage ownership | Unity Catalog | User
Storage location | Assigned by Unity Catalog | User-specified
Auto-cleanup on drop | Yes, UC deletes underlying files | No, files remain on drop
Data source format | DELTA only | DELTA, PARQUET, CSV, JSON, AVRO, ORC, TEXT
Creation flow | Two-step: `createStagingTable` → `createTable` | Single step: `createTable` with `table_type=EXTERNAL`
Credential management | Managed by Unity Catalog | Managed via external locations and storage credentials

### External Locations

An external location is a named, registered storage path (e.g., `s3://my-bucket/data/`) paired with a storage
credential. External locations serve two purposes:
- They enable Unity Catalog to govern which storage paths are accessible and by whom.
- They provide credentials that clients can use to access data in those paths.

When a user creates an external table at a path that falls within a registered external location, Unity Catalog
enforces that the user has either `OWNER` or `CREATE_EXTERNAL_TABLE` privilege on that external location.
This allows administrators to centrally control which users can register tables in a given storage path.

If the table's storage location does not overlap with any registered external location, Unity Catalog does not
enforce external location permissions, and table creation proceeds based solely on schema-level permissions.

## Terminology

- **External table**: A table whose data lifecycle is managed by the user. Unity Catalog stores only the metadata.
  Dropping the table removes the metadata entry but leaves data files in place.
- **External location**: A named, registered storage path in the metastore paired with a storage credential.
  Governs which users can create external tables under a given path.
- **Storage credential**: A credential object (AWS IAM role, Azure service principal, GCS OAuth token, etc.)
  stored in Unity Catalog and referenced by external locations to authenticate against cloud storage.
- **Data source format**: The format of the data files at the table's storage location. For external tables,
  supported formats are `DELTA`, `PARQUET`, `CSV`, `JSON`, `AVRO`, `ORC`, and `TEXT`.
- **Full name**: The three-part identifier for a table: `catalog_name.schema_name.table_name`.

## APIs

A Unity Catalog external table is registered and accessed according to the following specification:
- **Create a table**: An external table is created in a single API call. The client sends a `POST` request
  to the [`createTable`](#create-a-table) API with `table_type` set to `EXTERNAL` and provides a
  `storage_location` pointing to the user-managed data. Unity Catalog registers the metadata and assigns a
  unique `table_id`.
- **Get the table**: After creation, the client retrieves the table by its full three-part name using a `GET`
  request to the [`getTable`](#get-a-table) API.
- **List tables**: The client can list all tables in a given catalog and schema using a `GET` request to the
  [`listTables`](#list-tables) API.
- **Delete the table**: The client drops the table by sending a `DELETE` request to the
  [`deleteTable`](#delete-a-table) API. This removes the metadata entry in Unity Catalog. The data files at
  the `storage_location` are not deleted.
- **External locations**: Before registering external tables, administrators typically set up external locations
  using the [`createExternalLocation`](#create-an-external-location) API. Each external location pairs a storage
  path with a storage credential, establishing the access boundary under which external tables can be created.

The rest of the specification outlines the APIs and detailed requirements. All endpoints are prefixed with
`/api/2.1/unity-catalog`.

### Create a Table

```
POST .../tables
```

Creates a new external table entry in Unity Catalog. The client must provide a user-controlled `storage_location`
that points to the data files for the table. Unity Catalog does not create or modify any files at that location;
it only registers the metadata.

#### Request body

Field Name | Data Type | Description | Optional/Required
-|-|-|-
name | string | Name of the table, relative to parent schema. | required
catalog_name | string | Name of the parent catalog. | required
schema_name | string | Name of the parent schema relative to its parent catalog. | required
table_type | string | The type of the table. Must be set to `"EXTERNAL"`. | required
data_source_format | string | Format of the data files. One of: `DELTA`, `PARQUET`, `CSV`, `JSON`, `AVRO`, `ORC`, `TEXT`. | required
storage_location | string | Storage root URL for the external table data.<br/>Format: URI string (`s3://`, `abfss://`, `gs://`, `file://`, etc.). | required
columns | array of ColumnInfo | Column definitions for the table schema. | optional
&nbsp;&nbsp;columnInfo.name | string | Name of column. | optional
&nbsp;&nbsp;columnInfo.type_text | string | Full data type specification as SQL/catalogString text. | optional
&nbsp;&nbsp;columnInfo.type_json | string | Full data type specification, JSON-serialized. | optional
&nbsp;&nbsp;columnInfo.type_name | string | The type of the column. One of `BOOLEAN`, `BYTE`, `SHORT`, `INT`, `LONG`, `FLOAT`, `DOUBLE`, `DATE`, `TIMESTAMP`, `TIMESTAMP_NTZ`, `CHAR`, `STRING`, `BINARY`, `INTERVAL`, `ARRAY`, `STRUCT`, `MAP`, `DECIMAL`, etc. | optional
&nbsp;&nbsp;columnInfo.type_precision | int32 | Digits of precision; required for `DECIMAL` types. | optional
&nbsp;&nbsp;columnInfo.type_scale | int32 | Digits to the right of decimal; required for `DECIMAL` types. | optional
&nbsp;&nbsp;columnInfo.type_interval_type | string | Format of `INTERVAL` type. | optional
&nbsp;&nbsp;columnInfo.position | int32 | Ordinal position of the column (starting at 0). | optional
&nbsp;&nbsp;columnInfo.comment | string | User-provided description of the column. | optional
&nbsp;&nbsp;columnInfo.nullable | boolean | Whether the column allows null values. Default: `true`. | optional
&nbsp;&nbsp;columnInfo.partition_index | int32 | Partition index for the column. | optional
comment | string | User-provided description of the table. | optional
properties | object | Table properties as key-value pairs. | optional

#### Responses

**200: Request completed successfully.** Echoes all fields from the request and adds the following:

Field Name | Data Type | Description | Optional/Required
-|-|-|-
owner | string | Username of the current table owner. | required
created_at | int64 | Time at which the table was created, in epoch milliseconds. | required
created_by | string | Username of the table creator. | required
updated_at | int64 | Time at which the table was last modified, in epoch milliseconds. | required
updated_by | string | Username of the user who last modified the table. | optional
table_id | string | Unique identifier assigned to the table by Unity Catalog.<br/>Type: UUID string. | required

#### Request-Response Samples

1. Create an external Delta table

    Request:
    ```json
    {
      "name": "sales_data",
      "catalog_name": "main",
      "schema_name": "default",
      "table_type": "EXTERNAL",
      "data_source_format": "DELTA",
      "storage_location": "s3://my-bucket/external/main/default/sales_data/",
      "columns": [
        {
          "name": "order_id",
          "type_text": "bigint",
          "type_json": "{\"name\":\"order_id\",\"type\":\"long\",\"nullable\":false,\"metadata\":{}}",
          "type_name": "LONG",
          "position": 0,
          "nullable": false
        },
        {
          "name": "customer_id",
          "type_text": "bigint",
          "type_json": "{\"name\":\"customer_id\",\"type\":\"long\",\"nullable\":true,\"metadata\":{}}",
          "type_name": "LONG",
          "position": 1,
          "nullable": true
        },
        {
          "name": "order_date",
          "type_text": "date",
          "type_json": "{\"name\":\"order_date\",\"type\":\"date\",\"nullable\":true,\"metadata\":{}}",
          "type_name": "DATE",
          "position": 2,
          "partition_index": 0,
          "nullable": true
        }
      ],
      "comment": "External Delta table containing order data",
      "properties": {
        "owner_team": "data-engineering"
      }
    }
    ```

    Response: 200
    ```json
    {
      "name": "sales_data",
      "catalog_name": "main",
      "schema_name": "default",
      "table_type": "EXTERNAL",
      "data_source_format": "DELTA",
      "storage_location": "s3://my-bucket/external/main/default/sales_data/",
      "columns": [
        {
          "name": "order_id",
          "type_text": "bigint",
          "type_json": "{\"name\":\"order_id\",\"type\":\"long\",\"nullable\":false,\"metadata\":{}}",
          "type_name": "LONG",
          "position": 0,
          "nullable": false
        },
        {
          "name": "customer_id",
          "type_text": "bigint",
          "type_json": "{\"name\":\"customer_id\",\"type\":\"long\",\"nullable\":true,\"metadata\":{}}",
          "type_name": "LONG",
          "position": 1,
          "nullable": true
        },
        {
          "name": "order_date",
          "type_text": "date",
          "type_json": "{\"name\":\"order_date\",\"type\":\"date\",\"nullable\":true,\"metadata\":{}}",
          "type_name": "DATE",
          "position": 2,
          "partition_index": 0,
          "nullable": true
        }
      ],
      "comment": "External Delta table containing order data",
      "properties": {
        "owner_team": "data-engineering"
      },
      "owner": "user@example.com",
      "created_at": 1740873600000,
      "created_by": "user@example.com",
      "updated_at": 1740873600000,
      "updated_by": "user@example.com",
      "table_id": "bcdef123-4567-8901-bcde-f12345678901"
    }
    ```

2. Create an external Parquet table with a decimal column

    Request:
    ```json
    {
      "name": "product_catalog",
      "catalog_name": "main",
      "schema_name": "default",
      "table_type": "EXTERNAL",
      "data_source_format": "PARQUET",
      "storage_location": "s3://my-bucket/external/main/default/product_catalog/",
      "columns": [
        {
          "name": "product_id",
          "type_text": "string",
          "type_json": "{\"name\":\"product_id\",\"type\":\"string\",\"nullable\":false,\"metadata\":{}}",
          "type_name": "STRING",
          "position": 0,
          "nullable": false
        },
        {
          "name": "price",
          "type_text": "decimal(10,2)",
          "type_json": "{\"name\":\"price\",\"type\":\"decimal(10,2)\",\"nullable\":true,\"metadata\":{}}",
          "type_name": "DECIMAL",
          "type_precision": 10,
          "type_scale": 2,
          "position": 1,
          "nullable": true
        }
      ]
    }
    ```

    Response: 200
    ```json
    {
      "name": "product_catalog",
      "catalog_name": "main",
      "schema_name": "default",
      "table_type": "EXTERNAL",
      "data_source_format": "PARQUET",
      "storage_location": "s3://my-bucket/external/main/default/product_catalog/",
      "columns": [
        {
          "name": "product_id",
          "type_text": "string",
          "type_json": "{\"name\":\"product_id\",\"type\":\"string\",\"nullable\":false,\"metadata\":{}}",
          "type_name": "STRING",
          "position": 0,
          "nullable": false
        },
        {
          "name": "price",
          "type_text": "decimal(10,2)",
          "type_json": "{\"name\":\"price\",\"type\":\"decimal(10,2)\",\"nullable\":true,\"metadata\":{}}",
          "type_name": "DECIMAL",
          "type_precision": 10,
          "type_scale": 2,
          "position": 1,
          "nullable": true
        }
      ],
      "owner": "user@example.com",
      "created_at": 1740873700000,
      "created_by": "user@example.com",
      "updated_at": 1740873700000,
      "updated_by": "user@example.com",
      "table_id": "cdef1234-5678-9012-cdef-123456789012"
    }
    ```

#### Errors

Possible errors on this API are listed below:

Error Code | HTTP Status | Description
-|-|-
CATALOG_DOES_NOT_EXIST | 404 | The specified catalog doesn't exist.<br/>The client should verify the catalog name or create the catalog first.
SCHEMA_DOES_NOT_EXIST | 404 | The specified schema doesn't exist.<br/>The client should verify the schema name or create the schema first.
TABLE_ALREADY_EXISTS | 400 | A table with the same name already exists in this schema.<br/>The client should provide a different table name.
PERMISSION_DENIED | 403 | User lacks necessary permissions. Required: USE CATALOG on the catalog, and either OWNER on the schema or (USE SCHEMA + CREATE TABLE) on the schema. For external tables where the storage location falls within a registered external location: OWNER or CREATE EXTERNAL TABLE on that external location.<br/>The client should ask their Unity Catalog admin to grant the required privileges.
INVALID_PARAMETER_VALUE | 400 | One or more request fields are invalid. This includes providing a `table_type` other than `EXTERNAL`, an unsupported `data_source_format`, a missing `storage_location`, or a `storage_location` that conflicts with an existing table or volume.

### Get a Table

```
GET .../tables/{full_name}
```

Gets a table by its full three-part name in the form `catalog_name.schema_name.table_name`.

#### Request parameters

Field Name | Data Type | Description | Optional/Required
-|-|-|-
full_name | string | Full name of the table in the form `catalog_name.schema_name.table_name`. | required (path parameter)

#### Responses

**200: Request completed successfully.**

Field Name | Data Type | Description | Optional/Required
-|-|-|-
name | string | Name of the table, relative to parent schema. | required
catalog_name | string | Name of the parent catalog. | required
schema_name | string | Name of the parent schema relative to its parent catalog. | required
table_type | string | The type of the table. For external tables, value is `"EXTERNAL"`. | required
data_source_format | string | Format of the data files. | required
storage_location | string | Storage root URL for the table. | required
columns | array of ColumnInfo | Column definitions for the table schema. | optional
&nbsp;&nbsp;columnInfo.name | string | Name of column. | optional
&nbsp;&nbsp;columnInfo.type_text | string | Full data type specification as SQL/catalogString text. | optional
&nbsp;&nbsp;columnInfo.type_json | string | Full data type specification, JSON-serialized. | optional
&nbsp;&nbsp;columnInfo.type_name | string | The type of the column. | optional
&nbsp;&nbsp;columnInfo.type_precision | int32 | Digits of precision; required for `DECIMAL` types. | optional
&nbsp;&nbsp;columnInfo.type_scale | int32 | Digits to the right of decimal; required for `DECIMAL` types. | optional
&nbsp;&nbsp;columnInfo.type_interval_type | string | Format of `INTERVAL` type. | optional
&nbsp;&nbsp;columnInfo.position | int32 | Ordinal position of the column (starting at 0). | optional
&nbsp;&nbsp;columnInfo.comment | string | User-provided description of the column. | optional
&nbsp;&nbsp;columnInfo.nullable | boolean | Whether the column allows null values. | optional
&nbsp;&nbsp;columnInfo.partition_index | int32 | Partition index for the column. | optional
comment | string | User-provided description of the table. | optional
properties | object | Table properties as key-value pairs. | optional
owner | string | Username of the current table owner. | required
created_at | int64 | Time at which the table was created, in epoch milliseconds. | required
created_by | string | Username of the table creator. | required
updated_at | int64 | Time at which the table was last modified, in epoch milliseconds. | required
updated_by | string | Username of the user who last modified the table. | optional
table_id | string | Unique identifier for the table.<br/>Type: UUID string. | required

#### Request-Response Samples

Request: `GET .../tables/main.default.sales_data`

Response: 200
```json
{
  "name": "sales_data",
  "catalog_name": "main",
  "schema_name": "default",
  "table_type": "EXTERNAL",
  "data_source_format": "DELTA",
  "storage_location": "s3://my-bucket/external/main/default/sales_data/",
  "columns": [
    {
      "name": "order_id",
      "type_text": "bigint",
      "type_json": "{\"name\":\"order_id\",\"type\":\"long\",\"nullable\":false,\"metadata\":{}}",
      "type_name": "LONG",
      "position": 0,
      "nullable": false
    },
    {
      "name": "customer_id",
      "type_text": "bigint",
      "type_json": "{\"name\":\"customer_id\",\"type\":\"long\",\"nullable\":true,\"metadata\":{}}",
      "type_name": "LONG",
      "position": 1,
      "nullable": true
    },
    {
      "name": "order_date",
      "type_text": "date",
      "type_json": "{\"name\":\"order_date\",\"type\":\"date\",\"nullable\":true,\"metadata\":{}}",
      "type_name": "DATE",
      "position": 2,
      "partition_index": 0,
      "nullable": true
    }
  ],
  "comment": "External Delta table containing order data",
  "properties": {
    "owner_team": "data-engineering"
  },
  "owner": "user@example.com",
  "created_at": 1740873600000,
  "created_by": "user@example.com",
  "updated_at": 1740873600000,
  "updated_by": "user@example.com",
  "table_id": "bcdef123-4567-8901-bcde-f12345678901"
}
```

#### Errors

Possible errors on this API are listed below:

Error Code | HTTP Status | Description
-|-|-
CATALOG_DOES_NOT_EXIST | 404 | The specified catalog doesn't exist.<br/>The client should verify the catalog name or create the catalog first.
SCHEMA_DOES_NOT_EXIST | 404 | The specified schema doesn't exist.<br/>The client should verify the schema name or create the schema first.
TABLE_DOES_NOT_EXIST | 404 | The specified table doesn't exist.<br/>The client should verify the table name or create the table first.
PERMISSION_DENIED | 403 | User lacks necessary permissions. Required: USE CATALOG on the catalog, USE SCHEMA on the schema, and OWNER or SELECT or MODIFY on the table.<br/>The client should ask their Unity Catalog admin to grant the required privileges.

### List Tables

```
GET .../tables
```

Returns all tables within a specified catalog and schema. The response is paginated.

#### Request parameters

Field Name | Data Type | Description | Optional/Required
-|-|-|-
catalog_name | string | Name of the parent catalog for tables of interest. | required (query parameter)
schema_name | string | Name of the parent schema for tables of interest. | required (query parameter)
max_results | int32 | Maximum number of tables to return per page. When set to 0, the server uses a default page size. When set to a value greater than 0, the page size is the minimum of this value and the server-configured maximum. | optional (query parameter)
page_token | string | Opaque pagination token returned from a previous `listTables` response. Pass this value to retrieve the next page of results. | optional (query parameter)
omit_properties | boolean | If `true`, the response omits table properties from each table entry. Default: `false`. | optional (query parameter)
omit_columns | boolean | If `true`, the response omits column definitions from each table entry. Default: `false`. | optional (query parameter)

#### Responses

**200: Request completed successfully.**

Field Name | Data Type | Description | Optional/Required
-|-|-|-
tables | array of TableInfo | Array of table metadata objects. | required
next_page_token | string | Opaque token for retrieving the next page of results. Absent if no further pages remain. | optional

#### Request-Response Samples

Request: `GET .../tables?catalog_name=main&schema_name=default`

Response: 200
```json
{
  "tables": [
    {
      "name": "sales_data",
      "catalog_name": "main",
      "schema_name": "default",
      "table_type": "EXTERNAL",
      "data_source_format": "DELTA",
      "storage_location": "s3://my-bucket/external/main/default/sales_data/",
      "owner": "user@example.com",
      "created_at": 1740873600000,
      "created_by": "user@example.com",
      "updated_at": 1740873600000,
      "table_id": "bcdef123-4567-8901-bcde-f12345678901"
    },
    {
      "name": "product_catalog",
      "catalog_name": "main",
      "schema_name": "default",
      "table_type": "EXTERNAL",
      "data_source_format": "PARQUET",
      "storage_location": "s3://my-bucket/external/main/default/product_catalog/",
      "owner": "user@example.com",
      "created_at": 1740873700000,
      "created_by": "user@example.com",
      "updated_at": 1740873700000,
      "table_id": "cdef1234-5678-9012-cdef-123456789012"
    }
  ]
}
```

#### Errors

Possible errors on this API are listed below:

Error Code | HTTP Status | Description
-|-|-
CATALOG_DOES_NOT_EXIST | 404 | The specified catalog doesn't exist.<br/>The client should verify the catalog name or create the catalog first.
SCHEMA_DOES_NOT_EXIST | 404 | The specified schema doesn't exist.<br/>The client should verify the schema name or create the schema first.
PERMISSION_DENIED | 403 | User lacks necessary permissions. Required: USE CATALOG on the catalog and USE SCHEMA on the schema.<br/>The client should ask their Unity Catalog admin to grant the required privileges.
INVALID_PARAMETER_VALUE | 400 | The `max_results` parameter is less than 0.

### Delete a Table

```
DELETE .../tables/{full_name}
```

Removes the table's metadata entry from Unity Catalog. For external tables, the underlying data files at
`storage_location` are **not** deleted. After this call, the table name is no longer registered and cannot be
queried by name, but the data in cloud storage remains intact.

#### Request parameters

Field Name | Data Type | Description | Optional/Required
-|-|-|-
full_name | string | Full name of the table in the form `catalog_name.schema_name.table_name`. | required (path parameter)

#### Responses

**200: Request completed successfully.** Returns an empty response body.

#### Request-Response Samples

Request: `DELETE .../tables/main.default.sales_data`

Response: 200
```json
{}
```

#### Errors

Possible errors on this API are listed below:

Error Code | HTTP Status | Description
-|-|-
CATALOG_DOES_NOT_EXIST | 404 | The specified catalog doesn't exist.<br/>The client should verify the catalog name.
SCHEMA_DOES_NOT_EXIST | 404 | The specified schema doesn't exist.<br/>The client should verify the schema name.
TABLE_DOES_NOT_EXIST | 404 | The specified table doesn't exist.<br/>The client should verify the table name.
PERMISSION_DENIED | 403 | User lacks the necessary permissions. Required: OWNER on the catalog; or OWNER on the schema plus USE CATALOG on the catalog; or OWNER on the table plus USE SCHEMA on the schema plus USE CATALOG on the catalog.<br/>The client should ask their Unity Catalog admin to grant the required privileges.

### Create an External Location

```
POST .../external-locations
```

Registers a new external location in the metastore. An external location is a named storage path paired with
a storage credential. It establishes the boundary within which external tables may be created under a unified
set of access controls and cloud credentials.

The caller must be a metastore admin to create external locations.

#### Request body

Field Name | Data Type | Description | Optional/Required
-|-|-|-
name | string | Name of the external location. | required
url | string | Path URL of the external location.<br/>Format: URI string (`s3://`, `abfss://`, `gs://`, etc.). | required
credential_name | string | Name of the storage credential to associate with this location. | required
comment | string | User-provided description. | optional

#### Responses

**200: Request completed successfully.** Echoes all fields from the request and adds the following:

Field Name | Data Type | Description | Optional/Required
-|-|-|-
id | string | Unique identifier for the external location.<br/>Type: UUID string. | required
credential_id | string | Unique identifier of the associated storage credential. | required
owner | string | Username of the external location owner. | required
created_at | int64 | Time at which the external location was created, in epoch milliseconds. | required
created_by | string | Username of the creator. | required
updated_at | int64 | Time at which the external location was last modified, in epoch milliseconds. | required
updated_by | string | Username of the user who last modified the external location. | optional

#### Request-Response Samples

Request:
```json
{
  "name": "my_s3_location",
  "url": "s3://my-bucket/external/",
  "credential_name": "my_aws_credential",
  "comment": "External location for the data engineering team"
}
```

Response: 200
```json
{
  "name": "my_s3_location",
  "url": "s3://my-bucket/external/",
  "credential_name": "my_aws_credential",
  "comment": "External location for the data engineering team",
  "id": "def12345-6789-0123-def0-123456789012",
  "credential_id": "ef123456-7890-1234-ef01-234567890123",
  "owner": "admin@example.com",
  "created_at": 1740873600000,
  "created_by": "admin@example.com",
  "updated_at": 1740873600000
}
```

#### Errors

Possible errors on this API are listed below:

Error Code | HTTP Status | Description
-|-|-
ALREADY_EXISTS | 409 | An external location with this name already exists.<br/>The client should provide a different name.
NOT_FOUND | 404 | The specified storage credential does not exist.<br/>The client should verify the credential name or create the credential first.
PERMISSION_DENIED | 403 | User is not a metastore admin and does not have both CREATE EXTERNAL LOCATION on the metastore and OWNER or CREATE EXTERNAL LOCATION on the credential.<br/>The client should contact their Unity Catalog admin.
INVALID_PARAMETER_VALUE | 400 | One or more fields are invalid, such as a malformed URL.

### Get an External Location

```
GET .../external-locations/{name}
```

Returns the metadata for a registered external location. The caller must be a metastore admin, the owner of
the external location, or a user with some privilege on the external location.

#### Request parameters

Field Name | Data Type | Description | Optional/Required
-|-|-|-
name | string | Name of the external location. | required (path parameter)

#### Responses

**200: Request completed successfully.**

Field Name | Data Type | Description | Optional/Required
-|-|-|-
name | string | Name of the external location. | required
id | string | Unique identifier for the external location. | required
url | string | Path URL of the external location. | required
credential_name | string | Name of the associated storage credential. | required
credential_id | string | Unique identifier of the associated storage credential. | required
comment | string | User-provided description. | optional
owner | string | Username of the owner. | required
created_at | int64 | Time at which the location was created, in epoch milliseconds. | required
created_by | string | Username of the creator. | required
updated_at | int64 | Time at which the location was last modified, in epoch milliseconds. | required
updated_by | string | Username of the user who last modified the location. | optional

#### Request-Response Samples

Request: `GET .../external-locations/my_s3_location`

Response: 200
```json
{
  "name": "my_s3_location",
  "id": "def12345-6789-0123-def0-123456789012",
  "url": "s3://my-bucket/external/",
  "credential_name": "my_aws_credential",
  "credential_id": "ef123456-7890-1234-ef01-234567890123",
  "comment": "External location for the data engineering team",
  "owner": "admin@example.com",
  "created_at": 1740873600000,
  "created_by": "admin@example.com",
  "updated_at": 1740873600000
}
```

#### Errors

Possible errors on this API are listed below:

Error Code | HTTP Status | Description
-|-|-
EXTERNAL_LOCATION_DOES_NOT_EXIST | 404 | The specified external location doesn't exist.<br/>The client should verify the name or create the location first.
PERMISSION_DENIED | 403 | User lacks the required privileges on this external location.<br/>The client should contact their Unity Catalog admin.

### List External Locations

```
GET .../external-locations
```

Returns an array of external locations registered in the metastore. The caller must be a metastore admin, the
owner of the external locations, or a user with some privilege on the external locations. Results are paginated.

#### Request parameters

Field Name | Data Type | Description | Optional/Required
-|-|-|-
max_results | int32 | Maximum number of external locations to return. If not set, all external locations are returned. | optional (query parameter)
page_token | string | Opaque pagination token returned from a previous `listExternalLocations` response. | optional (query parameter)

#### Responses

**200: Request completed successfully.**

Field Name | Data Type | Description | Optional/Required
-|-|-|-
external_locations | array of ExternalLocationInfo | Array of external location metadata objects. | required
next_page_token | string | Opaque token for retrieving the next page. Absent if no further pages remain. | optional

#### Request-Response Samples

Request: `GET .../external-locations`

Response: 200
```json
{
  "external_locations": [
    {
      "name": "my_s3_location",
      "id": "def12345-6789-0123-def0-123456789012",
      "url": "s3://my-bucket/external/",
      "credential_name": "my_aws_credential",
      "credential_id": "ef123456-7890-1234-ef01-234567890123",
      "owner": "admin@example.com",
      "created_at": 1740873600000,
      "created_by": "admin@example.com",
      "updated_at": 1740873600000
    },
    {
      "name": "my_gcs_location",
      "id": "ef123456-7890-1234-ef01-234567890123",
      "url": "gs://my-gcs-bucket/data/",
      "credential_name": "my_gcs_credential",
      "credential_id": "f1234567-8901-2345-f012-345678901234",
      "owner": "admin@example.com",
      "created_at": 1740873700000,
      "created_by": "admin@example.com",
      "updated_at": 1740873700000
    }
  ]
}
```

#### Errors

Possible errors on this API are listed below:

Error Code | HTTP Status | Description
-|-|-
PERMISSION_DENIED | 403 | User lacks the required privileges to list external locations.<br/>The client should contact their Unity Catalog admin.
INVALID_PARAMETER_VALUE | 400 | The `max_results` parameter is less than 0.

### Update an External Location

```
PATCH .../external-locations/{name}
```

Updates the metadata for an existing external location. The caller must be the owner of the external location
or a metastore admin. Only the fields included in the request body are updated; omitted fields retain their
current values.

#### Request parameters

Field Name | Data Type | Description | Optional/Required
-|-|-|-
name | string | Name of the external location to update. | required (path parameter)

#### Request body

Field Name | Data Type | Description | Optional/Required
-|-|-|-
name | string | New name for the external location. | optional
url | string | New path URL for the external location. | optional
credential_name | string | Name of the new storage credential to associate. | optional
comment | string | Updated user-provided description. | optional
owner | string | New owner username. | optional

#### Responses

**200: Request completed successfully.** Returns the updated `ExternalLocationInfo` object with all fields.

#### Request-Response Samples

Request: `PATCH .../external-locations/my_s3_location`
```json
{
  "comment": "Updated: external location for data engineering and analytics teams"
}
```

Response: 200
```json
{
  "name": "my_s3_location",
  "id": "def12345-6789-0123-def0-123456789012",
  "url": "s3://my-bucket/external/",
  "credential_name": "my_aws_credential",
  "credential_id": "ef123456-7890-1234-ef01-234567890123",
  "comment": "Updated: external location for data engineering and analytics teams",
  "owner": "admin@example.com",
  "created_at": 1740873600000,
  "created_by": "admin@example.com",
  "updated_at": 1740960000000,
  "updated_by": "admin@example.com"
}
```

#### Errors

Possible errors on this API are listed below:

Error Code | HTTP Status | Description
-|-|-
EXTERNAL_LOCATION_DOES_NOT_EXIST | 404 | The specified external location doesn't exist.<br/>The client should verify the name.
ALREADY_EXISTS | 409 | The new name conflicts with an existing external location.<br/>The client should choose a different name.
PERMISSION_DENIED | 403 | User is not the owner or a metastore admin.<br/>The client should contact their Unity Catalog admin.
INVALID_PARAMETER_VALUE | 400 | One or more fields are invalid.

### Delete an External Location

```
DELETE .../external-locations/{name}
```

Removes an external location from the metastore. The caller must be a metastore admin or the owner of the
external location. Deleting an external location does not delete the underlying cloud storage data or any
external tables registered under that location. However, those tables will no longer have access credentials
provided by Unity Catalog after the location is removed.

By default, deletion fails if there are dependent external tables or mounts. Pass `force=true` to override
this behavior.

#### Request parameters

Field Name | Data Type | Description | Optional/Required
-|-|-|-
name | string | Name of the external location to delete. | required (path parameter)
force | boolean | If `true`, deletes the external location even if dependent external tables or mounts exist. Default: `false`. | optional (query parameter)

#### Responses

**200: Request completed successfully.** Returns an empty response body.

#### Request-Response Samples

Request: `DELETE .../external-locations/my_s3_location`

Response: 200 (empty body)

Request with force: `DELETE .../external-locations/my_s3_location?force=true`

Response: 200 (empty body)

#### Errors

Possible errors on this API are listed below:

Error Code | HTTP Status | Description
-|-|-
EXTERNAL_LOCATION_DOES_NOT_EXIST | 404 | The specified external location doesn't exist.<br/>The client should verify the name.
PERMISSION_DENIED | 403 | User is not the owner or a metastore admin.<br/>The client should contact their Unity Catalog admin.
INVALID_ARGUMENT | 400 | The external location is still referenced by tables, volumes, or registered models. Pass `force=true` to delete anyway.<br/>The client should drop dependent objects first or retry with `force=true`.

## API Reference

Method | HTTP request | Description
-|-|-
[**createTable**](../../api/Apis/TablesApi.md#createtable) | **POST** /tables | Create a Table
[**getTable**](../../api/Apis/TablesApi.md#gettable) | **GET** /tables/{full_name} | Get a Table
[**listTables**](../../api/Apis/TablesApi.md#listtables) | **GET** /tables | List Tables
[**deleteTable**](../../api/Apis/TablesApi.md#deletetable) | **DELETE** /tables/{full_name} | Delete a Table
[**createExternalLocation**](../../api/Apis/ExternalLocationsApi.md#createexternallocation) | **POST** /external-locations | Create an External Location
[**getExternalLocation**](../../api/Apis/ExternalLocationsApi.md#getexternallocation) | **GET** /external-locations/{name} | Get an External Location
[**listExternalLocations**](../../api/Apis/ExternalLocationsApi.md#listexternallocations) | **GET** /external-locations | List External Locations
[**updateExternalLocation**](../../api/Apis/ExternalLocationsApi.md#updateexternallocation) | **PATCH** /external-locations/{name} | Update an External Location
[**deleteExternalLocation**](../../api/Apis/ExternalLocationsApi.md#deleteexternallocation) | **DELETE** /external-locations/{name} | Delete an External Location

For full API documentation index, see [Catalog APIs README](../../api/README.md).

---

[[Back to Main README]](../../README.md)
