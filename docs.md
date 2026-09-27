# GROS — Application Architecture

MVP architecture

Next.js + AWS Serverless

GROS, built by Cogniheim, will be a multi-tenant SaaS platform that helps local businesses establish their digital presence, showcase products, attract customers, collect feedback and optionally sell online through GROS Market.

The architecture should support three distinct experiences: the GROS business dashboard, each merchant's public storefront and the GROS Market commerce system.

## 1. Application structure

COGNIHEIM

# GROS

Growth Operating System

GROS Command

Merchant dashboard, business setup, catalog, SEO, feedback and insights

gros.cogniheim.in

GROS Market

Storefronts, checkout, payments, orders and shipping

market.gros.cogniheim.in

Merchant storefronts

Public product and business pages, preferably on each merchant's own domain, with a GROS-hosted URL as an alternative.

Shared AWS backend

Authentication · APIs · Business data · Media · Integrations · Events

## 2. Technology stack

|
Layer

|

Technology

|
| --- | --- |
|

Frontend

|

Next.js App Router, TypeScript

|
|

UI

|

Tailwind CSS v4, shadcn/ui, Lucide

|
|

Hosting

|

Vercel initially, or AWS-compatible Next.js hosting

|
|

Authentication

|

Amazon Cognito

|
|

Backend

|

AWS Lambda with Node.js

|
|

API

|

Amazon API Gateway HTTP API

|
|

Database

|

DynamoDB, on-demand

|
|

Images and documents

|

Amazon S3

|
|

Media delivery

|

CloudFront

|
|

Background processing

|

SQS, Lambda, EventBridge

|
|

Monitoring

|

CloudWatch

|
|

Payments

|

Razorpay Route, subject to account approval

|
|

Shipping

|

Shiprocket API

|
|

AI

|

Provider-independent AI service

|

The priority is to keep idle infrastructure costs low, avoid managing servers and add services only when an actual product workflow requires them.


## 3. Backend architecture

Next.js applications

Business dashboard · Public storefront · Market checkout

Amazon API Gateway

JWT authentication · API routing · Throttling

AWS Lambda — Business services

Businesses & users

Catalog & SEO

Feedback & insights

Orders & payments

Shipping & tracking

Webhooks & integrations

DynamoDB

Application data

S3

Images & files

SQS

Background jobs

Razorpay

Payments & settlements

Shiprocket

Delivery

AI & Meta

Insights & messaging

Use API Gateway HTTP APIs rather than REST APIs for ordinary CRUD endpoints, since HTTP APIs offer a smaller feature set at lower cost.

