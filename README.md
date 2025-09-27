# Farcaster Chess Betting Mini App

A Farcaster Mini App that matches two players for on-chain chess with USDC stakes on Base. Players pick one of four tiers (1, 3, 5, or 10 USDC). Matching, move submission, payouts, and fee capture are orchestrated through the `ChessBettingMultiStake` escrow contract.

## Project Structure

```
contracts/
  ChessBettingMultiStake.sol   # Tiered escrow + lobby management
src/
  App.jsx                      # Root Farcaster mini app shell
  main.jsx                     # Vite entrypoint
  index.css                    # Styling (mobile-first)
  components/                  # UI building blocks
  hooks/                       # Lobby, contract, wallet, chess state
  utils/                       # Chess helpers, ABI, lobby polling
```

## Prerequisites

- Node.js 18+
- npm (or pnpm/yarn) for frontend tooling
- An EVM wallet able to sign on Base (Farcaster mini app wallet or browser wallet)
- USDC (Base mainnet) for staking
- Contract deployment tooling (Foundry/Hardhat/Remix) for `ChessBettingMultiStake.sol`

## Getting Started

1. **Install dependencies**
   ```bash
   npm install
   ```

2. **Configure environment**
   Copy `.env.example` → `.env.local` and set `VITE_CONTRACT_ADDRESS` to your deployed escrow contract.

3. **Start the mini app locally**
   ```bash
   npm run dev
   ```
   Visit the exposed URL or load it inside a Farcaster client that supports mini apps.

4. **Build for production**
   ```bash
   npm run build
   npm run preview
   ```

## Contract Deployment

`contracts/ChessBettingMultiStake.sol` targets Solidity `^0.8.20`. Deployment steps (example with Hardhat):

```bash
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox
npx hardhat compile
# Update hardhat.config.ts with Base RPC + deployer key, then:
npx hardhat run scripts/deploy.ts --network base
```

Constructor params:
- `usdcToken`: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` (USDC on Base)
- `feeRecipient_`: platform fee wallet address

After deployment, update `.env.local` with the new contract address.

### Core Functions
- `createLobby(stakeTier)` – deposit & join queue for tier (1/3/5/10)
- `joinLobby(stakeTier)` – match against earliest waiting player
- `switchStake(oldTier, newTier)` – move queue position while rebalancing escrow
- `makeMove(gameId, move)` – record algebraic move on-chain (string payload)
- `claimWin(gameId)` – withdraw winner payout (auto-assigns after 24h inactivity)
- `claimDrawRefund(gameId)` – pull stake back for drawn/cancelled games

Events power the frontend lobby polling + board sync (`LobbyEntered`, `GameStarted`, `MoveSubmitted`, `WinnerDeclared`, `PayoutClaimed`, `DrawRefunded`).

## Frontend Architecture

- **`useContract`** – resolves the Farcaster mini app wallet (or fallback EIP-1193 provider), enforces Base chain, and instantiates read/write contract instances.
- **`useUSDC`** – tracks USDC balance/allowance and auto-approves the escrow as needed.
- **`useLobby`** – orchestrates lobby polling, queue timers, game state, move submission, and payout actions.
- **`useChessGame`** – wraps `chess.js` for local validation, optimistic moves, and remote sync via events.
- **`LobbyManager`** – lightweight polling/event helper to keep lobby counts fresh across tiers.

UI components stay modular:
- `StakeSelector`, `LobbyStatus`, `QueueTimer` handle staking UX.
- `ChessBoard` renders an interactive 8×8 grid with move highlighting.
- `GameInterface` stitches lobby + board state together with Farcaster-friendly controls.

## Testing & Verification Checklist

- **Smart contract**
  - Write Foundry/Hardhat tests covering: lobby queueing, matching order, fee distribution (winner receives 1.8× stake, fee wallet receives 10%), inactivity claims, draw refunds, and `switchStake` fund adjustments.
  - Simulate edge cases: simultaneous joins, cancelling while matched, repeated auto-claim after 24h.

- **Frontend**
  - `npm run build` – confirms Vite + JSX compile.
  - Add component tests (React Testing Library) for stake selection and queue timers if required.
  - When running against a fork/Base testnet, verify:
    - Two wallets at 5 USDC immediately match and start a game.
    - Queue timeout triggers re-queue after 30 seconds.
    - Winner payout equals `stake * 1.8`, platform wallet receives 10% of pot.
    - `switchStake` keeps escrow balances consistent.

## Deployment Notes

- Host the built site on HTTPS with the Farcaster mini app manifest and embed (see [miniapps documentation](https://miniapps.farcaster.xyz/)).
- Ensure CORS allows Farcaster clients.
- Monitor `MoveSubmitted` and `WinnerDeclared` events for analytics or dispute resolution tooling.

## Next Steps

- Integrate an arbiter bot/on-chain verification strategy to call `declareWinner` automatically.
- Add serverless relays to expand move history beyond on-chain events if needed.
- Extend UI with chat/game history and richer analytics per stake tier.
