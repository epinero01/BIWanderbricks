# Wanderbricks — Semantic Contract & Data Model

**Version:** 0.1
**Status:** Working Semantic Contract
**Scope:** Data model, grain, relationships, data quality and KPI readiness

---

## 1. Purpose

This document defines the semantic interpretation of the Wanderbricks dataset based on the actual data available in the Databricks environment.

The objective is not simply to document the physical tables, but to establish which business concepts, relationships and metrics can be reliably derived from the data.

The semantic model follows one fundamental principle:

> **Business meaning must be supported by the data. Where the dataset does not provide enough evidence, the interpretation remains explicitly conditional or unsupported.**

This prevents business rules from being silently invented inside SQL or BI calculations.

---

# 2. Dataset Overview

The Wanderbricks dataset contains **16 tables** covering:

* Property management
* Hosts and employees
* Destinations and countries
* Amenities
* Bookings
* Booking updates
* Payments
* Reviews
* Digital engagement
* Customer support
* Users

The main analytical entity is the **property**, around which operational, transactional and engagement data can be connected.

---

# 3. High-Level Semantic Model

```text
                              COUNTRIES
                           /      |       \
                          /       |        \
                         ▼        ▼         ▼
                  DESTINATIONS  HOSTS    EMPLOYEES
                       │          │
                       │          └────── 1:N
                       │
                       ▼
                   PROPERTIES
                  /    │    \
                 /     │     \
                ▼      ▼      ▼
           AMENITIES IMAGES BOOKINGS
               ▲               │
               │               ├──────── PAYMENTS
               │               ├──────── REVIEWS
               │               └──────── BOOKING_UPDATES
               │
      PROPERTY_AMENITIES

USERS
  │
  ├──────── BOOKINGS
  ├──────── REVIEWS
  ├──────── CLICKSTREAM
  ├──────── PAGE_VIEWS
  └──────── CUSTOMER_SUPPORT_LOGS
```

The model contains several distinct analytical domains:

### Property Management

`properties`, `hosts`, `employees`, `destinations`, `countries`, `amenities`, `property_amenities`, `property_images`

### Booking & Transaction Management

`bookings`, `booking_updates`, `payments`

### Customer

`users`, `reviews`, `customer_support_logs`

### Digital Engagement

`clickstream`, `page_views`

---

# 4. Semantic Contract by Table

| Table                   | Semantic grain                                  | Key                         | Main relationships                      | Status                         |
| ----------------------- | ----------------------------------------------- | --------------------------- | --------------------------------------- | ------------------------------ |
| `countries`             | One country                                     | `country`                   | 1:N destinations, hosts, employees      | Certified                      |
| `destinations`          | One destination                                 | `destination_id`            | N:1 country, 1:N properties             | Certified                      |
| `hosts`                 | One host                                        | `host_id`                   | 1:N properties, 1:N employees           | Certified                      |
| `employees`             | One employee                                    | `employee_id`               | N:1 host                                | Certified                      |
| `properties`            | One property                                    | `property_id`               | N:1 host, N:1 destination               | Certified                      |
| `amenities`             | One amenity                                     | `amenity_id`                | N:N properties                          | Certified                      |
| `property_amenities`    | One property–amenity association                | `(property_id, amenity_id)` | Bridge between properties and amenities | Certified                      |
| `property_images`       | One property image                              | `image_id`*                 | N:1 property                            | Conditional                    |
| `bookings`              | One booking                                     | `booking_id`                | N:1 user, N:1 property                  | Certified                      |
| `booking_updates`       | One booking update                              | `booking_update_id`         | N:1 booking                             | Certified                      |
| `payments`              | One monetary movement associated with a booking | `payment_id` not reliable   | N:1 booking                             | Conditional                    |
| `reviews`               | One review record                               | `review_id` not reliable    | N:1 booking/property/user               | Conditional                    |
| `users`                 | One user                                        | `user_id`                   | 1:N bookings, reviews and engagement    | Pending semantic certification |
| `clickstream`           | One website event                               | No declared PK              | N:1 user/property                       | Certified                      |
| `page_views`            | One property-page interaction                   | `view_id` not reliable      | N:1 user/property                       | Conditional                    |
| `customer_support_logs` | One support ticket                              | `ticket_id`                 | N:1 user                                | Certified                      |

