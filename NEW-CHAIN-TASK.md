# New Chain Integration: SDKs

## Overview

Add a new EVM chain to the Uniswap SDK packages. This repo is the **foundation** — Smart Order Router and Routing API depend on these changes.

## Prerequisites

Gather these values before starting:

```
CHAIN_ID=<numeric>            # e.g. 545
CHAIN_NAME=<name>             # e.g. FLOW_TESTNET (UPPER_SNAKE for enums)
NATIVE_SYMBOL=<symbol>        # e.g. FLOW
NATIVE_DECIMALS=<decimals>    # e.g. 18
WRAPPED_NATIVE_ADDRESS=<addr> # e.g. 0xd3bF53DAC106A0290B0483EcBC89d40FcC961f3e
V2_FACTORY=<addr>
V2_ROUTER=<addr>
V3_FACTORY=<addr>
V3_MULTICALL=<addr>
V3_QUOTER=<addr>
V3_NFT_POSITION_MANAGER=<addr>
V3_TICK_LENS=<addr>
V3_SWAP_ROUTER_02=<addr>          # optional
V3_MIXED_ROUTE_QUOTER_V1=<addr>   # optional
V3_MIXED_ROUTE_QUOTER_V2=<addr>   # optional
V4_POOL_MANAGER=<addr>
V4_POSITION_MANAGER=<addr>
V4_STATE_VIEW=<addr>
V4_QUOTER=<addr>
UNIVERSAL_ROUTER_V1_2=<addr>      # use 0x0 if not deployed
UNIVERSAL_ROUTER_V2_0=<addr>      # use 0x0 if not deployed
UNIVERSAL_ROUTER_V2_1=<addr>      # use 0x0 if not deployed
UR_V1_2_CREATION_BLOCK=<number>   # use 1 if not deployed
UR_V2_0_CREATION_BLOCK=<number>   # use 1 if not deployed
UR_V2_1_CREATION_BLOCK=<number>   # use 1 if not deployed
PERMIT2=<addr>                    # usually 0x000000000022D473030F116dDEE9F6B43aC78BA3
```

**Notes on V4 addresses:** The `v4PoolManagerAddress`, `v4PositionManagerAddress`, `v4StateView`, and `v4QuoterAddress` fields are **optional** in the `ChainAddresses` type. Omit them if V4 is not deployed on your chain.

## Steps

### 1. Add Chain ID to enum and supported list

**File:** `sdks/sdk-core/src/chains.ts`

```ts
// Add to ChainId enum (before closing brace):
CHAIN_NAME = CHAIN_ID,

// Add to SUPPORTED_CHAINS array (before `] as const`):
ChainId.CHAIN_NAME,

// Add to NativeCurrencyName enum IF the native currency is new (not ETH/MATIC/etc):
NATIVE_NAME = 'NATIVE_SYMBOL',
```

### 2. Add contract addresses

**File:** `sdks/sdk-core/src/addresses.ts`

```ts
// 1. Add V2 factory address to V2_FACTORY_ADDRESSES:
[ChainId.CHAIN_NAME]: 'V2_FACTORY',

// 2. Add V2 router address to V2_ROUTER_ADDRESSES:
[ChainId.CHAIN_NAME]: 'V2_ROUTER',

// 3. Create chain addresses constant (place before CHAIN_TO_ADDRESSES_MAP):
const CHAIN_NAME_ADDRESSES: ChainAddresses = {
  // V3 addresses (required):
  v3CoreFactoryAddress: 'V3_FACTORY',
  multicallAddress: 'V3_MULTICALL',
  quoterAddress: 'V3_QUOTER',
  nonfungiblePositionManagerAddress: 'V3_NFT_POSITION_MANAGER',
  tickLensAddress: 'V3_TICK_LENS',
  swapRouter02Address: 'V3_SWAP_ROUTER_02',           // omit if not deployed
  mixedRouteQuoterV1Address: 'V3_MIXED_ROUTE_QUOTER_V1', // omit if not deployed
  mixedRouteQuoterV2Address: 'V3_MIXED_ROUTE_QUOTER_V2', // omit if not deployed
  // V4 addresses (optional — omit entire block if V4 not deployed):
  v4PoolManagerAddress: 'V4_POOL_MANAGER',
  v4PositionManagerAddress: 'V4_POSITION_MANAGER',
  v4StateView: 'V4_STATE_VIEW',
  v4QuoterAddress: 'V4_QUOTER',
}

// 4. Register in CHAIN_TO_ADDRESSES_MAP:
[ChainId.CHAIN_NAME]: CHAIN_NAME_ADDRESSES,
```

This auto-populates: `V3_CORE_FACTORY_ADDRESSES`, `MULTICALL_ADDRESSES`, `QUOTER_ADDRESSES`, `NONFUNGIBLE_POSITION_MANAGER_ADDRESSES`, `TICK_LENS_ADDRESSES`, `MIXED_ROUTE_QUOTER_V1_ADDRESSES`.

