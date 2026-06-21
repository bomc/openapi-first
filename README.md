# OpenAPI Spring Boot

## Working on your OpenAPI Definition

### Install

1. Install [Node JS](https://nodejs.org/).
2. Clone this repo and run `npm install` in the repo root.

### Usage

#### `npm start`
Starts the reference docs preview server.

#### `npm run build`
Bundles the definition to the dist folder.

#### `npm test`
Validates the definition.


```yaml
bomc:
  $example: ./for.code
```
---

openapi: 3.1.0

# ─────────────────────────────────────────────────────────────
# Meta-Informationen (#218, #215, #219, #116)
# ─────────────────────────────────────────────────────────────
info:
  title: order-management-api                         # kebab-case, endet auf -api (#218 + C-12 Namenskonvention)
  version: 1.2.0                                      # Semantic Versioning (#116)
  description: |
    REST API for managing customer orders.
    Supports creating, reading, updating and cancelling orders
    including order items and delivery information.
  contact:
    name: Platform Team
    email: platform-team@company.com
    url: https://wiki.company.com/apis/order-management
  x-api-id: d0184f38-b98d-11e7-9c56-68f728c1ba70     # Unveränderliche UUID (#215)
  x-audience: external-partner                         # Zielgruppe (#219)

externalDocs:
  description: Order Management API — Developer Guide
  url: https://developer.company.com/guides/order-management

# ─────────────────────────────────────────────────────────────
# Server (#101 — Spec zusammen mit Service deployed)
# ─────────────────────────────────────────────────────────────
servers:
  - url: https://api.company.com/v1
    description: Production
  - url: https://api.staging.company.com/v1
    description: Staging

# ─────────────────────────────────────────────────────────────
# Globale Sicherheit (#104 — alle Endpunkte absichern)
# Ausnahmen: /health, /ready, /openapi.yaml mit security: []
# ─────────────────────────────────────────────────────────────
security:
  - OAuth2: [read:orders]

# ─────────────────────────────────────────────────────────────
# Endpunkte
# ─────────────────────────────────────────────────────────────
paths:

  # ── Infrastruktur-Endpunkte (#104 Ausnahme) ────────────────

  /health:
    get:
      summary: Liveness check
      description: Returns service health status. No authentication required.
      operationId: getHealth
      tags: [Infrastructure]
      security: []                                     # Explizit kein Auth (#104 Ausnahme)
      responses:
        '200':
          description: Service is healthy
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/HealthStatus'
        '503':
          description: Service is unhealthy
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/HealthStatus'

  /ready:
    get:
      summary: Readiness check
      description: Returns whether the service is ready to accept traffic.
      operationId: getReadiness
      tags: [Infrastructure]
      security: []                                     # Explizit kein Auth (#104 Ausnahme)
      responses:
        '200':
          description: Service is ready
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/HealthStatus'
        '503':
          description: Service is not ready yet
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/HealthStatus'

  # ── Orders Collection (#134 Plural, #129 kebab-case, C-01 Versionierung) ──

  /orders:
    get:
      summary: List orders
      description: |
        Returns a paginated list of orders.
        Results are sorted by `created_at` descending by default.
      operationId: listOrders
      tags: [Orders]
      security:
        - OAuth2: [read:orders]
      parameters:
        - $ref: '#/components/parameters/Traceparent'  # C-03 W3C Trace Context
        - $ref: '#/components/parameters/Cursor'       # Pagination (#159, #160)
        - $ref: '#/components/parameters/Limit'
        - $ref: '#/components/parameters/Sort'
        - $ref: '#/components/parameters/Fields'       # Feldauswahl (#157)
        - name: status
          in: query
          required: false
          style: form                                  # Collection-Format (#154)
          explode: false
          schema:
            type: array
            items:
              type: string
              enum: [OPEN, IN_PROGRESS, COMPLETED, CANCELLED]
          description: |
            Filter by order status. Multiple values comma-separated.
            Example: ?status=OPEN,IN_PROGRESS
          example: "OPEN,IN_PROGRESS"
        - name: customer_id
          in: query
          required: false
          schema:
            type: string
          description: Filter by customer ID.
        - name: created_at_between
          in: query
          required: false
          schema:
            type: string
          description: |
            Filter by creation date range as ISO 8601 interval.
            Example: ?created_at_between=2024-01-01T00:00:00Z/2024-12-31T23:59:59Z
      responses:
        '200':
          description: Paginated list of orders
          headers:
            Cache-Control:
              schema:
                type: string
                example: "no-cache"
            ETag:
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderPage'
              example:
                items:
                  - id: "ord_abc123"
                    customer_id: "cust_789"
                    status: "OPEN"
                    total_amount: 149.95
                    currency_code: "EUR"
                    created_at: "2024-01-15T10:30:00Z"
                    updated_at: "2024-01-15T10:30:00Z"
                cursor:
                  next: "eyJpZCI6Im9yZF9hYmMxMjMifQ"
                  prev: null
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalServerError'

    post:
      summary: Create an order
      operationId: createOrder
      tags: [Orders]
      security:
        - OAuth2: [write:orders]
      parameters:
        - $ref: '#/components/parameters/Traceparent'
        - name: Idempotency-Key                        # Idempotenz (#229, #230)
          in: header
          required: false
          schema:
            type: string
            format: uuid
          description: |
            Optional UUID for idempotent request handling.
            Repeated requests with the same key return the cached response.
            Keys are stored for 24 hours.
          example: "7f7e3c1a-4b8d-4f6e-9a2b-1c3d5e7f9a0b"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderRequest'
      responses:
        '201':
          description: Order created successfully
          headers:
            Location:
              schema:
                type: string
              description: URL of the created order
              example: "/v1/orders/ord_abc123"
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '200':
          description: Existing order returned (idempotent repeat)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '422':
          $ref: '#/components/responses/UnprocessableEntity'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalServerError'

  # ── Order Resource (#143 Sub-Ressourcen via Pfadsegmente) ──

  /orders/{order_id}:
    parameters:
      - name: order_id
        in: path
        required: true
        schema:
          type: string
        description: Unique order identifier.
        example: "ord_abc123"
      - $ref: '#/components/parameters/Traceparent'

    get:
      summary: Get order by ID
      operationId: getOrder
      tags: [Orders]
      security:
        - OAuth2: [read:orders]
      parameters:
        - $ref: '#/components/parameters/Fields'
        - $ref: '#/components/parameters/Embed'        # Sub-Ressourcen einbetten (#158)
        - name: If-None-Match                          # Bedingter GET (#182)
          in: header
          required: false
          schema:
            type: string
          description: ETag for conditional request. Returns 304 if unchanged.
      responses:
        '200':
          description: Order details
          headers:
            Cache-Control:
              schema:
                type: string
                example: "no-cache"
            ETag:
              schema:
                type: string
                example: "\"a1b2c3d4e5f6\""
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '304':
          description: Not Modified — cached response is still valid
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalServerError'

    patch:
      summary: Update order (partial)
      operationId: updateOrder
      tags: [Orders]
      security:
        - OAuth2: [write:orders]
      parameters:
        - name: If-Match                               # Optimistisches Locking (#182)
          in: header
          required: false
          schema:
            type: string
          description: |
            ETag for optimistic locking.
            If provided and the resource has changed, returns 409 Conflict.
          example: "\"a1b2c3d4e5f6\""
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderPatch'
      responses:
        '200':
          description: Order updated
          headers:
            ETag:
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'
        '409':
          $ref: '#/components/responses/Conflict'
        '422':
          $ref: '#/components/responses/UnprocessableEntity'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalServerError'

    delete:
      summary: Cancel order
      operationId: deleteOrder
      tags: [Orders]
      security:
        - OAuth2: [write:orders]
      responses:
        '204':
          description: Order cancelled successfully
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'
        '409':
          $ref: '#/components/responses/Conflict'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalServerError'

  # ── Search (#237, #141 /search Ausnahme) ───────────────────

  /orders/search:
    post:
      summary: Search orders (complex filters)
      description: |
        Search orders with complex filter expressions.
        Use GET /orders with query parameters for simple filters.
        Results are paginated using cursor-based pagination.
      operationId: searchOrders
      tags: [Orders]
      security:
        - OAuth2: [read:orders]
      parameters:
        - $ref: '#/components/parameters/Traceparent'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderSearchRequest'
      responses:
        '200':
          description: Search results
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderPage'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '422':
          $ref: '#/components/responses/UnprocessableEntity'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalServerError'

  # ── Batch (#141 /batch Ausnahme, #152 Code 207) ────────────

  /orders/batch:
    post:
      summary: Create multiple orders (batch)
      operationId: batchCreateOrders
      tags: [Orders]
      security:
        - OAuth2: [write:orders]
      parameters:
        - $ref: '#/components/parameters/Traceparent'
        - name: Idempotency-Key
          in: header
          required: false
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderBatchRequest'
      responses:
        '207':
          description: Multi-status — check individual item status (#152)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderBatchResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalServerError'

  # ── Sub-Ressource: Order Items (#143) ──────────────────────

  /orders/{order_id}/order-items:                      # kebab-case (#129), Plural (#134)
    parameters:
      - name: order_id
        in: path
        required: true
        schema:
          type: string
        example: "ord_abc123"
      - $ref: '#/components/parameters/Traceparent'

    get:
      summary: List order items
      operationId: listOrderItems
      tags: [Order Items]
      security:
        - OAuth2: [read:orders]
      parameters:
        - $ref: '#/components/parameters/Cursor'
        - $ref: '#/components/parameters/Limit'
      responses:
        '200':
          description: Paginated list of order items
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderItemPage'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '404':
          $ref: '#/components/responses/NotFound'
        '429':
          $ref: '#/components/responses/TooManyRequests'
        '500':
          $ref: '#/components/responses/InternalServerError'

# ─────────────────────────────────────────────────────────────
# Wiederverwendbare Komponenten
# ─────────────────────────────────────────────────────────────
components:

  # ── Security Schemes (#104, #105, C-06) ────────────────────
  securitySchemes:
    OAuth2:
      type: oauth2
      flows:
        clientCredentials:
          tokenUrl: https://auth.company.com/oauth/token
          scopes:
            read:orders: Read orders and order items        # C-06: {action}:{resource}
            write:orders: Create, update and cancel orders
            admin:orders: Administrative operations on orders

  # ── Parameter-Definitionen ──────────────────────────────────
  parameters:

    Traceparent:                                           # C-03 W3C Trace Context
      name: traceparent
      in: header
      required: false
      schema:
        type: string
        pattern: '^00-[0-9a-f]{32}-[0-9a-f]{16}-[0-9a-f]{2}$'
        example: "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
      description: |
        W3C Trace Context header for distributed tracing.
        If not provided, a new trace is generated by the gateway.

    Cursor:                                               # Pagination (#159, #160)
      name: cursor
      in: query
      required: false
      schema:
        type: string
      description: |
        Opaque server-internal reference for cursor-based pagination.
        Obtained from a previous response. Do not construct or decode.
        Cursors expire after 24 hours.

    Limit:
      name: limit
      in: query
      required: false
      schema:
        type: integer
        format: int32
        minimum: 1
        maximum: 100
        default: 20
      description: Maximum number of items per page.

    Sort:                                                 # #137 Konventioneller Parameter
      name: sort
      in: query
      required: false
      style: form
      explode: false
      schema:
        type: array
        items:
          type: string
      description: |
        Sort fields as comma-separated list.
        Prefix with + for ascending (default), - for descending.
        Example: ?sort=-created_at,+status

    Fields:                                               # #157 Feldauswahl
      name: fields
      in: query
      required: false
      style: form
      explode: false
      schema:
        type: array
        items:
          type: string
      description: |
        Comma-separated list of fields to include in the response.
        Example: ?fields=id,status,total_amount,created_at

    Embed:                                                # #158 Sub-Ressourcen einbetten
      name: embed
      in: query
      required: false
      style: form
      explode: false
      schema:
        type: array
        items:
          type: string
          enum: [order-items, customer]
      description: |
        Comma-separated list of sub-resources to embed.
        Example: ?embed=order-items,customer

  # ── Schemas ─────────────────────────────────────────────────
  schemas:

    # Order — Hauptressource
    Order:
      type: object                                        # Immer Objekt (#110)
      required: [id, customer_id, status, items, currency_code, created_at, updated_at]
      properties:
        id:
          type: string
          readOnly: true                                  # Nur in Response (#252)
          description: Server-assigned unique identifier.
          example: "ord_abc123"

        external_order_id:
          type: string
          maxLength: 100
          description: |
            Optional client-provided identifier for idempotency.
            If an order with this ID already exists, the existing order is returned.
          example: "ERP-2024-00847"

        customer_id:
          type: string
          description: Reference to the customer resource.  # {entity}_id Muster (#174)
          example: "cust_789"

        warehouse_id:
          type: string
          description: Reference to the fulfilling warehouse.
          example: "wh_001"

        status:
          type: string
          readOnly: true
          x-extensible-enum:                             # Offene Enum-Liste (#112)
            - OPEN
            - IN_PROGRESS
            - COMPLETED
            - CANCELLED
          description: |
            Current order status.
            New values may be added in future versions. Handle unknown values gracefully.
          example: "OPEN"

        total_amount:
          type: number
          format: decimal                                 # Decimal für Geldbeträge (#171)
          readOnly: true
          minimum: 0
          description: Total order amount including taxes.
          example: 149.95

        tax_amount:
          type: number
          format: decimal
          readOnly: true
          minimum: 0
          description: Tax portion of the total amount.
          example: 23.99

        currency_code:
          type: string
          format: iso-4217                               # ISO 4217 (#170)
          pattern: '^[A-Z]{3}$'
          description: Currency code (ISO 4217).
          example: "EUR"

        country_code:
          type: string
          format: iso-3166-alpha-2                       # ISO 3166 (#170)
          pattern: '^[A-Z]{2}$'
          description: Destination country code (ISO 3166-1 alpha-2).
          example: "DE"

        locale:
          type: string
          format: bcp47                                  # BCP 47 (#170)
          description: Customer locale (BCP 47).
          example: "de-AT"

        delivery_date:
          type: string
          format: date                                   # Nur Datum — kein Zeitstempel (#255)
          description: Requested delivery date.
          example: "2024-01-20"

        processing_time:
          type: string
          format: duration                               # ISO 8601 Duration (#127)
          description: Estimated processing time as ISO 8601 duration.
          example: "PT4H"

        is_gift_wrapping_requested:
          type: boolean
          nullable: false                                # Kein null für Boolean (#122)
          description: |
            Whether gift wrapping was requested.
            Omitted if no selection has been made yet (equivalent to not set).

        terms_acceptance:
          type: string
          enum: [ACCEPTED, DECLINED, PENDING]            # Enum statt nullable Boolean (#122)
          description: Status of terms and conditions acceptance.
          example: "ACCEPTED"

        items:
          type: array                                    # Array-Name Plural (#120)
          minItems: 1
          items:
            $ref: '#/components/schemas/OrderItem'

        tags:
          type: array                                    # Leeres Array statt null (#124)
          items:
            type: string
            maxLength: 50
          description: Optional tags. Empty array if none assigned.
          example: []

        translations:
          type: object
          additionalProperties:                          # Map mit additionalProperties (#216)
            type: string
          description: Order descriptions keyed by BCP-47 language code.
          example:
            de: "Winterjacke Bestellung"
            en: "Winter Jacket Order"

        metadata:
          type: object                                   # C-07 Metadata-Feld
          propertyNames:
            pattern: '^[a-z][a-z0-9_]{0,39}$'          # snake_case Keys
          additionalProperties:
            type: string
            maxLength: 500
          maxProperties: 50
          description: |
            Optional key-value pairs for extensibility.
            Keys: snake_case, max 40 characters.
            Do not store sensitive data (passwords, tokens, payment details).

        etag:
          type: string
          readOnly: true
          description: Version hash for optimistic locking. (#174, #182)
          example: "a1b2c3d4e5f6"

        created_at:
          type: string
          format: date-time                              # UTC Zeitstempel (#169)
          readOnly: true
          description: Creation timestamp in UTC (RFC 3339).
          example: "2024-01-15T10:30:00Z"

        updated_at:
          type: string
          format: date-time                              # _at Suffix (#235)
          readOnly: true
          description: Last modification timestamp in UTC (RFC 3339).
          example: "2024-01-15T14:22:00Z"

        cancelled_at:
          type: string
          format: date-time                              # Optional, kein null (#123)
          readOnly: true
          description: Cancellation timestamp in UTC. Omitted if not cancelled.
          example: "2024-01-16T09:00:00Z"

    # Order Request — für POST
    OrderRequest:
      type: object
      required: [customer_id, items, currency_code]
      properties:
        external_order_id:
          type: string
          maxLength: 100
          example: "ERP-2024-00847"

        customer_id:
          type: string
          example: "cust_789"

        warehouse_id:
          type: string
          example: "wh_001"

        currency_code:
          type: string
          format: iso-4217
          pattern: '^[A-Z]{3}$'
          example: "EUR"

        country_code:
          type: string
          format: iso-3166-alpha-2
          pattern: '^[A-Z]{2}$'
          example: "DE"

        delivery_date:
          type: string
          format: date
          example: "2024-01-20"

        is_gift_wrapping_requested:
          type: boolean
          nullable: false

        items:
          type: array
          minItems: 1
          items:
            $ref: '#/components/schemas/OrderItemRequest'

        metadata:
          type: object
          propertyNames:
            pattern: '^[a-z][a-z0-9_]{0,39}$'
          additionalProperties:
            type: string
            maxLength: 500
          maxProperties: 50

    # Order Patch — für PATCH (alle Felder optional)
    OrderPatch:
      type: object
      properties:
        delivery_date:
          type: string
          format: date
          example: "2024-01-25"

        is_gift_wrapping_requested:
          type: boolean
          nullable: false

        warehouse_id:
          type: string
          example: "wh_002"

        metadata:
          type: object
          propertyNames:
            pattern: '^[a-z][a-z0-9_]{0,39}$'
          additionalProperties:
            type: string
            maxLength: 500
          maxProperties: 50

    # Order Item
    OrderItem:
      type: object
      required: [id, product_id, quantity, unit_price, currency_code]
      properties:
        id:
          type: string
          readOnly: true
          example: "item_001"

        product_id:
          type: string                                   # {entity}_id Muster (#174)
          example: "prod_xyz"

        quantity:
          type: integer
          format: int32                                  # Explizites Format (#171)
          minimum: 1
          example: 2

        unit_price:
          type: number
          format: decimal                                # Decimal für Preise (#171)
          minimum: 0
          example: 74.97

        currency_code:
          type: string
          format: iso-4217
          pattern: '^[A-Z]{3}$'
          example: "EUR"

        sku:
          type: string
          maxLength: 50
          description: Stock Keeping Unit identifier.
          example: "SKU-WJ-L-BLK"

        created_at:
          type: string
          format: date-time
          readOnly: true
          example: "2024-01-15T10:30:00Z"

    # Order Item Request
    OrderItemRequest:
      type: object
      required: [product_id, quantity]
      properties:
        product_id:
          type: string
          example: "prod_xyz"

        quantity:
          type: integer
          format: int32
          minimum: 1
          example: 2

        sku:
          type: string
          maxLength: 50
          example: "SKU-WJ-L-BLK"

    # Paginiertes Response-Format (#248, #110)
    OrderPage:
      type: object                                       # Immer Objekt, nie Array (#110)
      required: [items, cursor]
      properties:
        items:                                           # Array-Name Plural (#120)
          type: array
          items:
            $ref: '#/components/schemas/Order'
        cursor:
          $ref: '#/components/schemas/Cursor'

    OrderItemPage:
      type: object
      required: [items, cursor]
      properties:
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
        cursor:
          $ref: '#/components/schemas/Cursor'

    # Cursor-Objekt (#160, #248)
    Cursor:
      type: object
      required: [next, prev]
      properties:
        next:
          type: string
          nullable: true
          description: Cursor for the next page. null if this is the last page.
          example: "eyJpZCI6Im9yZF9hYmMxMjMifQ"
        prev:
          type: string
          nullable: true
          description: Cursor for the previous page. null if this is the first page.

    # Search Request (#237)
    OrderSearchRequest:
      type: object
      properties:
        filter:
          type: object
          properties:
            status:
              type: array
              items:
                type: string
                enum: [OPEN, IN_PROGRESS, COMPLETED, CANCELLED]
            customer_id:
              type: string
            total_amount:
              type: object
              properties:
                gte:
                  type: number
                  format: decimal
                lte:
                  type: number
                  format: decimal
            created_at:
              type: object
              properties:
                gte:
                  type: string
                  format: date-time
                lte:
                  type: string
                  format: date-time
        sort:
          type: array
          style: form
          explode: false
          items:
            type: string
          example: ["-created_at", "+status"]
        limit:
          type: integer
          format: int32
          minimum: 1
          maximum: 100
          default: 20
        cursor:
          type: string
          description: Opaque server-internal reference for pagination.

    # Batch Request (#152, #141)
    OrderBatchRequest:
      type: object
      required: [items]
      properties:
        items:
          type: array
          minItems: 1
          maxItems: 100
          items:
            type: object
            required: [id, order]
            properties:
              id:
                type: string
                description: Client-provided identifier for this batch item.
                example: "req_1"
              order:
                $ref: '#/components/schemas/OrderRequest'

    OrderBatchResponse:
      type: object
      required: [items]
      properties:
        items:
          type: array
          items:
            type: object
            required: [id, status]
            properties:
              id:
                type: string
                description: Matches the request item ID.
                example: "req_1"
              status:
                type: integer
                format: int32
                description: HTTP status code for this item.
                example: 201
              order:
                $ref: '#/components/schemas/Order'
              problem:
                $ref: '#/components/schemas/Problem'

    # Health Status (Infrastruktur)
    HealthStatus:
      type: object
      required: [status]
      properties:
        status:
          type: string
          enum: [UP, DOWN]
          example: "UP"

    # Problem JSON (#176, RFC 7807)
    Problem:
      type: object
      required: [type, title, status]
      properties:
        type:
          type: string
          format: uri
          description: URI identifying the problem type.
          example: "https://api.company.com/errors/validation-error"
        title:
          type: string
          description: Short, human-readable summary of the problem type.
          example: "Validation Error"
        status:
          type: integer
          format: int32
          description: HTTP status code.
          example: 422
        detail:
          type: string
          description: Human-readable explanation specific to this occurrence.
          example: "Field 'quantity' must be greater than 0."
        instance:
          type: string
          format: uri
          description: URI reference identifying the specific occurrence.
          example: "/v1/orders/ord_abc123"

    ProblemWithErrors:                                   # Erweiterung für Validierungsfehler
      allOf:
        - $ref: '#/components/schemas/Problem'
        - type: object
          properties:
            errors:
              type: array
              items:
                type: object
                properties:
                  field:
                    type: string
                    example: "quantity"
                  message:
                    type: string
                    example: "Must be greater than 0."
                  rejected_value:
                    description: The value that was rejected.

    ProblemWithTraceId:                                  # C-05: trace_id bei 5xx
      allOf:
        - $ref: '#/components/schemas/Problem'
        - type: object
          properties:
            trace_id:
              type: string
              description: Trace ID extracted from W3C traceparent header.
              example: "4bf92f3577b34da6a3ce929d0e0e4736"

  # ── Wiederverwendbare Responses (#176 Problem JSON) ─────────
  responses:

    BadRequest:
      description: Syntactically invalid request (#400)
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
          example:
            type: "https://api.company.com/errors/invalid-json"
            title: "Bad Request"
            status: 400
            detail: "Unexpected token at position 32."

    Unauthorized:
      description: Missing or invalid authentication token (#401)
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
          example:
            type: "https://api.company.com/errors/unauthorized"
            title: "Unauthorized"
            status: 401

    Forbidden:
      description: Insufficient permissions (#403)
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
          example:
            type: "https://api.company.com/errors/forbidden"
            title: "Forbidden"
            status: 403
            detail: "The scope 'write:orders' is required for this operation."

    NotFound:
      description: Resource not found (#404)
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
          example:
            type: "https://api.company.com/errors/resource-not-found"
            title: "Not Found"
            status: 404
            detail: "Order 'ord_abc123' does not exist."
            instance: "/v1/orders/ord_abc123"

    Conflict:
      description: Optimistic locking conflict or state conflict (#409)
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
          example:
            type: "https://api.company.com/errors/optimistic-locking-conflict"
            title: "Conflict"
            status: 409
            detail: "The resource was modified since it was last read. Please fetch the current version and retry."

    UnprocessableEntity:
      description: Semantically invalid request — validation failed (#422)
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/ProblemWithErrors'
          example:
            type: "https://api.company.com/errors/validation-error"
            title: "Validation Error"
            status: 422
            detail: "Multiple validation errors occurred."
            errors:
              - field: "quantity"
                message: "Must be greater than 0."
                rejected_value: -1
              - field: "delivery_date"
                message: "Must be a future date."
                rejected_value: "2020-01-01"

    TooManyRequests:
      description: Rate limit exceeded (#153, #429)
      headers:
        Retry-After:
          schema:
            type: integer
          description: Seconds to wait before retrying.
          example: 60
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/Problem'
          example:
            type: "https://api.company.com/errors/rate-limit-exceeded"
            title: "Too Many Requests"
            status: 429
            detail: "Rate limit of 1000 requests per minute exceeded."

    InternalServerError:
      description: Unexpected server error (#500, C-05)
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/ProblemWithTraceId'
          example:
            type: "https://api.company.com/errors/internal-error"
            title: "Internal Server Error"
            status: 500
            trace_id: "4bf92f3577b34da6a3ce929d0e0e4736"

# ─────────────────────────────────────────────────────────────
# Tags (Gruppierung im Developer Portal)
# ─────────────────────────────────────────────────────────────
tags:
  - name: Orders
    description: Order management operations
  - name: Order Items
    description: Order item operations
  - name: Infrastructure
    description: Health and readiness checks — no authentication required


---

# ============================================================
# Gravitee API Score — Custom Ruleset
# Basierend auf: api-styleguide-v2.md (Zalando / Adidas / Stripe)
# Format: Spectral YAML
# Import: APIM Console → API Score → Rulesets & Functions → Import
# ============================================================

rules:

  # ----------------------------------------------------------
  # 2. META-INFORMATIONEN
  # ----------------------------------------------------------

  # [218] info.title muss vorhanden und nicht leer sein
  has-info-title:
    description: "[218] API muss einen titel im info-Block haben."
    message: "info.title fehlt oder ist leer."
    severity: error
    given: "$.info"
    then:
      field: title
      function: truthy

  # [218] info.description muss vorhanden und nicht leer sein
  has-info-description:
    description: "[218] API muss eine Beschreibung im info-Block haben."
    message: "info.description fehlt oder ist leer. Beschreibe Zweck und Anwendungsfälle der API."
    severity: error
    given: "$.info"
    then:
      field: description
      function: truthy

  # [218] info.contact muss vorhanden sein
  has-info-contact:
    description: "[218] API muss Kontaktinformationen im info-Block enthalten."
    message: "info.contact fehlt. Bitte Name, E-Mail und URL des verantwortlichen Teams angeben."
    severity: error
    given: "$.info"
    then:
      field: contact
      function: truthy

  # [218] info.contact.email muss vorhanden sein
  has-info-contact-email:
    description: "[218] Kontakt-E-Mail im info.contact-Block muss angegeben sein."
    message: "info.contact.email fehlt."
    severity: error
    given: "$.info.contact"
    then:
      field: email
      function: truthy

  # [215] x-api-id muss vorhanden sein (UUID)
  has-x-api-id:
    description: "[215] Jede API benötigt eine global eindeutige, unveränderliche UUID als x-api-id."
    message: "info.x-api-id fehlt. Bitte eine UUID eintragen (z.B. d0184f38-b98d-11e7-9c56-68f728c1ba70)."
    severity: error
    given: "$.info"
    then:
      field: x-api-id
      function: truthy

  # [219] x-audience muss vorhanden sein
  has-x-audience:
    description: "[219] API muss ihre Zielgruppe über x-audience deklarieren."
    message: "info.x-audience fehlt. Erlaubte Werte: external-public, external-partner, company-internal, business-unit-internal, component-internal."
    severity: error
    given: "$.info"
    then:
      field: x-audience
      function: truthy

  # [219] x-audience muss einen gültigen Wert haben
  valid-x-audience:
    description: "[219] x-audience muss einen der erlaubten Werte haben."
    message: "info.x-audience hat einen ungültigen Wert. Erlaubt: external-public, external-partner, company-internal, business-unit-internal, component-internal."
    severity: error
    given: "$.info.x-audience"
    then:
      function: enumeration
      functionOptions:
        values:
          - external-public
          - external-partner
          - company-internal
          - business-unit-internal
          - component-internal

  # [116] Semantic Versioning: version muss MAJOR.MINOR.PATCH sein
  semver-version:
    description: "[116] API-Version muss Semantic Versioning (MAJOR.MINOR.PATCH) folgen."
    message: "info.version '{{value}}' entspricht nicht dem Format MAJOR.MINOR.PATCH (z.B. 1.2.3)."
    severity: error
    given: "$.info.version"
    then:
      function: pattern
      functionOptions:
        match: "^\\d+\\.\\d+\\.\\d+$"

  # [102] externalDocs sollte vorhanden sein (Benutzerhandbuch)
  has-external-docs:
    description: "[102] API sollte ein Benutzerhandbuch via externalDocs verlinken."
    message: "externalDocs fehlt. Verlinke ein Benutzerhandbuch mit Zweck, Beispielen und Fehlerfällen."
    severity: warn
    given: "$"
    then:
      field: externalDocs
      function: truthy

  # ----------------------------------------------------------
  # 3. SICHERHEIT
  # ----------------------------------------------------------

  # [104] Globales Security-Schema muss definiert sein
  has-security-schemes:
    description: "[104] API muss ein Security-Schema (OAuth2/JWT) in components.securitySchemes definieren."
    message: "components.securitySchemes fehlt. Alle Endpunkte müssen abgesichert sein."
    severity: error
    given: "$.components"
    then:
      field: securitySchemes
      function: truthy

  # [105] Jede Operation muss ein security-Feld haben
  operations-have-security:
    description: "[104/105] Jede Operation muss ein security-Feld definieren (oder explizit [] für bewusste Ausnahmen wie /health)."
    message: "Operation '{{path}}' hat kein security-Feld. Entweder OAuth2-Scopes oder explizit security: [] für Ausnahmen (Health, OpenAPI-Endpunkt)."
    severity: error
    given: "$.paths[*][get,post,put,patch,delete,head,options]"
    then:
      field: security
      function: defined

  # [C-06] Scope-Namen müssen dem Format read|write|admin:<ressource> folgen
  valid-scope-format:
    description: "[C-06] Scope-Namen müssen dem Format read:<ressource>, write:<ressource> oder admin:<ressource> folgen."
    message: "Scope '{{value}}' entspricht nicht dem Schema {aktion}:{ressource} (erlaubte Aktionen: read, write, admin)."
    severity: error
    given: "$.components.securitySchemes[*].flows[*].scopes"
    then:
      function: schema
      functionOptions:
        schema:
          type: object
          additionalProperties:
            type: string
          patternProperties:
            "^(read|write|admin):[a-z][a-z0-9-]*$":
              type: string

  # ----------------------------------------------------------
  # 5. URLs / PFADE
  # ----------------------------------------------------------

  # [C-01] Alle Pfade müssen mit /v{n}/ beginnen
  path-has-version-prefix:
    description: "[C-01] Jeder API-Pfad muss die Major-Version im Pfad enthalten (/v1/, /v2/, ...)."
    message: "Pfad '{{path}}' beginnt nicht mit einer Versionsnummer (/v1/, /v2/, ...). Ausnahmen: /health, /ready, /live, /startup, /openapi.yaml, /metrics."
    severity: error
    given: "$.paths"
    then:
      function: pattern
      functionOptions:
        match: "^(/v\\d+/|/health|/ready|/live|/startup|/openapi\\.yaml|/openapi\\.json|/docs|/metrics)"

  # [129] Pfadsegmente müssen kebab-case sein
  path-kebab-case:
    description: "[129] Pfadsegmente müssen in kebab-case sein (nur Kleinbuchstaben und Bindestriche)."
    message: "Pfad '{{path}}' enthält Segmente die nicht kebab-case sind. Keine camelCase oder snake_case."
    severity: error
    given: "$.paths"
    then:
      function: pattern
      functionOptions:
        match: "^(/v\\d+)?(/[a-z0-9][a-z0-9-]*|/\\{[a-zA-Z0-9_-]+\\})*(/search|/batch|/export)?(/[a-z0-9][a-z0-9-]*|/\\{[a-zA-Z0-9_-]+\\})*$"

  # [136] Keine Trailing Slashes in Pfaden
  no-trailing-slash:
    description: "[136] Pfade dürfen keinen abschliessenden Slash haben."
    message: "Pfad '{{path}}' endet mit einem Slash. Trailing Slashes sind verboten."
    severity: error
    given: "$.paths"
    then:
      function: pattern
      functionOptions:
        notMatch: "/$"

  # [130] Query-Parameter müssen snake_case sein
  query-param-snake-case:
    description: "[130] Query-Parameter müssen snake_case verwenden (keine camelCase oder kebab-case)."
    message: "Query-Parameter '{{value}}' ist nicht snake_case."
    severity: error
    given: "$.paths[*][*].parameters[?(@.in=='query')].name"
    then:
      function: pattern
      functionOptions:
        match: "^[a-z][a-z0-9_]*$"

  # ----------------------------------------------------------
  # 6. JSON PAYLOAD / SCHEMA
  # ----------------------------------------------------------

  # [118] Property-Namen müssen snake_case sein (kein camelCase)
  property-names-snake-case:
    description: "[118] JSON Property-Namen müssen snake_case verwenden — niemals camelCase."
    message: "Property '{{path}}' ist nicht snake_case. Beispiel: order_id statt orderId."
    severity: error
    given: "$.components.schemas[*].properties"
    then:
      function: schema
      functionOptions:
        schema:
          type: object
          patternProperties:
            "^[a-z][a-z0-9_]*$":
              type: object
          additionalProperties: false

  # [C-02] Kein HATEOAS — keine _links, href, self Properties
  no-hateoas:
    description: "[C-02] Kein HATEOAS erlaubt. Keine _links, href oder self Properties in Schemas."
    message: "Property '{{path}}' deutet auf HATEOAS hin (_links, href, self). HATEOAS ist gemäss Styleguide nicht erlaubt."
    severity: error
    given: "$.components.schemas[*].properties"
    then:
      function: schema
      functionOptions:
        schema:
          type: object
          not:
            anyOf:
              - required: ["_links"]
              - required: ["href"]
              - required: ["self"]

  # [235] Datum/Zeit Properties sollten _at-Suffix haben
  datetime-property-at-suffix:
    description: "[235] Properties vom Typ date-time sollten den Suffix _at haben (z.B. created_at, updated_at)."
    message: "Property '{{path}}' ist vom Typ date-time aber hat keinen _at-Suffix."
    severity: warn
    given: "$.components.schemas[*].properties[*][?(@.format=='date-time')]~"
    then:
      function: pattern
      functionOptions:
        match: "_at$"

  # [171] Integer-Properties müssen ein format angeben
  integer-must-have-format:
    description: "[171] Integer-Properties müssen ein explizites Format angeben (int32, int64, bigint)."
    message: "Integer-Property '{{path}}' hat kein format. Bitte int32, int64 oder bigint angeben."
    severity: error
    given: "$.components.schemas[*].properties[?(@.type=='integer')]"
    then:
      field: format
      function: truthy

  # [171] Number-Properties müssen ein format angeben
  number-must-have-format:
    description: "[171] Number-Properties müssen ein explizites Format angeben (float, double, decimal)."
    message: "Number-Property '{{path}}' hat kein format. Bitte float, double oder decimal angeben."
    severity: error
    given: "$.components.schemas[*].properties[?(@.type=='number')]"
    then:
      field: format
      function: truthy

  # ----------------------------------------------------------
  # 7. HTTP-ANFRAGEN / OPERATIONEN
  # ----------------------------------------------------------

  # [151] Jede Operation muss mindestens einen Response-Code definieren
  operations-have-responses:
    description: "[151] Jede Operation muss Response-Codes definieren."
    message: "Operation '{{path}}' hat keine Responses definiert."
    severity: error
    given: "$.paths[*][get,post,put,patch,delete,head,options]"
    then:
      field: responses
      function: truthy

  # [151] Jede Operation muss einen operationId haben
  operations-have-operation-id:
    description: "Jede Operation sollte eine eindeutige operationId haben (für Client-Generierung und Dokumentation)."
    message: "Operation '{{path}}' hat keine operationId."
    severity: warn
    given: "$.paths[*][get,post,put,patch,delete,head,options]"
    then:
      field: operationId
      function: truthy

  # [151] Jede Operation muss eine summary haben
  operations-have-summary:
    description: "[151] Jede Operation muss eine summary haben."
    message: "Operation '{{path}}' hat keine summary."
    severity: warn
    given: "$.paths[*][get,post,put,patch,delete,head,options]"
    then:
      field: summary
      function: truthy

  # [176] Fehler-Responses müssen application/problem+json verwenden (RFC 7807)
  error-responses-problem-json:
    description: "[176] Fehler-Responses (4xx, 5xx) müssen application/problem+json als Content-Type verwenden (RFC 7807)."
    message: "Fehler-Response '{{path}}' verwendet nicht application/problem+json. Problem JSON nach RFC 7807 ist Pflicht."
    severity: error
    given: "$.paths[*][*].responses[?(@property >= '400')].content"
    then:
      function: schema
      functionOptions:
        schema:
          type: object
          required:
            - "application/problem+json"

  # [153] 429-Response muss definiert sein wenn Rate Limiting aktiv ist
  has-429-response:
    description: "[153] Wenn Rate Limiting aktiv ist, muss eine 429-Response mit Retry-After definiert sein."
    message: "Operation '{{path}}' hat keine 429-Response. Bei Rate Limiting muss 429 Too Many Requests mit Retry-After definiert sein."
    severity: warn
    given: "$.paths[*][get,post,put,patch,delete].responses"
    then:
      function: schema
      functionOptions:
        schema:
          type: object
          required:
            - "429"

  # ----------------------------------------------------------
  # 8. HTTP-STATUSCODES
  # ----------------------------------------------------------

  # [152] POST auf Collections muss 201 zurückgeben
  post-collection-returns-201:
    description: "[152] POST auf Collection-Endpunkte (ohne ID-Segment) muss 201 Created zurückgeben."
    message: "POST '{{path}}' sollte 201 Created zurückgeben wenn eine Ressource erstellt wird."
    severity: warn
    given: "$.paths[?(!@property.match(/\\{[^}]+\\}$/))].post.responses"
    then:
      function: schema
      functionOptions:
        schema:
          type: object
          required:
            - "201"

  # [243] Kein 200 für DELETE
  delete-no-200:
    description: "[243] DELETE sollte 204 No Content zurückgeben, nicht 200 OK."
    message: "DELETE '{{path}}' verwendet 200 als Response. DELETE sollte 204 No Content zurückgeben."
    severity: warn
    given: "$.paths[*].delete.responses"
    then:
      function: schema
      functionOptions:
        schema:
          type: object
          not:
            required:
              - "200"

  # ----------------------------------------------------------
  # 11. PAGINIERUNG
  # ----------------------------------------------------------

  # [159] GET auf Collection-Endpunkte sollten Paginierung unterstützen
  collection-get-has-pagination-params:
    description: "[159] GET auf Collections sollte Paginierung via limit/offset oder cursor unterstützen."
    message: "GET '{{path}}' ist ein Collection-Endpunkt ohne Paginierungs-Parameter (limit, offset, cursor, page_token)."
    severity: warn
    given: "$.paths[?(!@property.match(/\\{[^}]+\\}$/))].get.parameters"
    then:
      function: schema
      functionOptions:
        schema:
          type: array
          contains:
            type: object
            properties:
              name:
                type: string
                enum: [limit, offset, cursor, page_token, after, before]
            required:
              - name

  # ----------------------------------------------------------
  # 13. DEPRECATION
  # ----------------------------------------------------------

  # [187] Deprecated Operations sollten x-sunset oder sunset info haben
  deprecated-has-sunset-info:
    description: "[187] Deprecated Operationen sollten Sunset-Informationen (x-sunset) enthalten."
    message: "Operation '{{path}}' ist als deprecated markiert, hat aber keine x-sunset Information."
    severity: warn
    given: "$.paths[*][?(@.deprecated==true)]"
    then:
      field: x-sunset
      function: truthy