* `property_images` has not been treated as a critical semantic blocker. It is considered a secondary/content entity.

---

# 5. Core Entity: Properties

`properties` is the central entity of the Wanderbricks product model.

A property is associated with:

* One host
* One destination
* Multiple amenities
* Multiple images
* Multiple bookings
* Multiple reviews
* Multiple digital interactions

The observed relationships are:

```text
HOST ─────────────── 1:N ───────────── PROPERTY

DESTINATION ──────── 1:N ───────────── PROPERTY

PROPERTY ─────────── N:N ───────────── AMENITY
             via PROPERTY_AMENITIES

PROPERTY ─────────── 1:N ───────────── BOOKING

PROPERTY ─────────── 1:N ───────────── REVIEW

PROPERTY ─────────── 1:N ───────────── PAGE_VIEW

PROPERTY ─────────── 1:N ───────────── CLICKSTREAM
```

`property_id` is a reliable primary key.

There are **18,163 properties** in the dataset.

---

# 6. Geographic Model

The geographic hierarchy is:

```text
COUNTRY
   │
   └── 1:N
        DESTINATION
             │
             └── 1:N
                  PROPERTY
```

The `countries` table contains:

* 168 countries
* 168 unique country codes
* 8 continents
* No null values in the checked attributes

`country` is the primary key.

`country_code` is a unique alternate key.

All destinations have a valid country reference.

There are **42 destinations**, and all 42 have at least one property.

---

# 7. Host & Employee Model

The observed relationship is:

```text
HOST
  │
  └── 1:N ─── EMPLOYEE

HOST
  │
  └── 0:N ─── PROPERTY
```

There are:

* 19,384 hosts
* 73,006 employees
* 18,163 properties

Every employee has a valid host.

Not every host has a property. The dataset contains hosts without associated properties.

Employee lifecycle data shows a consistent observed pattern:

```text
is_currently_employed = TRUE
    → end_service_date IS NULL

is_currently_employed = FALSE
    → end_service_date IS NOT NULL
```

No employee was found with an end date before the joining date.

---

# 8. Amenities Model

Properties and amenities form a genuine many-to-many relationship.

```text
PROPERTY
    │
    │ N:N
    ▼
PROPERTY_AMENITIES
    │
    ▼
AMENITY
```

The bridge contains:

* 118,108 associations
* 18,163 properties
* 38 amenities

The composite key:

```text
(property_id, amenity_id)
```

is unique.

Every property has between **3 and 10 amenities**, with an average of **6.5**.

The four observed amenity categories are:

* Basic
* Safety
* Luxury
* Outdoor

---

# 9. Booking Model

`bookings` is the primary transactional fact for reservations.

### Grain

> One row represents one booking.

### Key

`booking_id`

The key is unique and reliable.

There are **72,247 bookings**.

Observed booking statuses:

* `pending`
* `confirmed`
* `cancelled`
* `completed`

All observed bookings satisfy:

```text
check_out > check_in
```

Therefore stay duration can be derived as:

```text
DATEDIFF(check_out, check_in)
```

### Booking dates

Three different temporal concepts must be preserved:

* `created_at` → booking creation date
* `check_in` → arrival/stay start
* `check_out` → departure/stay end

These dates must not be treated as interchangeable.

---

# 10. Booking Updates

`booking_updates` represents a stream of updates associated with bookings.

Observed relationship:

```text
BOOKING 1:N BOOKING_UPDATE
```

There are:

* 83,068 updates
* 47,726 bookings with at least one update

Some bookings have multiple updates.

The dataset does not provide enough evidence to define every update as exclusively a status change. Therefore the semantic definition remains:

> **Booking update / historical update associated with a booking.**

---

# 11. Payments

Payments require special semantic treatment.