![](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon API Gateway

+1

Start with a modular monorepo and a small number of Lambda functions grouped by domain. You do not need dozens of microservices for the MVP.

## 4. Multi-tenant database

Every merchant has an isolated business workspace. One user can eventually own multiple businesses, and a business can have multiple staff members.

User (Cognito identity)

BusinessMembership

Owner · Admin · Staff

Business / Tenant

Products & categories

Business SEO

Feedback & reviews

Connected channels

Orders & shipments

Payment accounts

DynamoDB can initially use a small set of tables grouped by access pattern: core business data, commerce data and integration/event records. Every tenant-owned record must include `businessId`, with authorization enforced in Lambda—not merely filtered in the frontend.

For checkout, use conditional writes and idempotency keys to prevent duplicate orders or transfers.

## 5. GROS Market: payments and shipping

This is the most important integration boundary because you want to use your Razorpay account while merchants receive money in their own bank accounts.

Customer places an order

Product + delivery charge

GROS Checkout

Create order and calculate shipping

Your Razorpay account + Route

Capture payment and create merchant transfer

Merchant bank

Linked account settlement

Shiprocket

Shipment, courier and tracking

Razorpay Route supports linked merchant accounts and splitting payments, but merchant settlement follows the approved settlement schedule. Instant payouts must be treated as a separately approved capability, not promised to every merchant.

![](https://www.google.com/s2/favicons?domain=https://razorpay.com\&sz=32)

Razorpay

+2

Shiprocket exposes APIs for order creation, couriers, shipments, tracking and other shipping operations.

![](https://www.google.com/s2/favicons?domain=https://apidocs.shiprocket.in\&sz=32)

Shiprocket API

+1

Recommended financial design: keep Razorpay and Shiprocket charges visible as separate provider costs. GROS earns its subscription, with no per-product listing fee. For the first release, use a merchant's connected Shiprocket account where practical; if GROS centrally pays shipping, implement a separate shipping balance and reconciliation workflow.

Do not call an estimated balance a merchant wallet. Display payment captured, transfer pending and bank settlement completed as distinct statuses.


## 6. Next.js monorepo structure

Use one repository with two Next.js applications and shared packages. This avoids duplicating authentication, UI components and business logic.

```
gros/
├── apps/
│   ├── command/             # gros.cogniheim.in
│   │   ├── app/
│   │   │   ├── (auth)/
│   │   │   ├── dashboard/
│   │   │   │   ├── reach/
│   │   │   │   ├── shelf/
│   │   │   │   ├── signal/
│   │   │   │   ├── connect/
│   │   │   │   └── intelligence/
│   │   │   └── settings/
│   │   └── components/
│   │
│   └── market/              # market.gros.cogniheim.in
│       ├── app/
│       │   ├── stores/[slug]/
│       │   ├── cart/
│       │   ├── checkout/
│       │   └── orders/
│       └── components/
│
├── packages/
│   ├── ui/
│   ├── types/
│   ├── validation/
│   ├── auth/
│   └── integrations/
│       ├── razorpay/
│       ├── shiprocket/
│       └── ai/
│
├── services/
│   ├── api/                 # Lambda handlers
│   ├── workers/             # SQS consumers
│   └── webhooks/
│
├── infrastructure/
│   └── aws-cdk/
│
└── package.json
```

A shared API serves both applications. Use AWS CDK to define the infrastructure in TypeScript so development, staging and production environments are reproducible.

## 7. Essential background workflows

|
Event

|

Background action

|
| --- | --- |
|

Product published

|

Update sitemap and invalidate relevant cache

|
|

Image uploaded

|

Validate and optimize image

|
|

Payment captured

|

Verify payment and process merchant transfer

|
|

Order paid

|

Create shipping order

|
|

Shipment updated

|

Update tracking and notify merchant

|
|

Feedback submitted

|

Update business insights

|
|

Scheduled SEO check

|

Identify technical SEO issues

|

Use SQS for retryable jobs and a dead-letter queue for failed operations. Payment and shipping webhook handlers should verify signatures or other provider authentication, persist the event and return promptly. Workers can then process the event safely.

## 8. Security and cost controls

## Non-negotiable safeguards

* Validate tenant access on every business-data operation.

* Keep Razorpay and Shiprocket credentials on the backend.

* Encrypt sensitive data and use least-privilege AWS IAM roles.

* Never store card numbers, banking passwords or UPI PINs.

* Verify webhooks and make payment processing idempotent.

* Enable DynamoDB point-in-time recovery and configure billing alerts.

* Set CloudWatch log retention and limit unexpected AI usage.

For cost control, begin with Lambda, API Gateway HTTP API, DynamoDB on-demand and S3. Avoid NAT Gateways, always-running containers, Redis and OpenSearch until there is a demonstrated need. Keep Next.js rendering and hosting costs separate from backend costs.

## 9. Development roadmap

## GROS v0.1 — First release

Build first

0 of 17 tasks marked complete

Phase 1

Foundation

Next.js monorepo and Cogniheim design system

AWS CDK infrastructure

Cognito authentication and tenant permissions

Business onboarding and Command dashboard

Phase 2

Core growth system

Shelf product and service catalog

Public business and product pages

Reach SEO metadata and sitemap

Signal customer feedback

Connect enquiry links

Phase 3

Market

Storefront, cart and checkout

Razorpay Route merchant onboarding

Payment webhooks and order management

Shiprocket shipping and tracking

Phase 4

Intelligence and launch

AI-assisted SEO and feedback analysis

Business activity and actionable insights

Monitoring, backups and security testing

Pilot onboarding with real merchants

The first end-to-end milestone: a merchant registers, creates a business, publishes a product on a public page and receives a genuine customer enquiry. Once that works, add payments and shipping behind the optional Market module. This gives you a usable GROS product before taking on the complexity of merchant settlements and delivery operations.
