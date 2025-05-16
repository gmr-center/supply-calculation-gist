# Supply Calculation

This repository contains an example implementation of the supply calculation logic used in our public token supply endpoints:

## Live Demo

You can test these API endpoints directly:

<div>
  <a href="https://token-supply.gmr.center/supply/total-supply"><img src="https://img.shields.io/badge/Try%20It-Total%20Supply-FF6B00?style=for-the-badge" alt="Total Supply Demo"/></a>
  <a href="https://token-supply.gmr.center/supply/circulating-supply"><img src="https://img.shields.io/badge/Try%20It-Circulating%20Supply-FF6B00?style=for-the-badge" alt="Circulating Supply Demo"/></a>
</div>

## Deploying

For production implementations, we recommend as a minimum:

- Implementing rate limiting to prevent abuse
- Setting up database caching to reduce RPC calls
- Using paid RPC providers for better reliability and higher rate limits
- Adding monitoring and alerting for supply changes
- Implementing proper error handling and fallback mechanisms
- Setting up automated testing to ensure calculation accuracy

## Code

```typescript
/**
 * ----------------------------------------------------------------------------
 *  GAMER (GMR) · Public Supply-Calculation Snippet
 * ----------------------------------------------------------------------------
 *  • Chain-agnostic: works for any EVM network given an RPC URL
 *  • Precision-safe: *all* math in `bigint`, one `formatUnits` at the end
 *  • Transparent: subtracts burn addresses from total, then team wallets
 *  • No side-effects: no DB writes, logging, or env-specific secrets
 * ----------------------------------------------------------------------------
 */

import { ethers } from 'ethers'

/* -------------------------------------------------------------------------- */
/*  ⚙️  Configuration placeholders – replace with your own RPC URLs           */
/* -------------------------------------------------------------------------- */
const RPC_URLS: Record<string, string> = {
	bnb: 'https://bsc-dataseed.binance.org/', // BNB Smart Chain
	base: 'https://mainnet.base.org', // Base Mainnet
}

/* -------------------------------------------------------------------------- */
/*  Minimal ERC-20 ABI (totalSupply, decimals, balanceOf)                     */
/* -------------------------------------------------------------------------- */
const ERC20_ABI = [
	'function totalSupply() view returns (uint256)',
	'function decimals() view returns (uint8)',
	'function balanceOf(address) view returns (uint256)',
]

/* -------------------------------------------------------------------------- */
/*  Utilities                                                                 */
/* -------------------------------------------------------------------------- */

/** Sum an array of bigint values */
const sumBigInt = (values: bigint[]): bigint =>
	values.reduce((acc, n) => acc + n, 0n)

/* -------------------------------------------------------------------------- */
/*  Main helper                                                               */
/* -------------------------------------------------------------------------- */

/**
 * Fetches **total** and **circulating** supply for an ERC-20 token.
 *
 * @param chain          – key matching `RPC_URLS`
 * @param tokenAddress   – ERC-20 contract address
 * @param burnAddresses  – addresses to *always* subtract (e.g. 0x…dead)
 * @param teamAddresses  – team/treasury wallets to exclude from circulation
 *
 * @returns totalSupply & circulatingSupply as *decimal-string* values
 */
export async function fetchTokenSupplies(
	chain: 'bnb' | 'base', // narrow union keeps TS happy
	tokenAddress: string,
	burnAddresses: string[],
	teamAddresses: string[]
): Promise<{ totalSupply: string; circulatingSupply: string }> {
	/* 1. Initialise provider & contract */
	const provider = new ethers.JsonRpcProvider(RPC_URLS[chain])
	const token = new ethers.Contract(tokenAddress, ERC20_ABI, provider)

	/* 2. Get totalSupply & decimals (parallel) */
	const [rawTotal, rawDecimals] = await Promise.all([
		token.totalSupply() as Promise<bigint>,
		token.decimals().catch(() => 18),
	])

	/* 3. Sum burn + team balances (BigInt) */
	const rawBurned = sumBigInt(
		await Promise.all(
			burnAddresses.map(a => token.balanceOf(a).catch(() => 0n))
		)
	)
	const rawTeam = sumBigInt(
		await Promise.all(
			teamAddresses.map(a => token.balanceOf(a).catch(() => 0n))
		)
	)

	/* 4. Net supplies (still BigInt) */
	const rawAfterBurn = rawTotal - rawBurned
	const rawCirculating = rawAfterBurn - rawTeam

	/* 5. Convert to human-readable strings (ethers trims trailing zeros) */
	return {
		totalSupply: ethers.formatUnits(rawAfterBurn, rawDecimals),
		circulatingSupply: ethers.formatUnits(rawCirculating, rawDecimals),
	}
}

/* -------------------------------------------------------------------------- */
/*  Example usage                                                             */
/* -------------------------------------------------------------------------- */

async function demo() {
	const chain = 'bnb'
	const tokenAddress = '0x168e3b1746aa249a9b3603b70605924fe255ee1a' // GMR
	const burns = [
		'0x000000000000000000000000000000000000dead',
		ethers.ZeroAddress,
	]
	const teams = ['0x123...abc', '0xabc...123']

	const { totalSupply, circulatingSupply } = await fetchTokenSupplies(
		chain,
		tokenAddress,
		burns,
		teams
	)

	console.log('Total Supply      :', totalSupply)
	console.log('Circulating Supply:', circulatingSupply)
}

if (require.main === module) demo()
```
