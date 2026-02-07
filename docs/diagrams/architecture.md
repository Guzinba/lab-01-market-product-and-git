# Architecture Description

## Product Choice

- **Product name**: Wildberries  
- **Website**: [https://wildberries.ru](https://wildberries.ru)  
- **Description**: Wildberries is one of the largest e-commerce marketplaces in Russia and the CIS, connecting millions of customers with thousands of sellers. It offers a vast range of products — from fashion and electronics to groceries — supported by its own logistics network, payment system, and customer service infrastructure.

## Main components

![Wildberries Component Diagram](../../../docs/diagrams/out/wildberries/component-diagram/Component%20Diagram.svg)

[Wildberries Component Diagram Code](../../../docs/diagrams/src/wildberries/component-diagram.puml)

Based on the component diagram, here are five key components and their likely responsibilities:

1. **Storefront Gateway**  
   The public API entry point for customer-facing applications (mobile app, website). Handles authentication, request routing, rate limiting, and protocol translation (HTTPS → gRPC).

2. **Catalog & Search Service**  
   Manages product metadata, categories, pricing, and availability. Powers full-text search, filtering, and real-time inventory status using Elasticsearch and sharded databases.

3. **Order Management Service (OMS)**  
   Orchestrates the entire order lifecycle: creation, payment coordination, inventory reservation, and status tracking. Ensures data consistency across services during checkout.

4. **Warehouse Management System (WMS)**  
   Controls physical warehouse operations: item storage locations, pick lists, barcode scanning, and stock updates. Integrates with robotics and human workflows in fulfillment centers.

5. **Logistics & Routing Service**  
   Plans delivery routes, assigns couriers or pickup points (PVZ), and tracks shipments in real time. Integrates with third-party logistics providers (3PL) and optimizes based on geography and load.

## Data flow

![Wildberries Sequence Diagram](docs/diagrams/out/wildberries/architecture-sequence/Sequence%20Diagram.svg)

[Wildberries Sequence Diagram Code](../../../docs/diagrams/src/wildberries/sequence-diagram.puml)

I selected the **"Checkout (Reservation & Payment)"** group from the sequence diagram:

1. The user clicks **“Pay Now”** in the **WB Mobile App**.
2. The app sends a `createOrder` request to the **Storefront Gateway**.
3. The **Storefront Gateway** forwards it to the **Order Management Service (OMS)**.
4. The **OMS** calls the **Inventory Service** to **reserve stock**:
   - A database transaction decrements available stock
   - A reservation ID is returned with a 15-minute hold
5. The **OMS** saves the order with status `PENDING_PAYMENT`.
6. The **Payment Service** initiates a transaction with the bank/acquirer and returns a 3DS URL.
7. The user is redirected to complete authentication in their banking app.

**Components involved**: WB Mobile App → Storefront Gateway → OMS → Inventory Service → Sharded DB → Payment Service → Bank  
**Data exchanged**: cart ID, user ID, SKU list, reserved stock IDs, payment token, 3DS redirect URL.

## Deployment

![Wildberries Deployment Diagram](../../../docs/diagrams/out/wildberries/deployment-diagram/Deployment%20Diagram.svg)

[Wildberries Deployment Diagram Code](../../../docs/diagrams/src/wildberries/deployment-diagram.puml)

The system is deployed across multiple physical and logical layers:

- **Client devices**: Customer smartphones (iOS/Android), partner apps, warehouse terminals, and web browsers.
- **Edge layer**: Public APIs (Storefront, Partner, Internal Gateways) exposed via TLS 1.3 over HTTPS.
- **Compute clusters**: Microservices (e.g., OMS, WMS, Catalog) run in Kubernetes pods grouped by domain (E-Commerce, Logistics, Support).
- **Data infrastructure**:
  - **In-memory**: Redis cluster for session/cache
  - **Search**: Elasticsearch cluster
  - **Databases**: Sharded PostgreSQL (orders, products), ClickHouse (analytics)
  - **Storage**: S3-compatible object storage for media
- **Event backbone**: Kafka cluster handles asynchronous communication (e.g., order paid → warehouse fulfillment).
- **External integrations**: Payment providers, 3PL systems, and push notification services (APNs/FCM) connect via secure REST/gRPC APIs.

All internal microservices communicate over **gRPC with mTLS** within a private VPC, ensuring security and performance.

## Assumptions

1. Inventory reservations use distributed locking or optimistic concurrency control to prevent overselling during high-load events like flash sales.
2. The Kafka event bus decouples payment confirmation from warehouse fulfillment, allowing asynchronous processing and resilience to downstream failures.

## Open questions

1. How does Wildberries handle split shipments when items in a single order come from multiple warehouses?
2. What mechanism ensures exactly-once processing of critical events (e.g., “OrderPaid”) in the Kafka-based fulfillment pipeline?