The physical schema declares `payment_id` as a primary key, but the data does **not** support that constraint.

Observed:

* 49,638 payment rows
* 45,173 distinct `payment_id` values
* 4,107 duplicated payment IDs
* Some IDs occur multiple times and can be associated with unrelated bookings and users

Therefore:

> **`payment_id` must not be treated as a reliable unique transaction identifier.**

### Recommended semantic grain

A payment row should be treated as:

> **A monetary movement associated with a booking.**

Observed statuses:

* `completed`
* `refunded`
* `failed`

Observed amount behaviour:

* `refunded` movements are negative
* `completed` movements are almost always positive
* `failed` movements can be positive or negative
* In two-movement bookings, the second movement is exactly the negative of the first

This is compatible with payment/reversal behaviour, but the model should not claim business semantics beyond what the data demonstrates.

### Important consequence

The following concepts must remain separate:

**Completed payment volume**

```sql
SUM(amount)
WHERE status = 'completed'
```

**Refund volume**

```sql
SUM(ABS(amount))
WHERE status = 'refunded'
```

**Net payment movement**

```sql
SUM(amount)
```

None of these should automatically be labelled **Revenue** without an explicit business definition.

---

# 12. Reviews

`reviews` represents user reviews for properties.

However, `review_id` is not a reliable unique identifier.

Observed:

* 99,793 rows
* only 1,000 distinct `review_id` values

Therefore:

> `COUNT(DISTINCT review_id)` must not be used as the review count.

The table supports relationships with:

* bookings
* properties
* users

All checked foreign-key references are valid.

Ratings observed:

* 1.0–5.0
* increments of 0.1
* 476 null ratings

There are also 476 records with `is_deleted = true`.

Any certified rating KPI must explicitly define how deleted and null-rating records are handled.

---

# 13. Digital Engagement

Two independent engagement sources exist:

### `clickstream`

Represents website interaction events.

Observed event types:

* `view`
* `click`
* `search`
* `filter`

Metadata includes:

* device
* referrer

### `page_views`

Represents interactions with property pages.

The declared `view_id` is not unique:

* 500,000 rows
* only 5,000 distinct IDs

Therefore `view_id` cannot be used as a unique event identifier.

### Critical semantic rule

The following must **not** be assumed:

```text
clickstream.event = 'view'
        =
page_views
```

They are separate sources and their events must not be automatically combined or deduplicated without an explicit business/data rule.

---

# 14. Customer Support

`customer_support_logs` has a clean ticket-level grain.

### Grain

> One row represents one support ticket.

### Key

`ticket_id`

There are 1,900 tickets and all ticket IDs are unique.

Each ticket contains an array of four messages.

The messages contain:

* message text
* sender
* sentiment
* timestamp

The dataset contains raw sentiment values with inconsistencies such as:

* `agreeable`
* `aggreeable`

These values should not be silently normalized without an explicit semantic rule.

The apparent `support_agent_id` relationship to `employees` could not be validated and should therefore not be modelled as a confirmed foreign key.

---

# 15. Data Quality Findings

The most important data-quality findings are concentrated in identifiers.

| Table                | Declared ID      | Observed status  | Semantic consequence                |
| -------------------- | ---------------- | ---------------- | ----------------------------------- |
| `properties`         | `property_id`    | Reliable         | Can be used as PK                   |
| `bookings`           | `booking_id`     | Reliable         | Can be used as PK                   |
| `hosts`              | `host_id`        | Reliable         | Can be used as PK                   |
| `employees`          | `employee_id`    | Reliable         | Can be used as PK                   |
| `destinations`       | `destination_id` | Reliable         | Can be used as PK                   |
| `amenities`          | `amenity_id`     | Reliable         | Can be used as PK                   |
| `property_amenities` | composite        | Reliable         | Can be used as bridge PK            |
| `countries`          | `country`        | Reliable         | Can be used as PK                   |
| `payments`           | `payment_id`     | **Not reliable** | Do not use as unique transaction ID |
| `reviews`            | `review_id`      | **Not reliable** | Do not count distinct IDs           |
| `page_views`         | `view_id`        | **Not reliable** | Do not count distinct IDs as views  |

