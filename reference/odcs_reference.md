# Open Data Contract Standard (ODCS) v3.1.0 — Reference

> Source: [bitol-io/open-data-contract-standard](https://github.com/bitol-io/open-data-contract-standard)
> This is a local reference extracted from the official ODCS documentation for use by skills and agents.

---

## 1. Fundamentals

Top-level contract metadata.

```yaml
apiVersion: v3.1.0
kind: DataContract

id: 53581432-6c55-4ba2-a65f-72344a91553a
name: seller_payments_v1
version: 1.1.0
status: active        # proposed | draft | active | deprecated | retired
domain: seller
dataProduct: payments
tenant: ClimateQuantumInc

description:
  purpose: Views built on top of the seller tables.
  limitations: Cannot be used in conjunction with days with full moons.
  usage: Twice a day, preferable before meals.

tags: ['finance']
```

| Key | Required | Description |
|---|---|---|
| apiVersion | Yes | Version of the standard. Default `v3.1.0`. |
| kind | Yes | Must be `DataContract`. |
| id | Yes | Unique identifier (UUID recommended). |
| version | Yes | Semantic version of this contract. |
| status | Yes | `proposed`, `draft`, `active`, `deprecated`, `retired`. |
| name | No | Name of the data contract. |
| domain | No | Logical data domain name. |
| dataProduct | No | Data product name. |
| tenant | No | Property the data is associated with. |
| tags | No | List of categorization tags. |
| description.purpose | No | Intended purpose. |
| description.limitations | No | Technical/compliance/legal limitations. |
| description.usage | No | Recommended usage. |

---

## 2. Schema

Schema uses **objects** (tables, documents) and **properties** (columns, fields). Both are called **elements**.

### Complete example

```yaml
schema:
  - id: tbl_obj
    name: tbl
    logicalType: object
    physicalType: table
    physicalName: tbl_1
    description: Provides core payment metrics
    tags: ['finance']
    dataGranularityDescription: Aggregation on columns txn_ref_dt, pmt_txn_id
    properties:
      - id: txn_ref_dt_prop
        name: txn_ref_dt
        businessName: transaction reference date
        logicalType: date
        physicalType: date
        description: Reference date for transaction
        partitioned: true
        partitionKeyPosition: 1
        criticalDataElement: false
        classification: public
        examples:
          - 2022-10-03
      - id: rcvr_id_prop
        name: rcvr_id
        primaryKey: true
        primaryKeyPosition: 1
        businessName: receiver id
        logicalType: string
        physicalType: varchar(18)
        required: false
        description: A description for column rcvr_id.
        classification: restricted
```

### Element keys (objects and properties)

| Key | Required | Description |
|---|---|---|
| id | No | Unique identifier for stable references. |
| name | Yes | Element name. |
| physicalName | No | Physical name in the data source. |
| physicalType | No | Physical type. Objects: `table`, `view`, `topic`, `file`. Properties: `VARCHAR(2)`, `DOUBLE`, etc. |
| description | No | Description. |
| businessName | No | Business name. |
| tags | No | List of tags. |
| quality | No | List of data quality attributes. |

### Property-specific keys

| Key | Required | Description |
|---|---|---|
| primaryKey | No | Boolean, default false. |
| primaryKeyPosition | No | Position in composite PK, starts from 1. Default -1. |
| logicalType | No | One of: `string`, `date`, `timestamp`, `time`, `number`, `integer`, `object`, `array`, `boolean`. |
| required | No | True = NOT NULL. Default false. |
| unique | No | True = unique values. Default false. |
| partitioned | No | True = partition column. |
| partitionKeyPosition | No | Position in partition key. |
| classification | No | e.g. `confidential`, `restricted`, `public`. |
| criticalDataElement | No | Boolean CDE flag. |
| transformSourceObjects | No | List of source objects. |
| transformLogic | No | SQL/logic for transformation. |
| transformDescription | No | Business-terms description of transform. |
| examples | No | List of sample values. |
| items | No | For `logicalType: array` — list of items. |

---

## 3. Data Quality

Quality rules can be defined at the schema (object) level or property level. Four types: `text`, `library`, `sql`, `custom`.

### Library metrics (built-in)

| Metric | Level | Description |
|---|---|---|
| `nullValues` | Property | Counts null values. |
| `missingValues` | Property | Counts values considered missing (empty, N/A, etc.). |
| `invalidValues` | Property | Counts values not matching valid criteria. |
| `duplicateValues` | Property or Schema | Counts duplicates. |
| `rowCount` | Schema | Counts total rows. |

### Operators

| Operator | Math | Example |
|---|---|---|
| `mustBe` | = | `mustBe: 0` |
| `mustNotBe` | != | `mustNotBe: 3.14` |
| `mustBeGreaterThan` | > | `mustBeGreaterThan: 59` |
| `mustBeGreaterOrEqualTo` | >= | `mustBeGreaterOrEqualTo: 60` |
| `mustBeLessThan` | < | `mustBeLessThan: 1000` |
| `mustBeLessOrEqualTo` | <= | `mustBeLessOrEqualTo: 999` |
| `mustBeBetween` | range | `mustBeBetween: [0, 100]` |
| `mustNotBeBetween` | !range | `mustNotBeBetween: [0, 100]` |

### Examples

```yaml
# No nulls in a column
properties:
  - name: customer_id
    quality:
      - metric: nullValues
        mustBe: 0
        description: "No null values allowed."

# Null rate below 1%
properties:
  - name: order_status
    quality:
      - metric: nullValues
        mustBeLessThan: 1
        unit: percent

# Valid values only
properties:
  - name: payment_type
    quality:
      - metric: invalidValues
        arguments:
          validValues: ['credit_card', 'boleto', 'voucher', 'debit_card']
        mustBe: 0

# Row count range at schema level
schema:
  - name: orders
    quality:
      - metric: rowCount
        mustBeBetween: [10000, 200000]

# SQL-based quality check
quality:
  - type: sql
    query: |
      SELECT COUNT(*) FROM {object} WHERE {property} IS NOT NULL
    mustBeLessThan: 3600
    scheduler: cron
    schedule: 0 20 * * *
```

### Quality definition keys

| Key | Required | Description |
|---|---|---|
| quality.type | No | `library` (default), `text`, `sql`, `custom`. |
| quality.metric | No | Required for library: metric name. |
| quality.\<operator\> | No | Comparison operator and value. |
| quality.unit | No | `rows` (default) or `percent`. |
| quality.description | No | Human-readable description. |
| quality.dimension | No | `accuracy`, `completeness`, `conformity`, `consistency`, `coverage`, `timeliness`, `uniqueness`. |
| quality.severity | No | Severity of the rule. |
| quality.businessImpact | No | Consequences of failure. |
| quality.scheduler | No | `cron` or tool name. |
| quality.schedule | No | Cron expression or config. |

---

## 4. Service-Level Agreement (SLA)

SLA properties follow the Data QoS periodic table.

```yaml
slaProperties:
  - property: latency
    value: 4
    unit: d
    element: tab1.txn_ref_dt
  - property: generalAvailability
    value: 2022-05-12T09:30:10-08:00
  - property: endOfSupport
    value: 2032-05-12T09:30:10-08:00
  - property: endOfLife
    value: 2042-05-12T09:30:10-08:00
  - property: retention
    value: 3
    unit: y
    element: tab1.txn_ref_dt
  - property: frequency
    value: 1
    unit: d
    element: tab1.txn_ref_dt
  - property: timeOfAvailability
    value: 09:00-08:00
    element: tab1.txn_ref_dt
    driver: regulatory
```

| Key | Required | Description |
|---|---|---|
| slaProperties | No | List of SLA key/value pairs. |
| slaProperties.property | Yes | SLA property name (see valid values below). |
| slaProperties.value | Yes | Agreement value. |
| slaProperties.valueExt | No | Extended value. |
| slaProperties.unit | No | `d`/`day`/`days`, `y`/`yr`/`years`, etc. (ISO standard). |
| slaProperties.element | No | Element(s) to check. Format: `object.property`. |
| slaProperties.driver | No | Importance: `regulatory`, `analytics`, `operational`. |
| slaProperties.description | No | Human-readable description. |
| slaProperties.scheduler | No | Scheduler name (`cron`, etc.). |
| slaProperties.schedule | No | Scheduler config (e.g. `0 20 * * *`). |

### Valid SLA property values

`availability`, `throughput`, `errorRate`, `generalAvailability`, `endOfSupport`, `endOfLife`, `retention`, `frequency`, `latency` (preferred over freshness), `timeToDetect`, `timeToNotify`, `timeToRepair`.

---

## 5. Infrastructure & Servers

### PostgreSQL server

```yaml
servers:
  - server: my-postgres
    type: postgresql
    host: localhost
    port: 5432
    database: my_database
    schema: public
    environment: prod
```

| Key | Required | Description |
|---|---|---|
| server | Yes | Server identifier. |
| type | Yes | `postgresql` (or `postgres`). |
| host | Yes | Host. |
| port | No | Port (default 5432). |
| database | Yes | Database name. |
| schema | No | Schema name. |
| environment | No | `prod`, `preprod`, `dev`, `uat`. |

---

## 6. Roles

```yaml
roles:
  - role: analyst_read
    access: read
    firstLevelApprovers: Reporting Manager
    secondLevelApprovers: Data Owner
```

| Key | Required | Description |
|---|---|---|
| roles.role | Yes | IAM role name. |
| roles.access | No | Access type (e.g. `read`, `write`). |
| roles.firstLevelApprovers | No | First approver. |
| roles.secondLevelApprovers | No | Second approver. |

---

## 7. Team

```yaml
team:
  name: data-platform-team
  members:
    - username: jdoe
      role: Data Engineer
      dateIn: "2023-01-15"
    - username: asmith
      role: Owner
      description: Data product owner
      dateIn: "2022-06-01"
```

---

## 8. Support & Communication Channels

```yaml
support:
  - channel: '#data-help'
    tool: slack
  - channel: data-announcements
    tool: email
    url: mailto:data-ann@example.com
  - channel: data-issues
    tool: ticket
    url: https://jira.example.com/project/DATA
```

| Key | Required | Description |
|---|---|---|
| support.channel | Yes | Channel name or identifier. |
| support.tool | No | `email`, `slack`, `teams`, `discord`, `ticket`, `googlechat`, `other`. |
| support.url | No | Access URL. |
| support.scope | No | `interactive`, `announcements`, `issues`, `notifications`. |

---

## 9. Pricing

```yaml
price:
  priceAmount: 9.95
  priceCurrency: USD
  priceUnit: megabyte
```

---

## 10. Custom Properties

Available in many sections.

```yaml
customProperties:
  - property: refRulesetName
    value: gcsc.ruleset.name
  - property: someProperty
    value: property.value
    description: Human-readable description
```

---

## 11. References & Relationships (Foreign Keys)

### Property-level (from is implicit)

```yaml
properties:
  - name: customer_id
    relationships:
      - to: customers.id          # shorthand notation
      - to: schema/customers_tbl/properties/cust_id_pk  # fully qualified
```

### Schema-level (from and to required)

```yaml
schema:
  - name: order_items
    relationships:
      - type: foreignKey
        from: order_items.order_id
        to: orders.order_id
```

### Composite keys (arrays must match length)

```yaml
relationships:
  - type: foreignKey
    from:
      - order_items.order_id
      - order_items.product_id
    to:
      - product_inventory.order_id
      - product_inventory.product_id
```

---

## 12. Full Example Contract

```yaml
apiVersion: v3.1.0
kind: DataContract

id: 53581432-6c55-4ba2-a65f-72344a91553a
domain: seller
dataProduct: my quantum
version: 1.1.0
status: active
tenant: ClimateQuantumInc
name: seller_payments_v1

description:
  purpose: Views built on top of the seller tables.
  limitations: Data based on seller perspective, no buyer information.
  usage: Predict sales over time.

tags: ['transactions']

servers:
  - server: my-postgres
    type: postgres
    host: localhost
    port: 5432
    database: pypl-edw
    schema: pp_access_views

schema:
  - id: tbl_obj
    name: tbl
    physicalName: tbl_1
    physicalType: table
    businessName: Core Payment Metrics
    description: Provides core payment metrics
    tags: ['finance', 'payments']
    dataGranularityDescription: Aggregation on columns txn_ref_dt, pmt_txn_id
    properties:
      - id: txn_ref_dt_prop
        name: transaction_reference_date
        physicalName: txn_ref_dt
        businessName: transaction reference date
        logicalType: date
        physicalType: date
        required: false
        description: Reference date for transaction
        partitioned: true
        partitionKeyPosition: 1
        criticalDataElement: false
        classification: public
        examples:
          - "2022-10-03"
          - "2020-01-28"
      - id: rcvr_id_prop
        name: rcvr_id
        primaryKey: true
        primaryKeyPosition: 1
        businessName: receiver id
        logicalType: string
        physicalType: varchar(18)
        required: false
        description: A description for column rcvr_id.
        classification: restricted
        relationships:
          - to: receivers.id
            type: foreignKey
      - id: rcvr_cntry_code_prop
        name: rcvr_cntry_code
        businessName: receiver country code
        logicalType: string
        physicalType: varchar(2)
        required: false
        description: Country code
        classification: public
        quality:
          - metric: nullValues
            mustBe: 0
            description: Column should not contain null values
            dimension: completeness
            severity: error
            businessImpact: operational
            scheduler: cron
            schedule: 0 20 * * *
    quality:
      - metric: rowCount
        mustBeGreaterThan: 1000000
        description: Ensure row count is within expected range
        dimension: completeness
        severity: error

price:
  priceAmount: 9.95
  priceCurrency: USD
  priceUnit: megabyte

team:
  name: my-team
  members:
    - username: daustin
      role: Owner
      dateIn: "2022-10-01"
    - username: mhopper
      role: Data Scientist
      dateIn: "2022-10-01"

roles:
  - role: analyst_read
    access: read
    firstLevelApprovers: Reporting Manager

slaProperties:
  - property: latency
    value: 4
    unit: d
    element: tbl.txn_ref_dt
  - property: generalAvailability
    value: "2022-05-12T09:30:10-08:00"
  - property: retention
    value: 3
    unit: y
  - property: frequency
    value: 1
    unit: d

support:
  - channel: '#product-help'
    tool: slack
  - channel: datacontract-ann
    tool: email
    url: mailto:datacontract-ann@bitol.io

customProperties:
  - property: refRulesetName
    value: gcsc.ruleset.name

contractCreatedTs: "2022-11-15T02:59:43+00:00"
```
