# RPC Reference

## Endpoints

| Type | Mainnet | Testnet |
|------|---------|---------|
| HTTP | `https://mainnet.megaeth.com/rpc` | `https://carrot.megaeth.com/rpc` |
| WebSocket | `wss://mainnet.megaeth.com/ws` | `wss://carrot.megaeth.com/ws` |

Third-party providers: Alchemy, QuickNode (recommended for geo-distributed access)

## MegaETH-Specific Methods

### Instant Transaction Receipts

MegaETH supports synchronous transaction submission — get your receipt in <10ms instead of polling.

**Two equivalent methods:**

| Method | Origin | Status |
|--------|--------|--------|
| `realtime_sendRawTransaction` | MegaETH original | Supported |
| `eth_sendRawTransactionSync` | EIP-7966 standard | Supported (proxied) |

**History:** MegaETH created `realtime_sendRawTransaction` first. Later, `eth_sendRawTransactionSync` was standardized as EIP-7966. MegaETH now proxies both — they are functionally identical. Use whichever you prefer; `eth_sendRawTransactionSync` is recommended for cross-chain compatibility.

```bash
# Both work identically:
curl -X POST https://mainnet.megaeth.com/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "eth_sendRawTransactionSync",
    "params": ["0x...signedTx"],
    "id": 1
  }'

# Or use the original:
curl -X POST https://mainnet.megaeth.com/rpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "realtime_sendRawTransaction",
    "params": ["0x...signedTx"],
    "id": 1
  }'
```

**Response:** Full transaction receipt (same schema as `eth_getTransactionReceipt`)

**Corner cases on retry:**
- Tx still in pool: "already known" error
- Tx already executed: "nonce too low" error (future: will return receipt directly)

### eth_getLogsWithCursor

Paginated log queries for large result sets:

```javascript
const response = await client.request({
  method: 'eth_getLogsWithCursor',
  params: [{
    address: '0x...',
    topics: ['0x...'],
    fromBlock: '0x0',
    toBlock: 'latest'
  }]
});
// Returns { logs: [...], cursor: "..." }
// Use cursor in next request to continue
```

### Mini-Block Subscription

```javascript
{
  "jsonrpc": "2.0",
  "method": "eth_subscribe",
  "params": ["miniBlocks"],
  "id": 1
}
```

**Mini-block schema:**
```json
{
  "payload_id": "0x...",
  "block_number": 12345,
  "index": 0,
  "tx_offset": 0,
  "log_offset": 0,
  "gas_offset": 0,
  "timestamp": 1704067200000,
  "gas_used": 21000,
  "transactions": [...],
  "receipts": [...]
}
```

Mini-blocks are ephemeral — cannot be fetched via RPC after assembly into blocks.

## WebSocket Guidelines

### Keepalive Required

Send `eth_chainId` every 30 seconds to prevent disconnection:

```javascript
const ws = new WebSocket('wss://mainnet.megaeth.com/ws');

const keepalive = setInterval(() => {
  ws.send(JSON.stringify({
    jsonrpc: '2.0',
    method: 'eth_chainId',
    params: [],
    id: Date.now()
  }));
}, 30000);

ws.on('close', () => clearInterval(keepalive));
```

### Limits

- 50 WebSocket connections per VIP endpoint
- 10 subscriptions per connection (+1 on subscribe, -1 on unsubscribe)

### Supported Methods on WS

- `eth_sendRawTransaction`
- `eth_sendRawTransactionSync`
- `eth_subscribe` / `eth_unsubscribe`
- `eth_chainId` (for keepalive)

## Rate Limiting

Public endpoints have rate limits based on:
- CPU time consumption
- Network bandwidth (32 MB/s average, 15x burst for VIP)

### Methods Vulnerable to Abuse

These require VIP endpoints for heavy usage:
- `eth_getLogs` (large block ranges)
- `eth_call` (complex contracts)
- `debug_*` / `trace_*` (disabled on public)

### Best Practices

1. **Don't batch slow with fast** — `eth_getLogs` blocks entire batch response
2. **Use cursors** — `eth_getLogsWithCursor` instead of huge `eth_getLogs`
3. **Background historical queries** — Never block UX waiting for logs
4. **Cache aggressively** — `eth_getTransactionCount` hits CDN cache for same geo

## eth_getLogs Limits

- Multi-block queries: max 20,000 logs per call
- Single-block queries: no limit
- Workaround: Get headers first to count logs, then build queries accordingly

## Historical Data

Public endpoint supports `eth_call` on blocks from past ~15 days. For older data:
- Use Alchemy/QuickNode
- Run your own archive node
- Use indexers (Envio HyperSync recommended)

## Debugging Commands

```bash
# Estimate gas
cast estimate \
  --from 0xYourAddress \
  --value 0.001ether \
  0xRecipient \
  --rpc-url https://mainnet.megaeth.com/rpc

# Get balance at block
cast call --block 12345 0xContract "balanceOf(address)(uint256)" 0xAddress \
  --rpc-url https://mainnet.megaeth.com/rpc

# Trace transaction (requires VIP or local node)
cast run <txhash> --rpc-url <vip-endpoint>
```
