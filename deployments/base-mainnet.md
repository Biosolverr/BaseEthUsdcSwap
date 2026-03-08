# Base Mainnet Deployment

This document records the canonical deployment of **SecureEthUsdcSwapV3** on Base Mainnet.

All parameters below correspond exactly to the verified on-chain contract and are provided for auditability, reproducibility, and integration reference.

---

## Network

- Network: Base Mainnet
- Chain ID: 8453

---

## Contract

- Name: SecureEthUsdcSwapV3
- Version: v3
- Address:  
  0x32901898b85630eCE7D322Cee992EaF4bF17b19d

---

## Deployment Transaction

- Transaction hash:  
  0xf1d9c513ea3bac3ad8badfd4f721acd330e6498bb72e590e46cb1ca01ecbfc3c
- Block number:  
  43053559

---

## Constructor Parameters

The contract was deployed with the following constructor argument:

- USDC (Base Mainnet):  
  0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913

No other constructor parameters are used.

---

## Verification

- Status: Verified on BaseScan
- Verification method: Solidity Standard JSON Input
- Compiler version: 0.8.26
- Optimization: Enabled (200 runs)
- viaIR: true
- EVM version: cancun

The verified source code and the deployed bytecode are identical.

---

## Notes

- This deployment represents the canonical production instance of SecureEthUsdcSwapV3.
- The contract is non-upgradeable and has no admin or owner privileges.
- All protocol rules are enforced exclusively by the deployed bytecode.
