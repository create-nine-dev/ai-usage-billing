# ai-usage-billing --- SAP CAP Practice Project

![SAP
CAP](https://img.shields.io/badge/SAP-CAP%20Node.js-0FA6A6?style=flat-square)
![Runtime](https://img.shields.io/badge/Runtime-Node.js-339933?style=flat-square&logo=node.js&logoColor=white)
![Database](https://img.shields.io/badge/Database-SQLite%20%7C%20SAP%20HANA-0FA6A6?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-SAP%20BTP%20Cloud%20Foundry-1B74BB?style=flat-square)

A hands-on SAP Cloud Application Programming Model (CAP) practice
project. Build an AI usage billing service that stores raw token-usage
data and calculates account-level token totals and estimated costs
dynamically when the data is requested through OData.

> **Built for learning. Inspired by real-world AI SaaS usage patterns.**

------------------------------------------------------------------------

## 1. Instructions

In this practice exercise, you will build and deploy a small AI Usage
Billing Platform using SAP CAP (Node.js).

### Getting started

Use the repository as your starting point, just as you would with a
provided project template in a practical assessment.

1.  Open **SAP Business Application Studio**.

2.  Create and open a **Full Stack Cloud Application** dev space.

3.  Clone this repository:

    ``` bash
    git clone https://github.com/create-nine-dev/ai-usage-billing.git
    ```

4.  Open the cloned project folder:

    ``` bash
    cd ai-usage-billing
    ```

5.  Install the dependencies with a clean install:

    ``` bash
    npm ci
    ```

6.  Review the project structure and requirements below, then start with
    **Task 1**.

> **Important:** Start from the cloned repository. Do not create a new
> CAP project unless the exercise specifically asks you to. Preserve the
> existing project configuration and follow the requested names, field
> order, and data types carefully.

------------------------------------------------------------------------

## 2. Context

You have been asked to build a backend service for an AI SaaS platform.
Customers use different AI model tiers, and the platform records the
input and output tokens consumed by each request.

The platform needs to calculate the total tokens consumed by each
customer account and estimate the associated cost based on the account's
model tier.

To keep the database focused on raw usage data, **only the original
usage information should be persisted**. The account-level `totalTokens`
and `estimatedCost` values must be calculated dynamically when an
account is read through OData.

### Technical constraints

| Area | Requirement |
| --- | --- |
| Framework | SAP Cloud Application Programming Model (CAP), Node.js |
| Local persistence | SQLite |
| Production persistence | SAP HANA |
| Deployment packaging | Multi-Target Application (MTA) |
| Runtime | SAP BTP, Cloud Foundry |
| Authentication | Disabled for this learning exercise; configure according to the exercise environment |
| Service protocol | OData |

------------------------------------------------------------------------

## 3. Prerequisites

-   Access to SAP Business Application Studio.
-   A **Full Stack Cloud Application** dev space.
-   Access to an SAP BTP subaccount with Cloud Foundry enabled for the
    deployment task.
-   Git and the ability to clone this repository.
-   Node.js dependencies installed using `npm ci`.

------------------------------------------------------------------------

## 4. Task 1 --- Data Modeling and Data Initialization

Define the data model in the `db/` folder. Pay close attention to the
**entity names, field names, field order, and data types**.

### Data model requirements

-   **Namespace:** `ai.billing`
-   **Enum:** Create a type named `ModelTier` with the following values:
    -   `F` --- Fast
    -   `P` --- Premium
    -   `R` --- Reasoning
-   **Entity `AIAccounts`:** Stores customer account information and the
    selected AI model tier. It must contain a composition of many usage
    records.
-   **Entity `UsageRecords`:** Stores the raw usage details for each AI
    request and is associated with one AI account.
-   **Virtual fields:** `AIAccounts` must include `totalTokens` and
    `estimatedCost`. These values are calculated dynamically and must
    not be stored as regular database fields.

### Entity and field specification

The tables below define the required entity structure and field sequence
for this exercise.

#### Entity: `AIAccounts`

| Order | Field name | Data type | Key / modifier | Description |
| --- | --- | --- | --- | --- |
| 1 | `ID` | UUID | Key | Unique account identifier |
| 2 | `customerName` | String | --- | Customer or organization name |
| 3 | `modelTier` | `ModelTier` | --- | Account's AI model tier |
| 4 | `totalTokens` | Integer | Virtual | Calculated sum of input and output tokens across the account's usage records |
| 5 | `estimatedCost` | Decimal(10,2) | Virtual | Calculated estimated usage cost |
| 6 | `usageRecords` | Composition, many `UsageRecords` | Composition | Usage records belonging to this account |

#### Enum: `ModelTier`

| Enum key | Value / label |
| --- | --- |
| `F` | Fast |
| `P` | Premium |
| `R` | Reasoning |

#### Entity: `UsageRecords`

| Order | Field name | Data type | Key / modifier | Description |
| --- | --- | --- | --- | --- |
| 1 | `ID` | UUID | Key | Unique usage-record identifier |
| 2 | `requestDescription` | String | --- | Short description of the AI request |
| 3 | `inputTokens` | Decimal(10,2) | --- | Number of input tokens recorded for the request |
| 4 | `outputTokens` | Decimal(10,2) | --- | Number of output tokens recorded for the request |
| 5 | `parent` | Association to `AIAccounts` | Association | Account to which this usage record belongs |

> **Modeling note:** `usageRecords` is the composition exposed from
> `AIAccounts`; `parent` is the backlink association on `UsageRecords`.
> Use the names exactly as shown in the specification.

### Mock data

Initialize the database with the following records. Use standard CAP CSV
naming conventions and ensure the usage records are linked to the
correct account.

#### Table 1 --- AIAccounts

| ID | customerName | modelTier |
| --- | --- | --- |
| ACC-101 | Nova AI Labs | Premium |
| ACC-202 | CodeForge | Fast |
| ACC-303 | VisionWorks | Reasoning |

#### Table 2 --- UsageRecords for account `ACC-101`

| ID | requestDescription | inputTokens | outputTokens |
| --- | --- | --- | --- |
| U1 | Code generation | 1200 | 800 |
| U2 | Documentation | 900 | 600 |

#### Table 3 --- UsageRecords for account `ACC-202`

| ID | requestDescription | inputTokens | outputTokens |
| --- | --- | --- | --- |
| U3 | Chat request | 500 | 300 |
| U4 | Classification | 800 | 200 |

#### Table 4 --- UsageRecords for account `ACC-303`

| ID | requestDescription | inputTokens | outputTokens |
| --- | --- | --- | --- |
| U5 | Architecture analysis | 3000 | 2000 |
| U6 | Code review | 1800 | 1200 |

------------------------------------------------------------------------

## 5. Task 2 --- Service Exposure and Service Logic

Expose both entities through a CAP service in the `srv/` folder.

### Service requirements

-   **Service name:** `AIUsageService`
-   **Service path:** `/aibilling`
-   Expose `AIAccounts` and `UsageRecords`.
-   `UsageRecords` must be read-only.
-   Implement a JavaScript handler matching the service definition.

### Calculation logic

When `AIAccounts` is read, calculate the following virtual fields:

1.  **`totalTokens`** --- Sum `inputTokens + outputTokens` for every
    usage record belonging to the account.
2.  **`estimatedCost`** --- Calculate the cost using the account's model
    tier and the total token count.

Use these fictional rates per 1,000 tokens:

| Model tier | Rate per 1,000 tokens |
| --- | --- |
| Fast | 2 |
| Premium | 5 |
| Reasoning | 12 |

### Important implementation requirement

The calculation must work even when the client does **not** request
`$expand=usageRecords`.

Your handler must therefore retrieve the related usage records
independently when reading `AIAccounts`, rather than relying only on an
expanded navigation property in the incoming OData request.

### Expected calculation results

| Account ID | Customer | Model tier | Total tokens | Estimated cost |
| --- | --- | --- | --- | --- |
| ACC-101 | Nova AI Labs | Premium | 3,500 | 17.50 |
| ACC-202 | CodeForge | Fast | 1,800 | 3.60 |
| ACC-303 | VisionWorks | Reasoning | 8,000 | 96.00 |

**Calculation formula**

``` text
totalTokens = SUM(inputTokens + outputTokens)

estimatedCost = (totalTokens / 1000) * rateForModelTier
```

------------------------------------------------------------------------

## 6. Task 3 --- Production Readiness and Deployment

Prepare the application for deployment to SAP BTP Cloud Foundry.

### 1. SAP HANA configuration

-   Add the required dependency and configuration to use SAP HANA in
    production.
-   Keep SQLite available for local development.

### 2. MTA packaging

-   Configure the application as a Multi-Target Application.
-   Include a Node.js service module.
-   Configure an SAP HANA HDI container and the required database
    deployer module.
-   Do not add an XSUAA service for this exercise unless your training
    environment explicitly requires a different security setup.

### 3. Deployment configuration

Use the following exercise-specific deployment values:

| Setting | Required value |
| --- | --- |
| Cloud Foundry space | `AI-TRAINING` |
| Application name | `${space}-ai-usage-srv` |
| Route | `${org}-ai-usage.${default-domain}` |
| Buildpack | `nodejs_buildpack` |
| Instances | `1` |

Ensure the application route and application name are configured in the
MTA deployment descriptor.

### 4. Build and deploy

Build the MTA archive using the Cloud MTA Build Tool, then deploy it
using the Cloud Foundry CLI.

Typical commands:

``` bash
mbt build
cf login
cf target
cf deploy mta_archives/<generated-archive>.mtar
```

Before deployment, confirm that the Cloud Foundry target is the intended
subaccount and `AI-TRAINING` space. Use the exact archive name generated
by your build.

------------------------------------------------------------------------

## 7. Verification Checklist

### Data model and initialization

-   [ ] Namespace is `ai.billing`.
-   [ ] `ModelTier` contains `F`, `P`, and `R` with the specified
    values.
-   [ ] `AIAccounts` fields match the names, order, and types in this
    README.
-   [ ] `UsageRecords` fields match the names, order, and types in this
    README.
-   [ ] Composition and backlink association are correctly defined.
-   [ ] All mock data is initialized and linked to the correct account.

### Service and logic

-   [ ] Service is named `AIUsageService`.
-   [ ] Service is available at `/aibilling`.
-   [ ] Both entities are exposed.
-   [ ] `UsageRecords` is read-only.
-   [ ] `totalTokens` is calculated dynamically.
-   [ ] `estimatedCost` is calculated dynamically using the specified
    tier rates.
-   [ ] Account calculations work without `$expand=usageRecords`.
-   [ ] OData results match the expected calculation table.

### Production readiness

-   [ ] SQLite works for local development.
-   [ ] SAP HANA is configured for production.
-   [ ] MTA descriptor includes the application and HDI database
    deployment configuration.
-   [ ] The intended Cloud Foundry space, application name, and route
    are configured.
-   [ ] The MTA archive builds successfully.
-   [ ] Deployment completes and the service is running.

------------------------------------------------------------------------

## 8. Suggested OData Tests

Run the service locally:

``` bash
cds watch
```

Test the service metadata and entities:

``` http
GET http://localhost:4004/aibilling/$metadata
GET http://localhost:4004/aibilling/AIAccounts
GET http://localhost:4004/aibilling/UsageRecords
```

Test account calculations without expanding usage records:

``` http
GET http://localhost:4004/aibilling/AIAccounts
```

Then test the expanded navigation data:

``` http
GET http://localhost:4004/aibilling/AIAccounts?$expand=usageRecords
```

The `totalTokens` and `estimatedCost` values should be correct in the
first request as well as the expanded request.

------------------------------------------------------------------------

## 9. Learning Outcomes

By completing this project, you will practise:

-   Defining CAP entities, enums, UUID keys, compositions, and
    associations.
-   Initializing data using CAP CSV conventions.
-   Exposing entities through an OData service.
-   Implementing `after READ` logic in a Node.js service handler.
-   Calculating virtual fields from related records without depending on
    `$expand`.
-   Configuring SQLite for local development and SAP HANA for
    production.
-   Packaging and deploying a CAP application to SAP BTP Cloud Foundry
    using MTA.

------------------------------------------------------------------------

## About this project

> [!IMPORTANT]
> **This is an independent learning exercise** created to practise SAP CAP
> development concepts through a fictional AI SaaS billing scenario. It is
> **not** an SAP certification task, an official SAP sample, or a reproduction
> of any proprietary assessment material. The scenario, entities, names,
> sample data, and deployment identifiers are independently defined for
> this project.

> **Built for learning. Inspired by real-world AI SaaS usage patterns.**

<p align="center"><em>By Sachin Rudraghate</em></p>
