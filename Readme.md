# Metal.build Documentation Enhancement Proposal

This document provides a comprehensive guide to Metal.build's platform, focusing on token management, holder operations, and platform capabilities. Each section includes practical examples with detailed explanations.

## Core Features Documentation

### 1. Installation & Setup

Metal.build can be installed using various package managers. The core and client packages provide all necessary functionality for both server-side and client-side applications.

```bash
# Choose your preferred package manager
npm install @metal/core @metal/client
bun add @metal/core @metal/client
```

### 2. Basic Configuration

The Metal client initialization requires your API key and project ID. The environment parameter helps distinguish between development and production settings, affecting rate limits and logging behavior.

```javascript
const metal = new Metal({
  apiKey: "YOUR_SECRET_API_KEY",
  projectId: "PROJECT_ID",
  environment: "production"
});
```

### 3. Holder Management

#### Creating or Retrieving Holders
A holder represents a wallet in the Metal ecosystem. The following function creates a new holder or retrieves an existing one using a custom user ID. This is particularly useful for mapping your application's users to Metal wallets.

```javascript
async function getOrCreateHolder(userId) {
  try {
    const response = await fetch(`https://api.metal.build/holder/${userId}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': metal.apiKey,
      }
    });

    const holder = await response.json();
    return holder;
  } catch (error) {
    console.error('Holder creation failed:', error);
    throw error;
  }
}
```

#### Token Withdrawal
This function enables holders to withdraw tokens to external addresses. It's essential for allowing users to transfer their tokens outside of your application's ecosystem.

```javascript
async function withdrawTokens(holderId, tokenId, amount) {
  const withdrawal = await metal.holders.withdraw({
    holderId,
    tokenId,
    amount,
    destination: "specified_address"
  });
  return withdrawal;
}
```

#### Token Spending
The spend operation is designed for merchant interactions, allowing holders to use their tokens for purchases or services within your ecosystem. Each transaction can include metadata for tracking purposes.

```javascript
async function spendTokens(holderId, tokenId, amount) {
  const transaction = await metal.holders.spend({
    holderId,
    tokenId,
    amount,
    metadata: {
      purpose: "purchase",
      timestamp: Date.now()
    }
  });
  return transaction;
}
```

### 4. Token Management

#### Token Creation
Creating a new token establishes a new asset in your ecosystem. The token configuration includes basic information like name and symbol, as well as metadata for additional token properties.

```javascript
async function createToken(tokenConfig) {
  const token = await metal.tokens.create({
    name: tokenConfig.name,
    symbol: tokenConfig.symbol,
    totalSupply: tokenConfig.supply,
    metadata: {
      description: tokenConfig.description,
      image: tokenConfig.imageUrl
    }
  });
  return token;
}
```

#### Token Distribution
This function handles the distribution of tokens to multiple holders. It's useful for airdrops, rewards, or initial token distribution events.

```javascript
async function distributeTokens(tokenId, distributions) {
  const batch = await metal.tokens.distribute({
    tokenId,
    distributions: distributions.map(dist => ({
      holderId: dist.holderId,
      amount: dist.amount
    }))
  });
  return batch;
}
```

#### Liquidity Pool Creation
Liquidity pools enable token trading and price discovery. The settings include fee structures and slippage tolerance to protect traders.

```javascript
async function createLiquidity(tokenId, amount) {
  const pool = await metal.liquidity.create({
    tokenId,
    initialAmount: amount,
    settings: {
      fee: "0.3%",
      slippageTolerance: "1%"
    }
  });
  return pool;
}
```

## Implementation Guidelines

### 1. Error Handling

A robust error handling system is crucial for maintaining application stability. This handler manages common API response codes and implements appropriate recovery strategies.

```javascript
const handleMetalError = (error) => {
  if (error.response) {
    switch (error.response.status) {
      case 401:
        refreshApiKey();
        break;
      case 429:
        implementBackoff();
        break;
      case 400:
        console.error('Validation Error:', error.response.data);
        break;
      default:
        console.error('Operation Failed:', error);
    }
  }
};
```

### 2. Best Practices

#### Batch Processing
When dealing with multiple operations, batch processing helps optimize API calls and improve performance. This implementation handles large distributions by breaking them into manageable chunks.

```javascript
async function batchDistribution(holders, tokenId) {
  const BATCH_SIZE = 100;
  const batches = holders.reduce((acc, holder, i) => {
    const batchIndex = Math.floor(i / BATCH_SIZE);
    if (!acc[batchIndex]) acc[batchIndex] = [];
    acc[batchIndex].push(holder);
    return acc;
  }, []);

  for (const batch of batches) {
    await metal.tokens.distribute({
      tokenId,
      distributions: batch
    });
  }
}
```

#### Retry Logic
Network operations can fail temporarily. This retry mechanism implements exponential backoff to handle rate limiting and temporary failures gracefully.

```javascript
async function withRetry(operation, maxRetries = 3) {
  let lastError;
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error;
      if (error.response?.status === 429) {
        await new Promise(resolve =>
          setTimeout(resolve, Math.pow(2, i) * 1000)
        );
        continue;
      }
      throw error;
    }
  }
  throw lastError;
}
```

## Deployment Options

### 1. Self-Hosted Deployment

For organizations requiring complete control over their infrastructure, Metal.build supports both Docker and traditional VM deployments.

```bash
# Docker deployment for containerized environments
docker run -d \
  --name metal-api \
  -e METAL_API_KEY="your_api_key" \
  -e METAL_PROJECT_ID="your_project_id" \
  -p 3000:3000 \
  metal/api-server

# Traditional VM deployment using PM2 for process management
git clone https://github.com/your-org/metal-implementation
cd metal-implementation
npm install
npm run build
pm2 start dist/server.js
```

### 2. Cloud Deployment

For serverless environments, Metal.build can be integrated with cloud functions. This example shows an AWS Lambda implementation.

```javascript
module.exports.handler = async (event) => {
  const metal = new Metal({
    apiKey: process.env.METAL_API_KEY,
    projectId: process.env.METAL_PROJECT_ID
  });
};
```

## Security Considerations

Implementing proper security measures is crucial. This includes rate limiting to prevent abuse and secure configuration management.

```javascript
const rateLimit = require('express-rate-limit');

const metalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100
});

app.use('/api/metal/', metalLimiter);

const secureConfig = {
  apiKey: process.env.METAL_API_KEY,
  projectId: process.env.METAL_PROJECT_ID,
  security: {
    allowedOrigins: ['https://your-domain.com'],
    rateLimiting: true,
    maxRequestsPerMinute: 100
  }
};
```
