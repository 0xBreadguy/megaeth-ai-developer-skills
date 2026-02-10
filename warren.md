# Warren Protocol Integration

Warren is MegaETH's on-chain website hosting protocol using SSTORE2 storage.

## Overview

Warren enables fully on-chain websites by storing HTML/CSS/JS directly in smart contract storage. Sites are referenced via NFT ownership (Master NFT or Container NFT).

## Contenthash Format

When integrating Warren with naming services (like MegaNames), use this contenthash encoding:

### Format (7 bytes)

```
[codec 2B] [type 1B] [tokenId 4B]
```

| Field | Bytes | Value | Description |
|-------|-------|-------|-------------|
| **Codec** | 2 | `0xe9` | Warren namespace codec (pending multicodec registration) |
| **Type** | 1 | `0x01` or `0x02` | `0x01` = Master NFT, `0x02` = Container NFT |
| **TokenId** | 4 | uint32 | Warren site token ID (up to ~4.2B sites) |

### Fallback Codec

If the multicodec registration (`0xe9`) isn't approved, use the private-use range:
- Fallback codec: `0x300000` (3 bytes instead of 2, making total 8 bytes)

### Example Encoding

```
Site: bread.mega pointing to Warren Master NFT #1

Contenthash: 0xe9 01 00000001
             ^^^^ ^^ ^^^^^^^^
             codec type tokenId (uint32 big-endian)

Full bytes: 0xe90100000001
```

## Solidity Helpers

### Encoding

```solidity
/// @notice Encode a Warren contenthash
/// @param tokenId The Warren NFT token ID
/// @param isMaster True for Master NFT (0x01), false for Container NFT (0x02)
/// @return contenthash The encoded contenthash bytes
function encodeWarrenContenthash(
    uint32 tokenId,
    bool isMaster
) internal pure returns (bytes memory) {
    return abi.encodePacked(
        uint16(0xe9),           // Warren codec
        isMaster ? uint8(0x01) : uint8(0x02),  // Type
        tokenId                  // Token ID (4 bytes)
    );
}
```

### Decoding

```solidity
/// @notice Decode a Warren contenthash
/// @param contenthash The encoded contenthash bytes
/// @return tokenId The Warren NFT token ID
/// @return isMaster True if Master NFT, false if Container NFT
function decodeWarrenContenthash(
    bytes memory contenthash
) internal pure returns (uint32 tokenId, bool isMaster) {
    require(contenthash.length == 7, "Invalid Warren contenthash length");
    
    // Check codec (first 2 bytes)
    uint16 codec;
    assembly {
        codec := mload(add(contenthash, 2))
    }
    require(codec == 0xe9, "Invalid Warren codec");
    
    // Get type (byte 3)
    uint8 typeFlag = uint8(contenthash[2]);
    require(typeFlag == 0x01 || typeFlag == 0x02, "Invalid Warren type");
    isMaster = typeFlag == 0x01;
    
    // Get tokenId (bytes 4-7)
    assembly {
        tokenId := mload(add(contenthash, 7))
    }
    tokenId = uint32(tokenId >> 224); // Shift to get last 4 bytes
}
```

### Validation

```solidity
/// @notice Check if contenthash is a valid Warren reference
/// @param contenthash The contenthash to validate
/// @return True if valid Warren contenthash
function isWarrenContenthash(bytes memory contenthash) internal pure returns (bool) {
    if (contenthash.length != 7) return false;
    
    // Check Warren codec (0xe9)
    if (uint8(contenthash[0]) != 0x00 || uint8(contenthash[1]) != 0xe9) return false;
    
    // Check valid type
    uint8 typeFlag = uint8(contenthash[2]);
    if (typeFlag != 0x01 && typeFlag != 0x02) return false;
    
    return true;
}
```

## Integration with MegaNames

When a user sets their `.mega` name to point to a Warren site:

```solidity
// User flow
1. User owns bread.mega (MegaNames NFT)
2. User deploys site to Warren, receives tokenId = 42
3. User calls setContenthash on MegaNames:
   
   bytes memory contenthash = encodeWarrenContenthash(42, true);
   megaNames.setContenthash(namehash("bread.mega"), contenthash);

4. Gateway resolves bread.mega → decodes contenthash → fetches from Warren
```

## Gateway Resolution

Gateways (like meganame.market) should:

1. Resolve name → get contenthash
2. Detect Warren codec (`0xe9`)
3. Decode tokenId and type
4. Fetch content from Warren contract using tokenId
5. Serve the on-chain website

## Contract Addresses

| Network | Contract | Address |
|---------|----------|---------|
| MegaETH Mainnet | Warren | TBD |
| MegaETH Testnet | Warren | TBD |

## Resources

- [Warren Protocol Docs](https://docs.megaeth.com/warren) (TBD)
- [MegaNames Integration](https://github.com/0xBreadguy/mega-names)
- [Multicodec Registry](https://github.com/multiformats/multicodec)