This distinction is one of the most important outputs of the semantic modelling exercise.

---

# 16. KPI Readiness Framework

Every future KPI will be classified into one of three states.

### 🟢 Certified

The dataset provides the necessary grain, relationships and attributes to calculate the KPI reliably.

### 🟡 Conditional

The KPI is technically calculable, but a business definition or modelling decision is required.

### 🔴 Unsupported

The available data is insufficient to calculate the KPI reliably.

---

# 17. Initial KPI Catalogue

| KPI                        | Status | Main source               | Notes                                                                      |
| -------------------------- | ------ | ------------------------- | -------------------------------------------------------------------------- |
| Total Properties           | 🟢     | `properties`              | `COUNT(DISTINCT property_id)`                                              |
| Total Hosts                | 🟢     | `hosts`                   | `COUNT(DISTINCT host_id)`                                                  |
| Total Bookings             | 🟢     | `bookings`                | `COUNT(DISTINCT booking_id)`                                               |
| Completed Bookings         | 🟢     | `bookings`                | Filter by status                                                           |
| Cancelled Bookings         | 🟢     | `bookings`                | Filter by status                                                           |
| Stay Duration              | 🟢     | `bookings`                | `DATEDIFF(check_out, check_in)`                                            |
| Average Stay Duration      | 🟢     | `bookings`                | Based on stay nights                                                       |
| Properties per Destination | 🟢     | `properties`              | Destination relationship validated                                         |
| Amenities per Property     | 🟢     | `property_amenities`      | N:N relationship validated                                                 |
| Completed Payment Volume   | 🟢     | `payments`                | Use completed movements                                                    |
| Refund Volume              | 🟢     | `payments`                | Use refunded movements                                                     |
| Net Payment Movement       | 🟢     | `payments`                | Sum of monetary movements                                                  |
| Average Rating             | 🟡     | `reviews`                 | Requires deletion/null policy                                              |
| Review Count               | 🟡     | `reviews`                 | Must use row grain, not `review_id`                                        |
| Revenue                    | 🟡     | `payments` / `bookings`   | Requires business definition                                               |
| Cancellation Rate          | 🟡     | `bookings`                | Definition of denominator required                                         |
| Conversion Rate            | 🟡     | engagement + bookings     | Requires attribution/window definition                                     |
| Repeat Customer Rate       | 🟡     | `bookings`                | Requires business definition                                               |
| Occupancy                  | 🔴     | —                         | Dataset does not currently provide a reliable inventory/availability basis |
| Customer Lifetime Value    | 🟡     | users + bookings/payments | Requires financial and customer-lifecycle definition                       |

---

# 18. Semantic Principles

The following rules should be maintained throughout the implementation.

### 1. Never trust a declared primary key without validating the data.

The dataset contains several examples where the physical schema declares a PK that is not unique.

### 2. Never infer business meaning from column names alone.

A field named `payment_id` does not necessarily represent a unique payment entity.

### 3. Preserve physical data and semantic interpretation separately.

Original table/column comments describe the source dataset. The semantic model represents the interpretation supported by our analysis.

### 4. Do not silently convert assumptions into KPI logic.

Business decisions such as the definition of revenue, cancellation rate or conversion must be explicit.

### 5. Preserve grain.

Every KPI must identify the grain of its source data before aggregation.

### 6. Avoid double counting across independent event sources.

`clickstream` and `page_views` should remain separate until a formal reconciliation rule exists.

---

# 19. Current Semantic Model Status

The initial semantic profiling phase is considered **complete**.

The next phase is not additional exploratory profiling.

The next phase is:

> **KPI definition → semantic metric → SQL implementation → validation → certification.**

The objective is to move from a physical dataset to a governed analytical layer where every important metric has:

* a clear business definition
* an explicit grain
* a known source
* documented filters
* documented assumptions
* reproducible SQL
* a confidence/certification status

This becomes the foundation for the Wanderbricks semantic model and subsequent BI layer.
