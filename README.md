# Order Execution Engine

## Overview
The Order Execution Engine is a Node.js application designed to process **market orders** with decentralized exchange (DEX) routing. It utilizes WebSocket for real-time status updates, ensuring that users are informed about the state of their orders throughout the execution process.

## Why Market Orders?
**Market orders** were chosen as the primary order type because they represent the most common trading pattern and demonstrate the core architecture effectively. They execute immediately at current market prices, making them ideal for showcasing DEX routing logic and real-time WebSocket updates. The engine can be extended to support limit orders (price-triggered execution) and sniper orders (launch-detection) by adding conditional logic layers on top of the existing execution pipeline.

## Architecture & Design Decisions

### 1. **Tech Stack Rationale**
- **Fastify**: Chosen for its speed, low overhead, and built-in WebSocket support via `@fastify/websocket`
- **BullMQ + Redis**: Provides reliable queue management with automatic retry logic, essential for handling concurrent orders
- **PostgreSQL**: Persistent storage for order history and audit trails
- **TypeScript**: Ensures type safety and better code maintainability

### 2. **Order Processing Flow**

```
POST /api/orders/execute
         ↓
   Validate Order
         ↓
   Return orderId (HTTP 202)
         ↓
   Upgrade to WebSocket
         ↓
   [Queue Management]
         ↓
   Status: pending → routing → building → submitted → confirmed/failed
```

### 3. **DEX Router Implementation**
- **Multi-DEX Comparison**: Fetches quotes from both Raydium and Meteora simultaneously
- **Price Optimization**: Automatically selects the DEX with better price/liquidity
- **Slippage Protection**: Implements configurable slippage tolerance (default 1%)
- **Mock vs Real**: Supports both mock implementation (for development) and real devnet execution

### 4. **Concurrent Order Handling**
- **Queue Capacity**: Up to 10 concurrent orders processed simultaneously
- **Throughput**: Designed for ~100 orders/minute
- **Retry Logic**: Exponential backoff with max 3 attempts before marking as failed
- **Failure Persistence**: Failed orders logged for post-mortem analysis

### 5. **WebSocket Status Updates**
Each order progresses through lifecycle states:
- `pending`: Order received and queued
- `routing`: Comparing DEX prices
- `building`: Creating transaction
- `submitted`: Transaction sent to network
- `confirmed`: Successful execution (includes `txHash`)
- `failed`: Execution failed (includes error reason)

## Features
- ✅ **Market Order Processing**: Optimized for quick execution at current market prices
- ✅ **DEX Routing**: Integrates with Raydium and Meteora for optimal pricing
- ✅ **WebSocket Updates**: Real-time status streaming for order lifecycle
- ✅ **Order Queue Management**: BullMQ-based concurrent processing
- ✅ **Retry Logic**: Exponential backoff with configurable attempts
- ✅ **Order Persistence**: PostgreSQL storage for history and audit trails
- ✅ **Error Handling**: Comprehensive error tracking and logging

## Setup

### Prerequisites
- Node.js (v16+)
- PostgreSQL (v12+)
- Redis (v6+)

### Installation
1. **Clone the Repository**
   ```bash
   git clone <repository-url>
   cd order-execution-engine
   ```

2. **Install Dependencies**
   ```bash
   npm install
   ```

3. **Configure Environment**
   Create `.env` file:
   ```
   DATABASE_URL=postgresql://user:password@localhost:5432/order_execution
   REDIS_URL=redis://localhost:6379
   NODE_ENV=development
   PORT=3000
   MOCK_MODE=true
   ```

4. **Run Migrations**
   ```bash
   npm run migrate
   ```

5. **Start the Application**
   ```bash
   npm run dev
   ```

Server runs on `http://localhost:3000`

## API Usage

### Submit Market Order
```bash
POST /api/orders/execute
Content-Type: application/json

{
  "tokenIn": "SOL",
  "tokenOut": "USDC",
  "amount": 1.5,
  "slippage": 0.01
}
```

**Response (HTTP 202)**:
```json
{
  "orderId": "ord_abc123xyz",
  "status": "pending",
  "message": "Order queued for processing"
}
```

### WebSocket Connection
```javascript
const ws = new WebSocket('ws://localhost:3000/orders/ord_abc123xyz');

ws.onmessage = (event) => {
  const update = JSON.parse(event.data);
  console.log(update);
  // { status: "routing", timestamp: "...", message: "..." }
  // { status: "confirmed", txHash: "...", executedPrice: 23.5 }
};
```

## Extending to Other Order Types

### Limit Orders
Implement a price monitoring layer that polls target price thresholds. When the price reaches the limit, trigger the same market execution pipeline.

### Sniper Orders
Add token launch detection via blockchain listeners. Upon launch detection, execute the market order immediately with minimal slippage tolerance.

## Project Structure
```
src/
├── index.ts                 # Application entry point
├── engine/
│   ├── orderExecutor.ts     # Core order execution logic
│   └── dexRouter.ts         # DEX quote comparison & routing
├── websocket/
│   ├── server.ts            # WebSocket server setup
│   └── handlers.ts          # WebSocket event handlers
├── queue/
│   ├── orderQueue.ts        # BullMQ queue management
│   └── workers.ts           # Order processing workers
├── database/
│   ├── schema.ts            # Database schema definitions
│   └── migrations/          # DB migrations
├── types/
│   └── index.ts             # TypeScript type definitions
└── utils/
    ├── logger.ts            # Logging utility
    └── helpers.ts           # Helper functions
```

## Testing
```bash
npm test
```

Includes unit tests for:
- DEX routing logic
- Queue behavior
- WebSocket lifecycle
- Error handling & retries

## Contributing
Contributions are welcome! Please submit a pull request or open an issue.

## License
MIT License