### 3. Add wrapped native token

**File:** `sdks/sdk-core/src/entities/weth9.ts`

```ts
// Add to WETH9 object:
CHAIN_ID: new Token(CHAIN_ID, 'WRAPPED_NATIVE_ADDRESS', NATIVE_DECIMALS, 'WNATIVE', 'Wrapped Native'),
```

### 4. Add V3 pool init code hash (only if non-default)

**File:** `sdks/v3-sdk/src/constants.ts`

Only needed if the chain uses a different init code hash than `0xe34f199b19b2b4f47f68442619d555527d244f78a3297ea89325f843f87b8b54`. Add a case to `poolInitCodeHash()`:

```ts
case ChainId.CHAIN_NAME:
  return '0x<custom_hash>'
```

### 5. Add Universal Router config

**File:** `sdks/universal-router-sdk/src/utils/constants.ts`

Each chain must have entries for **all three** router versions (V1_2, V2_0, V2_1). Use `ZERO_ROUTER_CONFIG` (a sentinel with address `0x0...0` and `creationBlock: 1`) for any version **not deployed** on your chain.

```ts
// ZERO_ROUTER_CONFIG is defined at the top of the file:
// const ZERO_ROUTER_CONFIG: RouterConfig = { address: '0x0...0', creationBlock: 1 }

// Add to CHAIN_CONFIGS object:
[CHAIN_ID]: {
  weth: 'WRAPPED_NATIVE_ADDRESS',
  routerConfigs: {
    [UniversalRouterVersion.V1_2]: {
      address: 'UNIVERSAL_ROUTER_V1_2',
      creationBlock: UR_V1_2_CREATION_BLOCK,
    },
    [UniversalRouterVersion.V2_0]: {                    // use ZERO_ROUTER_CONFIG if not deployed
      address: 'UNIVERSAL_ROUTER_V2_0',
      creationBlock: UR_V2_0_CREATION_BLOCK,
    },
    [UniversalRouterVersion.V2_1]: {                    // use ZERO_ROUTER_CONFIG if not deployed
      address: 'UNIVERSAL_ROUTER_V2_1',
      creationBlock: UR_V2_1_CREATION_BLOCK,
    },
  },
},
```

**Example — Flow Testnet (only V1_2 and V2_1 deployed):**
```ts
[545]: {
  weth: '0xd3bF53DAC106A0290B0483EcBC89d40FcC961f3e',
  routerConfigs: {
    [UniversalRouterVersion.V1_2]: { address: '0x...', creationBlock: 12345 },
    [UniversalRouterVersion.V2_0]: ZERO_ROUTER_CONFIG,
    [UniversalRouterVersion.V2_1]: { address: '0xdAae...', creationBlock: 1 },
  },
},
```

### 6. Add Permit2 address (only if non-standard)

**File:** `sdks/permit2-sdk/src/constants.ts`

Only if Permit2 address differs from `0x000000000022D473030F116dDEE9F6B43aC78BA3`. Add case to `permit2Address()`:

```ts
case CHAIN_ID:
  return '0x<custom_permit2>'
```

### 7. Add UniswapX config (only if UniswapX is deployed)

**File:** `sdks/uniswapx-sdk/src/constants.ts`

Add entries to: `PERMIT2_MAPPING`, `UNISWAPX_ORDER_QUOTER_MAPPING`, `REACTOR_ADDRESS_MAPPING`, etc. Use zero addresses if not deployed.

### 8. Build and verify

```bash
cd sdks/sdks/sdk-core && yarn build
cd sdks/sdks/v2-sdk && yarn build
cd sdks/sdks/v3-sdk && yarn build
cd sdks/sdks/v4-sdk && yarn build
cd sdks/sdks/universal-router-sdk && yarn build
```

All must complete without errors.

## Files Changed (summary)

| File | Change |
|------|--------|
| `sdks/sdk-core/src/chains.ts` | Enum entry + SUPPORTED_CHAINS |
| `sdks/sdk-core/src/addresses.ts` | V2 maps + ChainAddresses + CHAIN_TO_ADDRESSES_MAP |
| `sdks/sdk-core/src/entities/weth9.ts` | Wrapped native token |
| `sdks/v3-sdk/src/constants.ts` | Init code hash (if non-default) |
| `sdks/universal-router-sdk/src/utils/constants.ts` | CHAIN_CONFIGS entry |
| `sdks/permit2-sdk/src/constants.ts` | Permit2 address (if non-standard) |
| `sdks/uniswapx-sdk/src/constants.ts` | UniswapX mappings (if deployed) |

## Validation

- `yarn build` passes for all 5 packages
- New chain ID is in `ChainId` enum and `SUPPORTED_CHAINS`
- All address maps resolve correctly for the new chain
- `WETH9[CHAIN_ID]` returns the wrapped native token
- `UNIVERSAL_ROUTER_ADDRESS(version, CHAIN_ID)` returns correct addresses
