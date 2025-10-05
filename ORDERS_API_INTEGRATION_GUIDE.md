<!-- @format -->

# HalalGoes Orders API - Complete Integration Guide

**Version:** 1.2.25
**Last Updated:** 2025-10-02
**API Base URL:** `http://localhost:3456` (development) | `https://api.halalgoes.com` (production)
**WebSocket URL:** `ws://localhost:9080` (development) | `wss://ws.halalgoes.com` (production)

---

## Table of Contents

1. [Overview](#overview)
2. [Order Lifecycle](#order-lifecycle)
3. [User App Integration](#user-app-integration)
4. [Restaurant App Integration](#restaurant-app-integration)
5. [Rider App Integration](#rider-app-integration)
6. [WebSocket Integration](#websocket-integration)
7. [Order Status Flow](#order-status-flow)
8. [Testing with Postman/Hoppscotch](#testing-with-postmanhoppscotch)
9. [Error Handling](#error-handling)
10. [Common Integration Patterns](#common-integration-patterns)

---

## Overview

The HalalGoes Orders API uses a **saga-based checkout workflow** powered by Temporal.io, ensuring reliable order processing with automatic compensation on failures. The system coordinates between:

- **User**: Places orders, tracks delivery
- **Restaurant**: Receives notifications, accepts/rejects orders, updates preparation status
- **Rider**: Gets delivery requests, updates delivery status and location
- **Backend**: Orchestrates workflows, manages state, broadcasts real-time updates

### Key Technologies

- **REST API**: For synchronous operations (cart, checkout initiation, order queries)
- **WebSocket**: For real-time updates (order status, rider location, notifications)
- **Temporal Workflows**: For orchestration (checkout, order processing, rider assignment)
- **Redis**: For caching and WebSocket channel management

---

## Order Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                         ORDER LIFECYCLE                          │
└─────────────────────────────────────────────────────────────────┘

1. CART MANAGEMENT (User)
   ├── Add items to cart
   ├── Update quantities
   └── Review cart

2. PRICING CALCULATION (User) **[REQUIRED]**
   ├── Call pricing API with cart ID
   ├── Get updated pricing breakdown
   └── Display pricing to user
        ↓
3. CHECKOUT INITIATION (User)
   ├── Select delivery address
   ├── Choose payment method
   ├── Include pricing from step 2
   └── Submit checkout request
        ↓
4. PAYMENT PROCESSING (System)
   ├── Validate payment method
   ├── Process payment
   └── Create payment record

5. ORDER CREATION (System)
   ├── Create order from cart
   ├── Snapshot pricing
   └── Generate order ID

6. RESTAURANT NOTIFICATION (System → Restaurant)
   ├── Send order request via WebSocket
   └── Wait for acceptance (with timeout)

7. RESTAURANT ACCEPTANCE (Restaurant)
   ├── Review order details
   └── Accept or Reject
        ↓ (if accepted)
8. RIDER ASSIGNMENT (System → Riders)
   ├── Find nearby available riders
   ├── Send delivery request to riders
   └── Wait for first acceptance

9. ORDER TRACKING (All Parties)
   ├── Restaurant: CONFIRMED → PREPARING
   ├── Rider: PICKED_UP → ON_THE_WAY
   ├── Real-time location updates
   └── Rider: DELIVERED

10. ORDER COMPLETION
   ├── Mark as delivered
   ├── Clear user cart
   └── Request rating/review
```

---

## User App Integration

### 1. Cart Management

#### Get User Cart

**Endpoint:** `GET /carts/:userId`

```bash
curl http://localhost:3456/carts/user-uuid-here
```

**Response:**

```json
{
	"id": "cart-uuid",
	"user_id": "user-uuid",
	"items": [
		{
			"id": "cart-item-uuid",
			"food_item_id": "food-uuid",
			"quantity": 2,
			"variant_id": "variant-uuid",
			"food_item": {
				"id": "food-uuid",
				"name": "Chicken Biryani",
				"price": "12.99",
				"restaurant_id": "restaurant-uuid"
			}
		}
	],
	"created_at": "2025-10-05T10:00:00Z",
	"updated_at": "2025-10-05T10:05:00Z"
}
```

#### Update Cart

**Endpoint:** `PUT /carts/:userId`

**Request Body:**

```json
{
	"items": [
		{
			"food_item_id": "food-uuid-1",
			"quantity": 2,
			"variant_id": "variant-uuid-1"
		},
		{
			"food_item_id": "food-uuid-2",
			"quantity": 1,
			"variant_id": null
		}
	],
	"restaurant_id": "restaurant-uuid"
}
```

**Response:**

```json
{
	"success": true,
	"cart_id": "cart-uuid",
	"items_count": 2,
	"total_items": 3
}
```

**Important Notes:**

- Setting `quantity: 0` removes the item from cart
- All items must be from the same restaurant
- Cart is automatically cleared after successful checkout
- **You MUST call the pricing API after every cart update** (see next section)

---

### 2. Get Pricing Calculation

**⚠️ CRITICAL: This step is REQUIRED before checkout**

After any cart modification (add, update, remove items), you **MUST** call the pricing API to get the updated pricing breakdown. This pricing object is required for checkout.

> **⚠️ DEPRECATION NOTICE:** This manual pricing call requirement will be **deprecated in v2.0**. In the next major version, the checkout API will automatically calculate pricing server-side, eliminating this step. However, for **v1.x**, this remains **mandatory**.

#### Calculate Pricing

**Endpoint:** `GET /pricing/:cartId`

```bash
curl http://localhost:3456/pricing/cart-uuid-here
```

**Response:**

```json
{
	"item_total": 25.98,
	"delivery_fee": 2.99,
	"platform_fee": 1.5,
	"discount_amount": 0.0,
	"amount_to_pay": 30.47
}
```

**When to Call:**

- ✅ After adding item to cart
- ✅ After updating item quantity
- ✅ After removing item from cart
- ✅ Before displaying checkout page
- ✅ When user changes delivery address (may affect delivery fee)

**Integration Pattern:**

```javascript
async function updateCart(userId, items, restaurantId) {
	// Update cart
	const cartResponse = await fetch(`/carts/${userId}`, {
		method: "PUT",
		headers: { "Content-Type": "application/json" },
		body: JSON.stringify({ items, restaurant_id: restaurantId }),
	})

	const cart = await cartResponse.json()

	// REQUIRED: Get updated pricing
	const pricingResponse = await fetch(`/pricing/${cart.cart_id}`)
	const pricing = await pricingResponse.json()

	// Display pricing to user
	displayPricing(pricing)

	// Store pricing for checkout
	return { cart, pricing }
}
```

**Why This Is Required:**

1. **Dynamic Pricing:** Delivery fees, platform fees, and discounts may change based on:

   - Cart total
   - Delivery distance
   - Time of day
   - Promotions
   - Restaurant settings

2. **Price Validation:** The checkout workflow validates that the pricing you send matches the server-side calculation

3. **User Transparency:** Users must see accurate pricing before confirming checkout

**Error Handling:**

```json
// If cart is empty
{
  "statusCode": 400,
  "message": "Cart is empty",
  "error": "Bad Request"
}

// If cart not found
{
  "statusCode": 404,
  "message": "Cart not found",
  "error": "Not Found"
}
```

---

### 3. Checkout Process

#### Initiate Checkout

**Endpoint:** `PUT /orders/checkout/:cartId`

**⚠️ IMPORTANT:** The `pricing` object in the request body MUST be the exact response from `GET /pricing/:cartId` (called in step 2).

**Metadata Object Structure:**

The `metadata` field accepts optional delivery preferences:

- `delivery_instructions` (array of enums, optional): Array of delivery instruction codes

  - Valid values: `"DO_NOT_CALL"`, `"DO_NOT_RING_BELL"`, `"LEAVE_AT_DOOR"`
  - Can combine multiple instructions: `["LEAVE_AT_DOOR", "DO_NOT_RING_BELL"]`

- `special_instructions` (string, optional): Free-text field for additional instructions
  - Examples: "Ring doorbell twice", "Leave with security guard"

**Request Body:**

```json
{
	"user_id": "user-uuid",
	"delivery_address_id": "address-uuid",
	"payment_method_id": "payment-method-uuid",
	"pricing": {
		"item_total": 25.98,
		"delivery_fee": 2.99,
		"platform_fee": 1.5,
		"discount_amount": 0.0,
		"amount_to_pay": 30.47
	},
	"metadata": {
		"delivery_instructions": ["LEAVE_AT_DOOR", "DO_NOT_RING_BELL"],
		"special_instructions": "Please leave at the front door"
	}
}
```

**Response:**

```json
{
	"success": true,
	"message": "checkout initiated successfully"
}
```

**What Happens Next:**

1. **Payment Processing** (automatic)

   - User receives WebSocket update: `checkout_update` with step `PAYMENT_PROCESSING`

2. **Order Creation** (automatic)

   - User receives WebSocket update: `checkout_update` with step `ORDER_CREATING`
   - Then: `checkout_update` with step `ORDER_CREATED` and `data.order_id`

3. **Restaurant Notification** (automatic)

   - User receives: `checkout_update` with step `WAITING_RESTAURANT_ACCEPTANCE`
   - Restaurant receives: `order_request` WebSocket message

4. **Restaurant Response**

   - If accepted: User receives `checkout_update` with step `RIDER_ASSIGNMENT`
   - If rejected: User receives `checkout_update` with step `FAILED` and refund is processed

5. **Rider Assignment** (automatic)

   - System finds nearby riders and sends delivery requests
   - First rider to accept gets the order

6. **Order Tracking**
   - User receives: `checkout_update` with step `COMPLETED` and tracking workflow ID
   - User should join order channel: `order:{orderId}` for real-time updates

**Complete Checkout Flow:**

```javascript
async function performCheckout(userId, cartId, addressId, paymentMethodId) {
	try {
		// Step 1: Get latest pricing (REQUIRED)
		const pricingResponse = await fetch(`/pricing/${cartId}`)
		const pricing = await pricingResponse.json()

		// Step 2: Show pricing to user for confirmation
		const userConfirmed = await showCheckoutConfirmation(pricing)
		if (!userConfirmed) return

		// Step 3: Initiate checkout with pricing from step 1
		const checkoutResponse = await fetch(`/orders/checkout/${cartId}`, {
			method: "PUT",
			headers: { "Content-Type": "application/json" },
			body: JSON.stringify({
				user_id: userId,
				delivery_address_id: addressId,
				payment_method_id: paymentMethodId,
				pricing: pricing, // Use exact pricing from API
				metadata: {
					delivery_instructions: getDeliveryInstructions(), // Array of enums
					special_instructions: getSpecialInstructions(), // Free text
				},
			}),
		})

		const result = await checkoutResponse.json()

		if (result.success) {
			console.log("Checkout initiated successfully")
			// Listen for checkout updates via WebSocket
		} else {
			showError("Checkout failed")
		}
	} catch (error) {
		showError("Network error during checkout")
	}
}
```

---

### 4. Order Queries

#### Get User Orders

**Endpoint:** `GET /orders/user/:userId`

```bash
curl http://localhost:3456/orders/user/user-uuid-here
```

**Response:**

```json
[
	{
		"id": "order-uuid",
		"customer_id": "user-uuid",
		"restaurant_id": "restaurant-uuid",
		"status": "ON_THE_WAY",
		"total_order_value": "32.59",
		"created_at": "2025-10-05T11:30:00Z",
		"estimated_delivery_time": "2025-10-05T12:15:00Z",
		"restaurant": {
			"id": "restaurant-uuid",
			"name": "Taste of India",
			"phone": "+1234567890"
		},
		"delivery_partner": {
			"id": "rider-uuid",
			"first_name": "John",
			"last_name": "Doe",
			"phone": "+1987654321"
		},
		"order_food_items": [
			{
				"food_item": {
					"name": "Chicken Biryani",
					"price": "12.99"
				},
				"quantity": 2
			}
		]
	}
]
```

#### Get Specific Order

**Endpoint:** `GET /orders/:orderId`

**Response:** Same structure as above, single order object

#### Get Order Tracking

**Endpoint:** `GET /orders/:orderId/tracking`

**Response:**

```json
{
	"order_id": "order-uuid",
	"status": "ON_THE_WAY",
	"estimated_delivery_time": "2025-10-05T12:15:00Z",
	"rider_name": "John Doe",
	"rider_phone": "+1987654321"
}
```

#### Get Order Analytics

**Endpoint:** `GET /orders/analytics/user/:userId`

**Response:**

```json
{
	"total_orders": 25,
	"total_spent": "542.75",
	"avg_order_value": "21.71",
	"by_status": {
		"DELIVERED": 20,
		"ON_THE_WAY": 2,
		"PREPARING": 1,
		"CANCELLED": 2
	}
}
```

---

### 5. WebSocket Connection (User)

#### Connect to WebSocket

```javascript
const ws = new WebSocket("ws://localhost:9080")

ws.onopen = () => {
	console.log("WebSocket connected")

	// Authenticate connection
	ws.send(
		JSON.stringify({
			event: "connect_user",
			data: {
				userId: "user-uuid",
				userType: "user",
			},
		})
	)
}
```

#### Listen for Connection Confirmation

```javascript
ws.onmessage = event => {
	const message = JSON.parse(event.data)

	if (message.event === "connection_confirmed") {
		console.log("Connected:", message.data)
		// Output: { userId, userType, queuedNotificationsDelivered: 0 }

		// Now join order channel
		joinOrderChannel("order-uuid-here")
	}
}
```

#### Join Order Channel

```javascript
function joinOrderChannel(orderId) {
	ws.send(
		JSON.stringify({
			event: "join_channel",
			data: {
				channelName: `order:${orderId}`,
				userId: "user-uuid",
				userType: "user",
			},
		})
	)
}
```

#### Listen for Order Updates

```javascript
ws.onmessage = event => {
	const message = JSON.parse(event.data)

	switch (message.event) {
		case "channel_joined":
			console.log("Joined channel:", message.data.channelName)
			break

		case "order_update":
			handleOrderUpdate(message.data)
			break

		case "notification":
			handleNotification(message.data)
			break
	}
}

function handleOrderUpdate(data) {
	console.log("Order update received:", data)
	/*
  {
    type: 'ORDER_STATUS_UPDATE' | 'RIDER_LOCATION_UPDATE' | 'TRACKING_STARTED',
    orderId: 'order-uuid',
    status: 'PREPARING',
    source: 'restaurant' | 'rider',
    message: 'Restaurant is preparing your order',
    timestamp: 1696512000000,
    location: { latitude: 40.7128, longitude: -74.0060 } // if type is RIDER_LOCATION_UPDATE
  }
  */

	// Update UI based on type
	if (data.type === "ORDER_STATUS_UPDATE") {
		updateOrderStatus(data.status, data.message)
	} else if (data.type === "RIDER_LOCATION_UPDATE") {
		updateRiderLocationOnMap(data.location)
	}
}
```

#### Checkout Updates

```javascript
ws.onmessage = event => {
	const message = JSON.parse(event.data)

	if (
		message.event === "notification" &&
		message.data.type === "checkout_update"
	) {
		const { step, message: msg, data, error } = message.data

		console.log(`Checkout Step: ${step} - ${msg}`)

		// Checkout steps:
		// - INITIATED
		// - PAYMENT_PROCESSING
		// - ORDER_CREATING
		// - ORDER_CREATED (data.order_id available)
		// - RESTAURANT_NOTIFYING
		// - WAITING_RESTAURANT_ACCEPTANCE
		// - RIDER_ASSIGNMENT
		// - ORDER_CHANNEL_CREATED (data.tracking_workflow_id available)
		// - CART_CLEARED
		// - COMPLETED
		// - FAILED (error message available)

		if (step === "ORDER_CREATED") {
			// Save order ID
			const orderId = data.order_id
			// Prepare to join order channel
		} else if (step === "COMPLETED") {
			// Checkout successful, join order channel
			joinOrderChannel(data.order_id)
		} else if (step === "FAILED") {
			// Show error to user
			showError(error)
		}
	}
}
```

---

## Restaurant App Integration

### 1. Restaurant Order Management

#### Get Restaurant Orders

**Endpoint:** `GET /restaurants/:restaurantId/orders`

**Query Parameters:**

- `status` (optional): Filter by order status (e.g., `PENDING`, `CONFIRMED`, `PREPARING`)

```bash
curl "http://localhost:3456/restaurants/restaurant-uuid/orders?status=CONFIRMED"
```

**Response:**

```json
[
	{
		"id": "order-uuid",
		"customer_id": "user-uuid",
		"restaurant_id": "restaurant-uuid",
		"status": "CONFIRMED",
		"total_order_value": "32.59",
		"created_at": "2025-10-05T11:30:00Z",
		"customer": {
			"first_name": "Jane",
			"last_name": "Smith",
			"phone": "+1122334455"
		},
		"delivery_to_address": {
			"street": "123 Main St",
			"building": "Apt 4B",
			"city": "New York",
			"state": "NY",
			"zip_code": "10001"
		},
		"order_food_items": [
			{
				"food_item": {
					"name": "Chicken Biryani",
					"price": "12.99"
				},
				"quantity": 2,
				"special_instructions": "Extra spicy"
			}
		]
	}
]
```

---

### 2. Accepting/Rejecting Orders

#### Accept Order

**Endpoint:** `PUT /restaurants/:restaurantId/orders/:orderId`

**Query Parameters:**

- `action=accept`
- `estimatedPrepTime` (optional): Minutes until ready (default: 25)

```bash
curl -X PUT "http://localhost:3456/restaurants/restaurant-uuid/orders/order-uuid?action=accept&estimatedPrepTime=30"
```

**Response:**

```json
{
	"success": true,
	"action": "accept",
	"order": {
		"id": "order-uuid",
		"status": "CONFIRMED"
	},
	"signaled": true
}
```

**What Happens:**

1. Order status updated to `CONFIRMED` in database
2. Checkout workflow signaled with acceptance
3. System starts rider assignment workflow
4. User receives notification: "Restaurant confirmed your order"

---

#### Reject Order

**Endpoint:** `PUT /restaurants/:restaurantId/orders/:orderId`

**Query Parameters:**

- `action=reject`
- `reason` (required): Reason for rejection

```bash
curl -X PUT "http://localhost:3456/restaurants/restaurant-uuid/orders/order-uuid?action=reject&reason=Out%20of%20ingredients"
```

**Response:**

```json
{
	"success": true,
	"action": "reject",
	"signaled": true
}
```

**What Happens:**

1. Order status updated to `REJECTED` in database
2. Checkout workflow signaled with rejection
3. Payment automatically refunded to user
4. User receives notification: "Restaurant rejected: Out of ingredients"

---

### 3. Updating Order Preparation Status

#### Update Order Status

**Endpoint:** `PUT /restaurants/orders/:orderId/status`

**Request Body:**

```json
{
	"status": "PREPARING"
}
```

**Valid Restaurant Status Transitions:**

- `CONFIRMED` → `PREPARING`

```bash
curl -X PUT http://localhost:3456/restaurants/orders/order-uuid/status \
  -H "Content-Type: application/json" \
  -d '{"status": "PREPARING"}'
```

**Response:**

```json
{
	"success": true,
	"orderId": "order-uuid",
	"status": "PREPARING",
	"message": "Order status updated to PREPARING"
}
```

**What Happens:**

1. Order tracking workflow receives status update
2. Status validated against allowed transitions
3. Database updated
4. All parties in order channel receive WebSocket update:

```json
{
	"event": "order_update",
	"data": {
		"type": "ORDER_STATUS_UPDATE",
		"orderId": "order-uuid",
		"status": "PREPARING",
		"source": "restaurant",
		"message": "Restaurant is preparing your order",
		"timestamp": 1696512000000
	}
}
```

---

### 4. WebSocket Connection (Restaurant)

#### Connect and Authenticate

```javascript
const ws = new WebSocket("ws://localhost:9080")

ws.onopen = () => {
	ws.send(
		JSON.stringify({
			event: "connect_user",
			data: {
				userId: "restaurant-uuid",
				userType: "restaurant",
			},
		})
	)
}
```

#### Listen for Order Requests

```javascript
ws.onmessage = event => {
	const message = JSON.parse(event.data)

	if (message.event === "order_request") {
		handleNewOrderRequest(message.data)
	}
}

function handleNewOrderRequest(orderData) {
	console.log("New order request:", orderData)
	/*
  {
    orderId: 'order-uuid',
    checkoutId: 'checkout-uuid',
    customer: {
      name: 'Jane Smith',
      phone: '+1122334455'
    },
    items: [
      {
        name: 'Chicken Biryani',
        quantity: 2,
        price: '12.99',
        special_instructions: 'Extra spicy'
      }
    ],
    total: '32.59',
    deliveryAddress: {
      street: '123 Main St',
      building: 'Apt 4B',
      city: 'New York'
    },
    timestamp: 1696512000000
  }
  */

	// Show notification to restaurant
	showOrderNotification(orderData)

	// Auto-join order channel for updates
	ws.send(
		JSON.stringify({
			event: "join_channel",
			data: {
				channelName: `order:${orderData.orderId}`,
				userId: "restaurant-uuid",
				userType: "restaurant",
			},
		})
	)
}
```

---

## Rider App Integration

### 1. Rider Availability Management

#### Toggle Availability

**Endpoint:** `PUT /riders/:riderId/availability`

**Request Body:**

```json
{
	"lat": 40.7128,
	"lng": -74.006
}
```

**Response:**

```json
{
	"id": "rider-uuid",
	"first_name": "John",
	"last_name": "Doe",
	"phone": "+1234567890",
	"phone_verified": true,
	"email": "john.doe@email.com",
	"email_verified": true,
	"identity_docs": [],
	"is_accepting_orders": true,
	"total_earnings": "0",
	"total_orders_delivered": 0,
	"rating_avg": "0",
	"is_deleted": false,
	"deleted_at": null,
	"deleted_by": null,
	"created_at": "2025-09-22T02:16:28.230Z",
	"last_modified_at": "2025-09-22T02:57:11.047Z"
}
```

**Important:**

- Toggles the rider's `is_accepting_orders` status
- If toggling to `true` (going online), adds rider to Redis active pool
- If toggling to `false` (going offline), rider is removed from active pool
- Availability is automatically set to `false` when rider accepts an order
- Rider must manually toggle availability to `true` after completing delivery

---

### 2. Location Updates

#### Update Rider Location

**Endpoint:** `PUT /riders/:riderId/location`

**Request Body:**

```json
{
	"latitude": 40.7128,
	"longitude": -74.006,
	"orderId": "order-uuid"
}
```

**Parameters:**

- `latitude` (required): Current latitude
- `longitude` (required): Current longitude
- `orderId` (optional): If provided, sends location update to order tracking workflow

```bash
curl -X PUT http://localhost:3456/riders/rider-uuid/location \
  -H "Content-Type: application/json" \
  -d '{"latitude": 40.7128, "longitude": -74.0060, "orderId": "order-uuid"}'
```

**Response:**

```json
{
	"success": true,
	"latitude": 40.7128,
	"longitude": -74.006
}
```

**What Happens (if orderId provided):**

1. Rider location updated in database
2. Order tracking workflow receives location update
3. All parties in order channel receive WebSocket update:

```json
{
	"event": "order_update",
	"data": {
		"type": "RIDER_LOCATION_UPDATE",
		"orderId": "order-uuid",
		"riderId": "rider-uuid",
		"location": {
			"latitude": 40.7128,
			"longitude": -74.006
		},
		"message": "Rider location updated",
		"timestamp": 1696512000000
	}
}
```

**Best Practices:**

- Update location every 10-30 seconds while on delivery
- Only send `orderId` when rider has an active delivery
- Stop sending updates after marking order as delivered

---

### 3. Accepting/Rejecting Delivery Requests

#### Accept Delivery

**Endpoint:** `PUT /riders/:riderId/orders/:orderId`

**Query Parameters:**

- `action=accept`

```bash
curl -X PUT "http://localhost:3456/riders/rider-uuid/orders/order-uuid?action=accept"
```

**Response:**

```json
{
	"success": true,
	"action": "accept",
	"order": {
		"id": "order-uuid",
		"status": "RIDER_ASSIGNED",
		"delivery_partner_id": "rider-uuid"
	},
	"signaled": true,
	"notificationSignaled": true
}
```

**What Happens:**

1. Order updated with rider assignment
2. Rider availability set to `false`
3. Other riders' delivery requests for this order invalidated
4. Checkout workflow signaled (if still active)
5. Order tracking workflow started
6. User and restaurant receive notification: "Rider assigned"

---

#### Reject Delivery

**Endpoint:** `PUT /riders/:riderId/orders/:orderId`

**Query Parameters:**

- `action=reject`
- `reason` (optional): Reason for rejection

```bash
curl -X PUT "http://localhost:3456/riders/rider-uuid/orders/order-uuid?action=reject&reason=Too%20far"
```

**Response:**

```json
{
	"success": true,
	"action": "reject",
	"signaled": false,
	"notificationSignaled": true
}
```

**What Happens:**

1. Rejection logged
2. System continues searching for other available riders
3. If all riders reject, order may be cancelled and refunded

---

### 4. Updating Delivery Status

#### Update Order Status

**Endpoint:** `PUT /riders/:riderId/orders/:orderId/status`

**Query Parameters:**

- `status`: New order status

**Valid Rider Status Transitions:**

- `RIDER_ASSIGNED` → `PICKED_UP` (picked up from restaurant)
- `PICKED_UP` → `ON_THE_WAY` (heading to customer)
- `ON_THE_WAY` → `DELIVERED` (delivered to customer)

```bash
curl -X PUT "http://localhost:3456/riders/rider-uuid/orders/order-uuid/status?status=PICKED_UP"
```

**Response:**

```json
{
	"success": true,
	"orderId": "order-uuid",
	"status": "PICKED_UP",
	"message": "Order status updated to PICKED_UP"
}
```

**What Happens:**

1. Order tracking workflow receives status update
2. Status validated against allowed transitions
3. Database updated
4. All parties receive WebSocket update with appropriate message:
   - `PICKED_UP`: "Rider picked up your order"
   - `ON_THE_WAY`: "Your order is on the way"
   - `DELIVERED`: "Order delivered! Enjoy your meal!"

---

#### Alternative Endpoint (Global)

**Endpoint:** `PUT /riders/orders/:orderId/status`

**Request Body:**

```json
{
	"status": "ON_THE_WAY"
}
```

Same functionality as above, but doesn't require rider ID in path.

---

### 5. Get Rider Orders

**Endpoint:** `GET /riders/:riderId/orders`

```bash
curl http://localhost:3456/riders/rider-uuid/orders
```

**Response:**

```json
[
	{
		"id": "order-uuid",
		"customer_id": "user-uuid",
		"restaurant_id": "restaurant-uuid",
		"status": "ON_THE_WAY",
		"total_order_value": "32.59",
		"created_at": "2025-10-05T11:30:00Z",
		"estimated_delivery_time": "2025-10-05T12:15:00Z",
		"restaurant": {
			"name": "Taste of India",
			"phone": "+1234567890",
			"address": {
				"street": "456 Restaurant Ave",
				"city": "New York"
			}
		},
		"customer": {
			"first_name": "Jane",
			"phone": "+1122334455"
		},
		"delivery_to_address": {
			"street": "123 Main St",
			"building": "Apt 4B",
			"city": "New York",
			"zip_code": "10001"
		}
	}
]
```

---

### 6. WebSocket Connection (Rider)

#### Connect and Authenticate

```javascript
const ws = new WebSocket("ws://localhost:9080")

ws.onopen = () => {
	ws.send(
		JSON.stringify({
			event: "connect_user",
			data: {
				userId: "rider-uuid",
				userType: "rider",
			},
		})
	)
}
```

#### Listen for Delivery Requests

```javascript
ws.onmessage = event => {
	const message = JSON.parse(event.data)

	if (message.event === "order_request") {
		handleDeliveryRequest(message.data)
	}
}

function handleDeliveryRequest(deliveryData) {
	console.log("New delivery request:", deliveryData)
	/*
  {
    orderId: 'order-uuid',
    checkoutId: 'checkout-uuid',
    restaurant: {
      name: 'Taste of India',
      phone: '+1234567890',
      address: {
        street: '456 Restaurant Ave',
        coordinates: { latitude: 40.7589, longitude: -73.9851 }
      }
    },
    customer: {
      name: 'Jane Smith',
      phone: '+1122334455'
    },
    deliveryAddress: {
      street: '123 Main St',
      building: 'Apt 4B',
      coordinates: { latitude: 40.7128, longitude: -74.0060 }
    },
    distance: 2.5, // km
    deliveryFee: '2.99',
    estimatedTime: 15, // minutes
    timestamp: 1696512000000,
    expiresAt: 1696512300000 // 5 minutes to accept
  }
  */

	// Show notification to rider
	showDeliveryRequestNotification(deliveryData)
}
```

---

## WebSocket Integration

### Message Types Reference

#### Events Sent by Client

| Event          | Data                              | Description                         |
| -------------- | --------------------------------- | ----------------------------------- |
| `connect_user` | `{userId, userType}`              | Authenticate WebSocket connection   |
| `join_channel` | `{channelName, userId, userType}` | Join a specific channel for updates |
| `heartbeat`    | `{}`                              | Keep connection alive               |
| `status`       | `{}`                              | Get server status                   |
| `health_check` | `{}`                              | Check server health                 |

#### Events Received by Client

| Event                  | Data                                               | Description                                   |
| ---------------------- | -------------------------------------------------- | --------------------------------------------- |
| `connection_confirmed` | `{userId, userType, queuedNotificationsDelivered}` | Connection successful                         |
| `channel_joined`       | `{channelName, userId, message}`                   | Successfully joined channel                   |
| `order_update`         | `{type, orderId, status, message, ...}`            | Real-time order status/location update        |
| `order_request`        | `{orderId, ...}`                                   | New order/delivery request                    |
| `notification`         | `{type, ...}`                                      | General notification (checkout updates, etc.) |
| `connection_error`     | `{error}`                                          | Connection failed                             |
| `channel_error`        | `{error}`                                          | Channel join failed                           |

---

### WebSocket Message Formats

#### Connect User

**Send:**

```json
{
	"event": "connect_user",
	"data": {
		"userId": "user-uuid",
		"userType": "user"
	}
}
```

**Receive:**

```json
{
	"event": "connection_confirmed",
	"data": {
		"userId": "user-uuid",
		"userType": "user",
		"timestamp": "2025-10-05T12:00:00.000Z",
		"message": "Successfully connected to notifications",
		"queuedNotificationsDelivered": 3
	}
}
```

---

#### Join Channel

**Send:**

```json
{
	"event": "join_channel",
	"data": {
		"channelName": "order:order-uuid",
		"userId": "user-uuid",
		"userType": "user"
	}
}
```

**Receive:**

```json
{
	"event": "channel_joined",
	"data": {
		"channelName": "order:order-uuid",
		"userId": "user-uuid",
		"userType": "user",
		"timestamp": "2025-10-05T12:00:00.000Z",
		"message": "Successfully joined channel order:order-uuid"
	}
}
```

---

#### Order Status Update

**Receive:**

```json
{
	"event": "order_update",
	"data": {
		"type": "ORDER_STATUS_UPDATE",
		"orderId": "order-uuid",
		"status": "PREPARING",
		"source": "restaurant",
		"message": "Restaurant is preparing your order",
		"timestamp": 1696512000000
	}
}
```

---

#### Rider Location Update

**Receive:**

```json
{
	"event": "order_update",
	"data": {
		"type": "RIDER_LOCATION_UPDATE",
		"orderId": "order-uuid",
		"riderId": "rider-uuid",
		"location": {
			"latitude": 40.7128,
			"longitude": -74.006
		},
		"message": "Rider location updated",
		"timestamp": 1696512000000
	}
}
```

---

#### Checkout Update

**Receive:**

```json
{
	"event": "notification",
	"data": {
		"type": "checkout_update",
		"checkout_id": "checkout-uuid",
		"step": "ORDER_CREATED",
		"message": "Order created successfully",
		"data": {
			"order_id": "order-uuid"
		},
		"timestamp": "2025-10-05T12:00:00.000Z"
	}
}
```

---

#### Order Request (Restaurant/Rider)

**Receive:**

```json
{
	"event": "order_request",
	"data": {
		"orderId": "order-uuid",
		"checkoutId": "checkout-uuid",
		"customer": {
			"name": "Jane Smith",
			"phone": "+1122334455"
		},
		"items": [
			{
				"name": "Chicken Biryani",
				"quantity": 2,
				"price": "12.99"
			}
		],
		"total": "32.59",
		"deliveryAddress": {
			"street": "123 Main St",
			"city": "New York"
		},
		"timestamp": 1696512000000
	},
	"timestamp": "2025-10-05T12:00:00.000Z"
}
```

---

## Order Status Flow

### Complete Status Diagram

```
PENDING_PAYMENT (Initial - rare, internal)
    ↓
PLACED (After payment successful)
    ↓
CONFIRMED (Restaurant accepted) ←── Restaurant Action
    ↓
PREPARING (Restaurant preparing) ←── Restaurant Action
    ↓
RIDER_ASSIGNED (Rider accepted delivery)
    ↓
PICKED_UP (Rider picked up from restaurant) ←── Rider Action
    ↓
ON_THE_WAY (Rider heading to customer) ←── Rider Action
    ↓
DELIVERED (Order delivered) ←── Rider Action

Alternative Flows:
REJECTED (Restaurant rejected) ←── Restaurant Action
CANCELLED (System/User cancelled)
DISPUTED (Issue raised after delivery)
```

---

### Status Transition Rules

| Current Status         | Allowed Next Status | Actor      | Endpoint                                                   |
| ---------------------- | ------------------- | ---------- | ---------------------------------------------------------- |
| PLACED                 | CONFIRMED           | Restaurant | `PUT /restaurants/:id/orders/:orderId?action=accept`       |
| PLACED                 | REJECTED            | Restaurant | `PUT /restaurants/:id/orders/:orderId?action=reject`       |
| CONFIRMED              | PREPARING           | Restaurant | `PUT /restaurants/orders/:orderId/status`                  |
| PREPARING              | RIDER_ASSIGNED      | System     | Automatic on rider acceptance                              |
| RIDER_ASSIGNED         | PICKED_UP           | Rider      | `PUT /riders/:id/orders/:orderId/status?status=PICKED_UP`  |
| PICKED_UP              | ON_THE_WAY          | Rider      | `PUT /riders/:id/orders/:orderId/status?status=ON_THE_WAY` |
| ON_THE_WAY             | DELIVERED           | Rider      | `PUT /riders/:id/orders/:orderId/status?status=DELIVERED`  |
| Any (before DELIVERED) | CANCELLED           | System     | Automatic on timeout/failure                               |

---

## Testing with Postman/Hoppscotch

### Setting Up WebSocket Testing

#### Using Hoppscotch

1. **Open Hoppscotch WebSocket Client:**

   - Go to https://hoppscotch.io/realtime/websocket
   - Or use desktop app

2. **Connect to WebSocket:**

   ```
   URL: ws://localhost:9080
   ```

   - Click "Connect"

3. **Authenticate:**

   ```json
   {
   	"event": "connect_user",
   	"data": {
   		"userId": "test-user-123",
   		"userType": "user"
   	}
   }
   ```

   - Paste in message box
   - Click "Send"

4. **Wait for Confirmation:**

   - You should receive `connection_confirmed` event

5. **Join Order Channel:**

   ```json
   {
   	"event": "join_channel",
   	"data": {
   		"channelName": "order:test-order-123",
   		"userId": "test-user-123",
   		"userType": "user"
   	}
   }
   ```

6. **Monitor Messages:**
   - All incoming messages appear in the "Messages" panel
   - Use filters to show only specific event types

---

#### Using Postman

1. **Create New WebSocket Request:**

   - Click "New" → "WebSocket Request"
   - Enter URL: `ws://localhost:9080`
   - Click "Connect"

2. **Send Authentication:**

   - In "Message" tab, select "JSON"
   - Paste connect_user message (see above)
   - Click "Send"

3. **View Responses:**

   - Switch to "Messages" tab
   - All incoming messages displayed with timestamps

4. **Save Common Messages:**
   - Save frequently used messages as "Examples"
   - Quickly send with one click

---

### Testing Complete Order Flow

#### Step 1: Add Items to Cart

**Postman Collection:**

```
PUT http://localhost:3456/carts/test-user-123
Content-Type: application/json

{
  "items": [
    {
      "food_item_id": "food-item-uuid",
      "quantity": 2,
      "variant_id": null
    }
  ],
  "restaurant_id": "restaurant-uuid"
}
```

**Expected Response:**

```json
{
	"success": true,
	"cart_id": "cart-uuid-123",
	"items_count": 1,
	"total_items": 2
}
```

---

#### Step 2: Get Pricing (REQUIRED)

```
GET http://localhost:3456/pricing/cart-uuid-123
```

**Expected Response:**

```json
{
	"item_total": 25.98,
	"delivery_fee": 2.99,
	"platform_fee": 1.5,
	"discount_amount": 0.0,
	"amount_to_pay": 30.47
}
```

**⚠️ Save this pricing object - you'll need it for checkout**

---

#### Step 3: Connect WebSocket (All 3 Clients)

Open 3 WebSocket connections:

**User WebSocket:**

```
ws://localhost:9080

Send:
{
  "event": "connect_user",
  "data": {
    "userId": "test-user-123",
    "userType": "user"
  }
}
```

**Restaurant WebSocket:**

```
ws://localhost:9080

Send:
{
  "event": "connect_user",
  "data": {
    "userId": "test-restaurant-456",
    "userType": "restaurant"
  }
}
```

**Rider WebSocket:**

```
ws://localhost:9080

Send:
{
  "event": "connect_user",
  "data": {
    "userId": "test-rider-789",
    "userType": "rider"
  }
}
```

---

#### Step 4: Initiate Checkout

**⚠️ Use the exact pricing from Step 2**

```
PUT http://localhost:3456/orders/checkout/cart-uuid-123
Content-Type: application/json

{
  "user_id": "test-user-123",
  "delivery_address_id": "address-uuid",
  "payment_method_id": "payment-uuid",
  "pricing": {
    "item_total": 25.98,
    "delivery_fee": 2.99,
    "platform_fee": 1.50,
    "discount_amount": 0.00,
    "amount_to_pay": 30.47
  },
  "metadata": {
    "delivery_instructions": ["LEAVE_AT_DOOR"]
  }
}
```

**Watch User WebSocket for:**

- `PAYMENT_PROCESSING`
- `ORDER_CREATING`
- `ORDER_CREATED` → Save `order_id`
- `WAITING_RESTAURANT_ACCEPTANCE`

**Watch Restaurant WebSocket for:**

- `order_request` event with order details

---

#### Step 5: Restaurant Accepts Order

```
PUT http://localhost:3456/restaurants/test-restaurant-456/orders/{order_id}?action=accept&estimatedPrepTime=25
```

**Watch User WebSocket for:**

- `RIDER_ASSIGNMENT`

**Watch Rider WebSocket for:**

- `order_request` event with delivery details

---

#### Step 6: Rider Accepts Delivery

```
PUT http://localhost:3456/riders/test-rider-789/orders/{order_id}?action=accept
```

**Watch User WebSocket for:**

- `ORDER_CHANNEL_CREATED`
- `COMPLETED`

---

#### Step 7: Join Order Channel (All Clients)

**User, Restaurant, Rider all send:**

```json
{
	"event": "join_channel",
	"data": {
		"channelName": "order:{order_id}",
		"userId": "respective-user-id",
		"userType": "respective-type"
	}
}
```

---

#### Step 8: Restaurant Updates to Preparing

```
PUT http://localhost:3456/restaurants/orders/{order_id}/status
Content-Type: application/json

{
  "status": "PREPARING"
}
```

**All 3 WebSockets receive:**

```json
{
	"event": "order_update",
	"data": {
		"type": "ORDER_STATUS_UPDATE",
		"orderId": "{order_id}",
		"status": "PREPARING",
		"source": "restaurant",
		"message": "Restaurant is preparing your order"
	}
}
```

---

#### Step 9: Rider Picks Up Order

```
PUT http://localhost:3456/riders/test-rider-789/orders/{order_id}/status?status=PICKED_UP
```

**All WebSockets receive:**

- Status update: `PICKED_UP`

---

#### Step 10: Rider Updates Location

```
PUT http://localhost:3456/riders/test-rider-789/location
Content-Type: application/json

{
  "latitude": 40.7128,
  "longitude": -74.0060,
  "orderId": "{order_id}"
}
```

**All WebSockets receive:**

```json
{
	"event": "order_update",
	"data": {
		"type": "RIDER_LOCATION_UPDATE",
		"orderId": "{order_id}",
		"riderId": "test-rider-789",
		"location": {
			"latitude": 40.7128,
			"longitude": -74.006
		},
		"message": "Rider location updated"
	}
}
```

---

#### Step 11: Rider Marks On The Way

```
PUT http://localhost:3456/riders/test-rider-789/orders/{order_id}/status?status=ON_THE_WAY
```

---

#### Step 12: Rider Delivers Order

```
PUT http://localhost:3456/riders/test-rider-789/orders/{order_id}/status?status=DELIVERED
```

**All WebSockets receive:**

- Status update: `DELIVERED`
- Message: "Order delivered! Enjoy your meal!"

---

## Error Handling

### Common Error Responses

#### 400 Bad Request

```json
{
	"statusCode": 400,
	"message": "Invalid cart",
	"error": "Bad Request"
}
```

**Causes:**

- Cart not found
- Invalid payment method
- Items from multiple restaurants

---

#### 404 Not Found

```json
{
	"statusCode": 404,
	"message": "Order not found",
	"error": "Not Found"
}
```

---

#### 500 Internal Server Error

```json
{
	"statusCode": 500,
	"message": "Internal server error",
	"error": "Internal Server Error"
}
```

---

### Checkout Failure Scenarios

#### Payment Failure

**WebSocket Message:**

```json
{
	"event": "notification",
	"data": {
		"type": "checkout_update",
		"checkout_id": "checkout-uuid",
		"step": "FAILED",
		"message": "Checkout failed - payment processing failed",
		"error": "Payment declined"
	}
}
```

**Action:** Show error to user, cart remains intact

---

#### Restaurant Rejection

**WebSocket Message:**

```json
{
	"event": "notification",
	"data": {
		"type": "checkout_update",
		"step": "FAILED",
		"message": "Checkout failed - restaurant did not accept the order",
		"error": "Restaurant rejected: Out of ingredients"
	}
}
```

**Action:** Payment automatically refunded, notify user

---

#### Rider Assignment Timeout

**WebSocket Message:**

```json
{
	"event": "order_update",
	"data": {
		"type": "ORDER_TIMEOUT",
		"orderId": "order-uuid",
		"message": "Order timed out after 5 hours. Processing cancellation and refund..."
	}
}
```

**Action:** Order cancelled, payment refunded, notify all parties

---

### WebSocket Error Messages

#### Connection Error

```json
{
	"event": "connection_error",
	"data": {
		"error": "Invalid connection data. Required: userId and userType",
		"timestamp": "2025-10-05T12:00:00.000Z"
	}
}
```

---

#### Channel Error

```json
{
	"event": "channel_error",
	"data": {
		"error": "Channel name is required",
		"timestamp": "2025-10-05T12:00:00.000Z"
	}
}
```

---

### Workflow Update Failures

When sending status updates to tracking workflow:

**Success Response:**

```json
{
	"success": true,
	"orderId": "order-uuid",
	"status": "PREPARING",
	"message": "Order status updated to PREPARING"
}
```

**Failure Response:**

```json
{
	"success": false,
	"error": "Workflow not found or has already completed"
}
```

**Causes:**

- Workflow timed out (5 hours)
- Order already delivered/cancelled
- Invalid status transition

---

## Common Integration Patterns

### Pattern 1: Order Polling + WebSocket

**Use Case:** Ensure UI is always in sync, even if WebSocket disconnects

```javascript
// Connect WebSocket for real-time updates
const ws = connectWebSocket()

// Also poll orders every 30 seconds as fallback
setInterval(async () => {
	const orders = await fetch(`/orders/user/${userId}`).then(r => r.json())
	updateOrdersList(orders)
}, 30000)

// WebSocket takes priority
ws.onmessage = event => {
	const message = JSON.parse(event.data)
	if (message.event === "order_update") {
		updateOrderInRealTime(message.data)
	}
}
```

---

### Pattern 2: Pessimistic UI Updates

**Use Case:** Wait for server confirmation before updating UI (Recommended)

```javascript
async function updateOrderStatus(orderId, newStatus) {
	// Show loading state
	displayLoadingState(true)

	try {
		const response = await fetch(`/restaurants/orders/${orderId}/status`, {
			method: "PUT",
			headers: { "Content-Type": "application/json" },
			body: JSON.stringify({ status: newStatus }),
		})

		const result = await response.json()

		if (result.success) {
			// Only update UI after server confirms
			displayStatus(newStatus)
			showSuccess("Status updated successfully")
		} else {
			// Keep previous status
			showError(result.error)
		}
	} catch (error) {
		// Keep previous status on network error
		showError("Network error. Please try again.")
	} finally {
		displayLoadingState(false)
	}
}
```

**Why Pessimistic?**

- Prevents inconsistent state between client and server
- Status updates trigger workflow actions and WebSocket broadcasts
- Reverting optimistic updates can confuse users seeing real-time updates
- Loading states provide clear feedback without false states

---

### Pattern 3: Channel Management

**Use Case:** Automatically join/leave order channels

```javascript
class OrderChannelManager {
	constructor(ws, userId, userType) {
		this.ws = ws
		this.userId = userId
		this.userType = userType
		this.activeChannels = new Set()
	}

	joinOrderChannel(orderId) {
		const channelName = `order:${orderId}`

		if (this.activeChannels.has(channelName)) {
			return // Already joined
		}

		this.ws.send(
			JSON.stringify({
				event: "join_channel",
				data: {
					channelName,
					userId: this.userId,
					userType: this.userType,
				},
			})
		)

		this.activeChannels.add(channelName)
	}

	leaveAllChannels() {
		// Channels are automatically left on disconnect
		this.activeChannels.clear()
	}

	onChannelJoined(channelName) {
		console.log(`Joined ${channelName}`)
	}
}
```

---

### Pattern 4: Reconnection with Queued Messages

**Use Case:** Handle WebSocket disconnections gracefully

```javascript
class ResilientWebSocket {
	constructor(url, userId, userType) {
		this.url = url
		this.userId = userId
		this.userType = userType
		this.reconnectAttempts = 0
		this.maxReconnectAttempts = 10
		this.connect()
	}

	connect() {
		this.ws = new WebSocket(this.url)

		this.ws.onopen = () => {
			console.log("WebSocket connected")
			this.reconnectAttempts = 0

			// Authenticate
			this.send({
				event: "connect_user",
				data: {
					userId: this.userId,
					userType: this.userType,
				},
			})
		}

		this.ws.onclose = () => {
			console.log("WebSocket disconnected")
			this.reconnect()
		}

		this.ws.onerror = error => {
			console.error("WebSocket error:", error)
		}

		this.ws.onmessage = event => {
			const message = JSON.parse(event.data)

			if (message.event === "connection_confirmed") {
				console.log(
					"Queued notifications:",
					message.data.queuedNotificationsDelivered
				)
				this.onReady()
			}
		}
	}

	reconnect() {
		if (this.reconnectAttempts >= this.maxReconnectAttempts) {
			console.error("Max reconnection attempts reached")
			return
		}

		this.reconnectAttempts++
		const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000)

		console.log(
			`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`
		)

		setTimeout(() => {
			this.connect()
		}, delay)
	}

	send(data) {
		if (this.ws.readyState === WebSocket.OPEN) {
			this.ws.send(JSON.stringify(data))
		} else {
			console.warn("WebSocket not ready, message not sent")
		}
	}

	onReady() {
		// Override this method
	}
}
```

---

### Pattern 5: Location Batching

**Use Case:** Reduce network calls for rider location updates

```javascript
class LocationUpdateBatcher {
	constructor(riderId, orderId, intervalMs = 15000) {
		this.riderId = riderId
		this.orderId = orderId
		this.intervalMs = intervalMs
		this.lastLocation = null
		this.updateTimer = null
	}

	start() {
		this.updateTimer = setInterval(() => {
			if (this.lastLocation) {
				this.sendLocationUpdate(this.lastLocation)
			}
		}, this.intervalMs)
	}

	stop() {
		if (this.updateTimer) {
			clearInterval(this.updateTimer)
			this.updateTimer = null
		}
	}

	updateLocation(latitude, longitude) {
		this.lastLocation = { latitude, longitude }
	}

	async sendLocationUpdate(location) {
		try {
			await fetch(`/riders/${this.riderId}/location`, {
				method: "PUT",
				headers: { "Content-Type": "application/json" },
				body: JSON.stringify({
					latitude: location.latitude,
					longitude: location.longitude,
					orderId: this.orderId,
				}),
			})
		} catch (error) {
			console.error("Failed to send location update:", error)
		}
	}
}

// Usage
const batcher = new LocationUpdateBatcher("rider-uuid", "order-uuid", 15000)
batcher.start()

// GPS updates
navigator.geolocation.watchPosition(position => {
	batcher.updateLocation(position.coords.latitude, position.coords.longitude)
})

// When delivery complete
batcher.stop()
```

---

## Best Practices

### For User Apps

1. **ALWAYS call pricing API after cart updates**

   - Call `GET /pricing/:cartId` after every cart modification
   - Display pricing to user before checkout
   - Use exact pricing response in checkout request

2. **Always connect WebSocket before checkout**

   - Ensures real-time updates during payment/order creation

3. **Join order channel immediately after `ORDER_CREATED`**

   - Don't wait for `COMPLETED` step

4. **Handle all checkout steps**

   - Show loading states for each step
   - Display appropriate messages

5. **Implement reconnection logic**

   - WebSocket may disconnect during long orders
   - Queued notifications are delivered on reconnect

6. **Poll orders as fallback**
   - Don't rely solely on WebSocket
   - Refresh order list periodically

---

### For Restaurant Apps

1. **Auto-accept or show notification within 2 minutes**

   - Orders may timeout if not acknowledged

2. **Update to PREPARING immediately after acceptance**

   - Improves user experience

3. **Keep WebSocket connected during business hours**

   - Don't disconnect/reconnect frequently

4. **Join order channels for active orders**
   - Receive rider updates

---

### For Rider Apps

1. **Update location frequently (every 10-30 seconds)**

   - Use location batching to reduce battery drain

2. **Only send orderId when on active delivery**

   - Don't include orderId when just updating rider location

3. **Mark as PICKED_UP before leaving restaurant**

   - Don't wait until en route

4. **Update to ON_THE_WAY before DELIVERED**

   - Follow proper status flow

5. **Handle network failures gracefully**
   - Queue location updates if offline
   - Retry failed status updates

---

## Support & Troubleshooting

### Common Issues

#### WebSocket not receiving messages

**Solution:**

1. Check WebSocket connection status
2. Verify `connect_user` was sent
3. Confirm `connection_confirmed` received
4. Check if channel was joined (`channel_joined` event)

---

#### Order stuck in WAITING_RESTAURANT_ACCEPTANCE

**Cause:** Restaurant hasn't responded or WebSocket disconnected

**Solution:**

1. Restaurant should check for queued notifications on reconnect
2. Check Redis for order mapping: `order_checkout_mapping:{orderId}:{restaurantId}`
3. If expired, order will auto-cancel and refund

---

#### Status update fails with "Workflow not found"

**Cause:** Order tracking workflow ended (timeout or completion)

**Solution:**

1. Check order status in database
2. If DELIVERED/CANCELLED, workflow is complete
3. If stuck, order may have timed out (5 hour limit)

---

#### Location updates not showing

**Cause:**

- `orderId` not included in request
- Order tracking workflow not active
- WebSocket channel not joined

**Solution:**

1. Ensure rider sends `orderId` with location update
2. Verify order is in RIDER_ASSIGNED, PICKED_UP, or ON_THE_WAY status
3. Check all parties joined `order:{orderId}` channel

---

### Debug Endpoints

#### Check WebSocket Server Status

```javascript
// Send via WebSocket
{
  "event": "status"
}

// Response
{
  "event": "status_response",
  "data": {
    "connectedClients": 5,
    "uptime": 3600000,
    "startTime": "2025-10-05T10:00:00.000Z"
  }
}
```

---

#### Check Health

```javascript
// Send via WebSocket
{
  "event": "health_check"
}

// Response
{
  "event": "health_check_response",
  "data": {
    "status": "healthy",
    "connectedClients": 5,
    "uptime": 3600000
  }
}
```

---

### Contact

For integration support:

- **API Docs:** http://localhost:3456/api-docs (Scalar API Reference)
- **GitHub Issues:** https://github.com/shaiknoorullah/hg-docker/issues

---

**Document Version:** 1.0.0
**Last Updated:** 2025-10-05
**Maintained By:** Shaik Noorullah
