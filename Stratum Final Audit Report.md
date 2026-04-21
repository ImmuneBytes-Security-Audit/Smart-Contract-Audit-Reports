# **Stratum Protocol** **Final Smart Contract Report** {#stratum-protocol-final-smart-contract-report}

[**Stratum Protocol**](#stratum-protocol-final-smart-contract-report)  
[**Final Smart Contract Report	1**](#stratum-protocol-final-smart-contract-report)

[**Project Details	2**](#project-details)

[**Executive Summary	3**](#executive-summary)

[**Scope	4**](#scope)

[**System Overview	5**](#system-overview)

[**Trust Assumptions	6**](#trust-assumptions)

[**Engagement Goals	7**](#engagement-goals)

[**Security Concerns	8**](#security-concerns)

[**Methodology	10**](#methodology)

[**Findings	11**](#findings)

[High Severity Issues	12](#high-severity-issues)

[Medium Severity Issues	38](#medium-severity-issues)

[Low Severity Issues	43](#low-severity-issues)

[Informational	50](#informational)

[**Disclaimer	54**](#disclaimer)

# **Project Details** {#project-details}

Name:				Stratum Protocol  
Blockchain:			EVM Compatible (Ethereum Mainnet, Layer 2s)  
Branch:			`fix-subscribe-stablecoin-decimals`  
Solidity:			`^0.8.25` · Framework: Foundry  
Timeline:			March 30 – April 4, 2026  
Auditors:			Muhammad Jariruddin & Kaif Ahmed  
Report Type:			Final Audit Report \- April 17, 2026

# 

# **Executive Summary** {#executive-summary}

ImmuneBytes performed the security audit of Stratum Protocol's tokenized trade-finance smart contracts during March and April 2026\. The assessment covered the complete five-contract system — `StratumAccessControl`, `TokenFactory`, `SubscriptionContract`, `RedemptionContract`, and `TradeToken` — as a cohesive on-chain investment platform enabling tokenized real-world trade receivables, structured investor subscriptions, and settlement-based redemption. The protocol demonstrates sound architectural separation through UUPS upgradeability, ERC-7201 namespaced storage, a clearly defined multi-role access control hierarchy, and a structured trade lifecycle with distinct phases for fundraising, capital deployment, and investor redemption. However, we identified six High and two Medium vulnerabilities concentrated on the fund-flow paths that govern investor capital and settlement arithmetic. Chief among these are: a settlement denominator that includes unsold token supply, silently routing investor profits to the protocol as unearned revenue; an unoverridden `ERC20Burnable.burnFrom()` path that defeats burn restrictions, compliance blacklisting, and subscription accounting simultaneously; a precision-truncation zero-cost token acquisition vector viable at realistic prices on low-gas Layer 2 networks; and protocol-wide incompatibility with USDT on Ethereum mainnet due to missing `SafeERC20` semantics across all eleven fund-transfer sites. Additional concerns include a single dust deposit capable of permanently locking the redemption rate and unconditionally closing the funding phase, a mempool-visible settlement transaction exploitable for risk-free front-running arbitrage, permissionless O(n) request queue spam capable of halting token approvals via gas exhaustion, and the absence of any on-chain mechanism to recover stablecoins transferred directly to protocol contracts. Collectively these issues create credible paths to irreversible investor capital loss, compliance enforcement bypass, and protocol-wide operational denial of service.

On balance, the protocol is recoverable with targeted, low-complexity changes that are straightforward to implement and verify. We recommend: computing `redemptionValuePerToken` against circulating supply rather than `totalSupply`; overriding `burnFrom()` to revert in `TradeToken`; adding a zero-cost guard to `subscribe()`; applying `SafeERC20.safeTransfer` and `safeTransferFrom` across all eleven transfer sites; enforcing a minimum settlement floor relative to `totalRaised`; requiring `fundingClosed` before `depositSettlement` is accepted; restricting `requestTradeToken` to a designated role; and adding a role-gated rescue function to each contract. The audit produced a 25-test Foundry proof-of-concept suite that verified all material findings on-chain against the live contract logic. Expanding existing test coverage to include adversarial settlement scenarios, non-standard ERC-20 semantics, and permissionless queue growth will materially raise deployment assurance before production.

# **Scope** {#scope}

In Scope

- `src/access/StratumAccessControl.sol`  
- `src/factory/TokenFactory.sol`  
- `src/funding/SubscriptionContract.sol`  
- `src/redemption/RedemptionContract.sol`  
- `src/tokens/TradeToken.sol`  
- `src/storage/*.sol`  
- `src/interfaces/*.sol`  
- `src/types/Types.sol`

Out of Scope

- `src/mocks/`  
- `script/`  
- `test/`  
- Deployment scripts  
- Any offchain component assumed to be trusted and controlled centrally.

# 

# **System Overview** {#system-overview}

Stratum is a tokenized trade-finance protocol that represents real-world short-term trade receivables — invoices, purchase orders, and factoring facilities — as on-chain ERC-20 tokens. The protocol enables a trader to tokenize a trade asset, investors to subscribe by paying stablecoins in exchange for trade tokens, the trader to withdraw the raised capital for deployment in the underlying trade, and — after trade completion off-chain — to settle by depositing stablecoins back into the protocol, from which investors redeem their tokens at a calculated per-token rate.

Architecture

The system is composed of five primary smart contracts. `StratumAccessControl` serves as the root of trust and single source of authority for all role assignments, cross-contract address resolution, and fee accumulation across the protocol. Every other contract holds a direct reference to it and delegates all authorization checks through it, making it the singular point whose compromise would affect the entire system simultaneously. `TokenFactory` manages the two-step lifecycle for trade token creation: an open request phase followed by an `OPERATIONS_ADMIN`\-gated approval that deploys a new `TradeToken` instance. `SubscriptionContract` handles investor-facing token purchases, stablecoin escrow, and trader fund withdrawal. `RedemptionContract` manages the settlement deposit, confirmation workflow, redemption window, and investor payout. `TradeToken` is a non-upgradeable per-trade ERC-20 whose parameters — price, stablecoin, total supply, decimals — are all set as immutable fields at construction.

All upgradeable contracts (`StratumAccessControl`, `TokenFactory`, `SubscriptionContract`, `RedemptionContract`) use the UUPS (ERC-1967) proxy pattern and store state via ERC-7201 namespaced storage slots to prevent storage collisions across upgrades. Upgrade authorization on all four contracts is ultimately delegated to `DEFAULT_ADMIN_ROLE` held by `StratumAccessControl`.

Trade Lifecycle

A trade token is requested by any caller and approved by `OPERATIONS_ADMIN`, which deploys the `TradeToken` with its full supply minted directly to `SubscriptionContract`. Investors call `subscribe()` to pay stablecoins and receive trade tokens; fees are collected to `StratumAccessControl` at this point. The trader calls `withdrawInvestorFunds()` to extract raised capital for deployment in the underlying trade. After trade completion, the trader calls `depositSettlement()`, which locks `redemptionValuePerToken` for the trade and unconditionally triggers `closeFundingOnSettlement()`. `OPERATIONS_ADMIN` then calls `confirmSettlement()` and `openRedemption()` to open the investor redemption window. Investors call `redeem()` to burn their trade tokens in exchange for stablecoins at the locked rate. After the redemption deadline, `OPERATIONS_ADMIN` calls `handleUnredeemedTokens()` to sweep any remaining settlement balance to `StratumAccessControl` as protocol revenue.

Key Implementation Details

`TradeToken.decimals()` returns `TOKEN_DECIMALS`, which is set to the linked stablecoin's decimal precision at deployment — the fix introduced in the branch under audit. This value is used as the divisor in `subscribe()`'s cost calculation. Transfer restrictions on `TradeToken` perform blacklist checks against both a token-level blacklist stored locally and a protocol-level blacklist via a cross-call to `StratumAccessControl`. The `requestTradeToken` function is open to any caller; the `requestId` is caller-supplied and must be globally unique within the factory's history. Pending request IDs are stored in a dynamic array with O(n) linear-scan removal.

# **Trust Assumptions** {#trust-assumptions}

- `DEFAULT_ADMIN_ROLE` is fully trusted and treated as protocol owner; findings requiring its malicious exercise are outside scope.  
- `SUPER_ADMIN_ROLE`, `OPERATIONS_ADMIN_ROLE`, and `COMPLIANCE_ADMIN_ROLE` are fully trusted in their designated domains; findings requiring malicious exercise of their explicitly granted authority are treated as Informational.  
- The trader is semi-trusted: off-chain legal agreements provide the only enforcement mechanism. The trader's on-chain behavior, including the settlement amount deposited, is not bounded by any protocol invariant. Findings arising from trader adversarial action are in scope.  
- Stablecoin issuers (USDC, USDT) are trusted ERC-20 implementations; their contracts are not considered adversarial inputs.  
- The off-chain backend system supplying `requestId` values is trusted for content validity; on-chain, `requestId` uniqueness is the only enforced property.  
- Investors hold no privileged role; they are purely protocol users with no governance rights and no implicit trust assumption.  
- No oracle or external price feed dependency exists in the protocol; price discovery is entirely off-chain.

# 

# **Engagement Goals** {#engagement-goals}

1. If StratumAccessControl is the root of all authority, what actually remains secure if it is upgraded, misconfigured, or adversarially controlled? Or does the entire protocol trivially collapse by design?  
2. Given the trader has zero on-chain obligation to deposit a fair settlement, is investor return ever enforced — or is “investment” just unsecured exposure wrapped in ERC-20 UX?  
3. While funds sit in SubscriptionContract, are they meaningfully protected — or can they be extracted early, upgraded away, or redirected before the lifecycle completes?  
4. Since lifecycle progression depends on discrete admin actions (approve → confirm → open redemption), what guarantees the system cannot halt mid-state and strand all capital indefinitely?  
5. Does depositSettlement actually represent investor profit distribution — or just a free-form number that can irreversibly define outcomes regardless of fairness?  
6. Why is settlement computed over total supply (including undistributed tokens), and does this imply that part of investor returns is structurally redirected away from actual participants?  
7. Do token holders have a guaranteed right to redeem — or is redemption merely a conditional window that can be delayed, shortened, or never meaningfully opened?  
8. Is unredeemed settlement fundamentally user-owned capital — or intentionally designed protocol revenue extracted through timing and user inaction?  
9. Do investors actually hold enforceable financial claims — or just transferable tokens whose value depends entirely on admin actions and trader honesty?  
10. Does blacklisting enforce regulation — or function as an irreversible fund-freezing mechanism with no recovery path?  
11. Are fees economically bounded in practice, or can the system legally extract extreme value through configurable rates without violating any invariant?  
12. Are upgrades constrained to preserve system invariants, or can they arbitrarily redefine balances, flows, and permissions post-deployment?

# 

# **Security Concerns** {#security-concerns}

Settlement value integrity

The settlement value formula in `depositSettlement` computes `redemptionValuePerToken` over the full token `totalSupply`, which includes unsold tokens held by `SubscriptionContract` that no investor can ever redeem. In any partially subscribed round, this inflates the denominator and systematically underpays every investor. A separate but compound concern is that `depositSettlement` accepts any non-zero amount with no minimum floor: a 1-wei deposit sets a near-zero rate, unconditionally closes the funding phase, and locks that garbage rate permanently with no on-chain correction path.

Token burn and access control

`TradeToken` overrides `ERC20Burnable.burn()` to restrict callers to `REDEMPTION_CONTRACT`, but does not override `burnFrom()`. Any token holder can call the inherited `burnFrom()` directly, bypassing the restriction entirely. This single omission defeats compliance blacklisting enforcement, corrupts the `tokensDistributed` accounting in `SubscriptionContract`, and enables a permanent subscription denial of service by reducing `totalSupply` below the recorded distribution ceiling.

Subscription cost arithmetic

The token cost calculation in `subscribe()` performs integer division before scaling, which truncates to zero for small token amounts at sub-dollar prices. On low-gas Layer 2 networks where sub-dollar token prices are commercially viable, any caller can acquire trade tokens with no stablecoin payment, bypassing the protocol's entire economic model.

ERC-20 implementation compatibility

All eleven stablecoin `transfer` and `transferFrom` call sites across the protocol use the raw IERC20 interface and assume a boolean return value. USDT on Ethereum mainnet does not return a value from these functions. A failed USDT transfer will silently succeed at the call site with no revert, causing the protocol to record fund movements that never occurred. The absence of `SafeERC20` across all fund-flow paths makes the protocol non-deployable with USDT on mainnet.

Mempool front-running exposure

`depositSettlement` is submitted as a standard transaction visible in the public mempool before execution. Any observer who detects the pending call can immediately subscribe to the trade token at the current price in the same block window. Because settlement fixes the redemption rate above the subscription price, the attacker is guaranteed a profit equal to the settlement return with zero trade-finance risk — the entire purpose of the subscription being gated to the pre-settlement phase.

Protocol denial of service

`requestTradeToken` is open to any caller and stores all pending requests in a dynamic array. `approveTradeToken` scans this array with O(n) complexity to locate and remove the approved entry. An unauthenticated attacker can fill the pending queue with arbitrary requests, growing the array to a size at which `approveTradeToken` exhausts the block gas limit and can no longer be called. Legitimate trade token creation is halted for the entire protocol until the queue is administratively drained, which has no on-chain implementation.

Fund recovery and interaction ordering

No contract in the protocol exposes a role-gated rescue or sweep function for stablecoins or trade tokens sent to a contract address outside of the intended entry points. Any such transfer is permanently locked. Separately, `depositSettlement` and `handleUnredeemedTokens` perform external token transfers before updating internal accounting state, violating Checks-Effects-Interactions ordering and creating a latent reentrancy surface that will become exploitable if a non-standard stablecoin with transfer callbacks is ever added.

Fee policy consistency

`MAX_FEE_RATE` is set to 10,000 basis points (100%) in the deployed contract while all NatSpec comments across the three fee setter functions document a maximum of 1,000 basis points (10%). The on-chain constant is the enforced value; the documented ceiling is ignored. Any fee setter can assign fees up to ten times the stated maximum with no on-chain violation.

# 

# **Methodology** {#methodology}

This audit was conducted through manual line-by-line review of all in-scope contracts with particular focus on fund-flow correctness, access control completeness, and arithmetic precision. The review was structured across multiple analytical passes: an initial architecture review to establish the full call graph and state model, a per-function deep analysis using the Trail of Bits audit-context-building methodology to trace all external call chains and callback surfaces, dimensional analysis to verify decimal precision alignment across all arithmetic paths, and a Semgrep static analysis scan to surface common vulnerability patterns across the Solidity codebase. A sharp-edges review examined all inherited public and external functions for override completeness. All material findings were verified with a dedicated Foundry proof-of-concept test suite, comprising 25 tests covering end-to-end fund flows, exploit paths, and economic scenarios.

# 

# **Findings**  {#findings}

| ID | Title | Severity | Status |
| ----- | ----- | :---: | :---: |
| H-01 | Incorrect Redemption Supply Denominator | **High** | **FIXED** |
| H-02 | Unrestricted Token Burn Path | **High** | **FIXED** |
| H-03 | Zero-Cost Subscription Acquisition | **High** | **FIXED** |
| H-04 | Unsafe ERC-20 Transfer Pattern | **High** | **FIXED** |
| H-05 | Dust Settlement Rate Lock | **High** | **PARTIALLY FIXED** |
| H-06 | Settlement Deposit Front-Running | **High** | **FIXED** |
| M-01 | Permissionless Request Queue Griefing | **Medium** | **FIXED** |
| M-02 | Irrecoverable Direct Transfer Funds | **Medium** | **ACKNOWLEDGED** |
| L-01 | CEI Ordering Pattern Violation | **Low** | **FIXED** |
| L-02 | Fee Rate Constant Mismatch | **Low** | **FIXED** |
| L-03 | Inline Role Hash Computation | **Low** | **FIXED** |
| L-04 | Redemption Rounding Remainder | **Low** | **ACKNOWLEDGED** |
| L-05 | Missing Redemption Finalization Guard | **Low** | **ACKNOWLEDGED** |
| I-01 | Silent Decimal Fallback | **Informational** | **FIXED** |
| I-02 | Premature Investor Fund Withdrawal | **Informational** | **ACKNOWLEDGED** |
| I-03 | Redundant Restriction Check Calls | **Informational** | **FIXED** |
| I-04 | Non-Unique TradeToken Symbol | **Informational** | **FIXED** |
| I-05 | Inverted Redemption Error Name | **Informational** | **FIXED** |
| I-06 | Absent Minimum Redemption Deadline | **Informational** | **ACKNOWLEDGED** |
| I-07 | Absent On-Chain Token Whitelist | **Informational** | **ACKNOWLEDGED** |

## **High Severity Issues** {#high-severity-issues}

1. **Incorrect Redemption Supply Denominator**  
   Severity: High   
   Status: **Fixed**  
   Location: `src/redemption/RedemptionContract.sol:94–99`   
   Category: Arithmetic / Economic Logic

   **Description**  
   `depositSettlement` computes `redemptionValuePerToken` by dividing the settlement amount by `IERC20(tradeToken).totalSupply()`:  
     
   // RedemptionContract.sol:94–99  
     
   uint256 totalSupply \= IERC20(tradeToken).totalSupply();  
     
   uint256 redemptionValuePerToken \= (settlementAmount \* 1e18) / totalSupply;  
     
   `totalSupply()` includes all tokens minted at factory creation — including the portion still held in `SubscriptionContract` that was never sold to any investor. In any partial or expired subscription round, a share of the supply remains unowned by any participant who can redeem. The per-token rate is computed over a denominator larger than the set of tokens that can actually claim settlement, systematically underpaying every investor.  
     
   After all valid redemptions are processed, the stablecoins corresponding to unsold tokens remain in `settlementPools`. These are swept to `StratumAccessControl` as protocol revenue by `handleUnredeemedTokens()`. No mechanism burns or excludes unsold tokens before settlement is computed, and no protocol documentation frames this surplus transfer as an intended design outcome. The pre-review threat model (AUDIT\_PRE\_REVIEW.md, TOK-05) flags this calculation as an open concern.

   **Impact**  
   Investor Yield Understatement  
     
- In every partial subscription, investors receive a proportionally reduced settlement payout. A trade returning 2× capital on investor funds distributes only 1× if 50% of tokens were unsold.  
- `redemptionValuePerToken` is immutable once set; there is no corrective path after deposit.  
    
    
  Unearned Protocol Revenue  
    
- The settlement share corresponding to unsold tokens flows to `StratumAccessControl` via `handleUnredeemedTokens()`, inflating protocol fees with capital that belongs economically to investors.  
    
  Systematic Scope  
    
- The defect is not limited to edge cases; it applies to every trade where the subscription round does not reach 100% of token supply. Partial subscriptions are expected under any normal demand distribution.

  **Proof of Concept**  
  **File:** `test_PartialSubscription_DilutesRedemption`

| function test\_PartialSubscription\_DilutesRedemption() public {     // 1000 tokens at $1 each     TradeToken token \= \_createToken(1000e6, 1e6);     // Only 500 tokens sold (partial subscription)     stablecoin.mint(ALICE, 500e6);     vm.startPrank(ALICE);     stablecoin.approve(address(subscriptionContract), type(uint256).max);     subscriptionContract.subscribe(address(token), 500e6);     vm.stopPrank();     // Admin closes funding (only 50% sold)     vm.prank(OPERATIONS\_ADMIN);     subscriptionContract.closeFunding(address(token));     // Trader deposits $1000 settlement (trade doubled the money)     stablecoin.mint(TRADER, 1000e6);     vm.startPrank(TRADER);     stablecoin.approve(address(redemptionContract), type(uint256).max);     redemptionContract.depositSettlement(address(token), 1000e6);     vm.stopPrank();     // redemptionValuePerToken is based on totalSupply (1000), not sold (500)     uint256 redemptionValue \= redemptionContract.getRedemptionValue(address(token));     // Expected if correct: 1000e6 \* 1e18 / 500e6 \= 2e18 ($2/token)     // Actual: 1000e6 \* 1e18 / 1000e6 \= 1e18 ($1/token)     vm.startPrank(OPERATIONS\_ADMIN);     redemptionContract.confirmSettlement(address(token));     redemptionContract.openRedemption(address(token), 30 days);     vm.stopPrank();     // Alice redeems all 500 tokens     vm.startPrank(ALICE);     token.approve(address(redemptionContract), type(uint256).max);     redemptionContract.redeem(address(token), 500e6);     vm.stopPrank();     uint256 aliceReceived \= stablecoin.balanceOf(ALICE);     uint256 remainingInPool \= redemptionContract.getSettlementPool(address(token));     // Alice paid $500, trade doubled to $1000, but Alice only gets $500 back     assertEq(aliceReceived, 500e6, "Alice only gets $500 (break even, no profit)");     assertEq(remainingInPool, 500e6, "$500 stuck for unsold tokens");     // After deadline, protocol sweeps the $500     vm.warp(block.timestamp \+ 31 days);     vm.prank(OPERATIONS\_ADMIN);     redemptionContract.handleUnredeemedTokens(address(token));     uint256 protocolGot \= stablecoin.balanceOf(address(accessControl));     assertEq(protocolGot, 500e6, "Protocol takes $500 that should be Alice's profit"); } |
| :---- |


  

  **Result:** Alice invested $500, trade doubled to $1000, but she only receives $500 back (0% return). The protocol sweeps the other $500 as "unredeemed" fees.


  **Additional PoC:** `test_ExpiredFunding_Settlement_Dilution` — same dilution triggered via expired funding deadline (700 of 1000 tokens sold, 30% dilution confirmed).


  **Recommendations**

- Exclude unsold tokens from the settlement denominator by computing circulating supply at deposit time:


  uint256 subscriptionBal \= IERC20(tradeToken).balanceOf(subscriptionContract);


  uint256 circulatingSupply \= totalSupply \- subscriptionBal;


  uint256 redemptionValuePerToken \= (settlementAmount \* 1e18) / circulatingSupply;


- Revert if `circulatingSupply == 0` to prevent division-by-zero when no tokens were sold.  
- Consider burning unsold tokens in `closeFunding()` or `closeFundingOnSettlement()` so `totalSupply()` reflects only investor-held tokens at settlement time, removing the need for the balance subtraction.

### 

2. **Unrestricted Token Burn Path**  
   Severity: High   
   Status: **Fixed**   
   Location: `src/tokens/TradeToken.sol:114–118` · `src/funding/SubscriptionContract.sol:93–96`   
   Category: Access Control / Inherited Function Override

   **Description**  
   `TradeToken` overrides `ERC20Burnable.burn()` to restrict burning to `REDEMPTION_CONTRACT`:  
     
   // TradeToken.sol:114–118  
     
   function burn(uint256 amount) public override {  
     
       if (msg.sender \!= REDEMPTION\_CONTRACT) {  
     
           revert ITradeToken.TradeToken\_UnauthorizedBurner();  
     
       }  
     
       \_burn(msg.sender, amount);  
     
   }  
     
   `ERC20Burnable.burnFrom(address account, uint256 amount)` is never overridden. `burnFrom` is a public function on the inherited interface. Any token holder can call `approve(address(this), amount)` followed by `burnFrom(address(this), amount)` to burn their own tokens, bypassing the `REDEMPTION_CONTRACT` restriction entirely.  
     
   The `_burn` path invoked by `burnFrom` calls `_update()` directly. `TradeToken` does not override `_update()` to invoke `_checkTransferRestrictions`, so the burn path bypasses blacklist enforcement at `TradeToken.sol:161–174`. Three independent downstream effects follow from uncontrolled burn access.  
     
   Effect 1 — Supply distortion: `redemptionValuePerToken` in `depositSettlement` divides by `totalSupply()`. Burning tokens before settlement inflates the rate for remaining investors, but the burned investor receives nothing. If burning occurs after settlement, the distortion is gone but the burned investor has permanently forfeited their claim.  
     
   Effect 2 — Subscription underflow: `SubscriptionContract.getAvailableTokens()` computes `totalSupply - tokensDistributed`. `tokensDistributed` only increments; it never decrements on burn. After `burnFrom` reduces `totalSupply` such that `totalSupply < tokensDistributed`, `getAvailableTokens()` reverts with an arithmetic underflow, permanently halting new subscriptions for that trade token.  
     
   Effect 3 — Blacklist bypass: Blacklisted token holders can call `burnFrom` to destroy their tokens. The burn path does not invoke `_checkTransferRestrictions`, so `TradeToken_ProtocolBlacklisted` is never raised. Compliance enforcement via blacklisting can be trivially circumvented.

   **Impact**  
   Access Control Bypass  
     
- The `REDEMPTION_CONTRACT` burn restriction on `burn()` is trivially circumvented by any token holder using the unoverridden `burnFrom()` entry point.  
- Blacklisted addresses can destroy tokens without triggering compliance controls.  
    
  Subscription System Denial of Service  
    
- Once `burnFrom` reduces `totalSupply` below `tokensDistributed`, `getAvailableTokens()` reverts permanently for that trade token. New subscriptions are halted with no recovery path short of a contract upgrade.  
    
  Phantom Token Lock  
    
- Tokens burned directly from the `SubscriptionContract` balance do not reduce `tokensDistributed`. These phantom entries are permanently locked — they cannot be subscribed, transferred, or recovered.

    
  **Proof of Concept**  
  **Four tests covering each attack vector:**  
    
  **PoC 1 — Burn restriction bypass** (`test_BurnFrom_Bypass`):

| function test\_BurnFrom\_Bypass() public {     TradeToken token \= \_createToken(1000e6, 1e6);     stablecoin.mint(ALICE, 500e6);     vm.startPrank(ALICE);     stablecoin.approve(address(subscriptionContract), type(uint256).max);     subscriptionContract.subscribe(address(token), 500e6);     // burn() correctly reverts     vm.expectRevert();     token.burn(100e6);     // burnFrom() bypasses the restriction entirely     token.approve(ALICE, 500e6);     token.burnFrom(ALICE, 200e6); // NO REVERT     vm.stopPrank();     assertEq(token.balanceOf(ALICE), 300e6, "Alice has 300 tokens left");     assertEq(token.totalSupply(), 800e6, "TotalSupply reduced to 800"); } |
| :---- |


  **PoC 2 — Blacklist bypass** (`test_BurnFrom_Bypasses_Blacklist`):

| function test\_BurnFrom\_Bypasses\_Blacklist() public {     TradeToken token \= \_createToken(1000e6, 1e6);     stablecoin.mint(ALICE, 500e6);     vm.startPrank(ALICE);     stablecoin.approve(address(subscriptionContract), type(uint256).max);     subscriptionContract.subscribe(address(token), 500e6);     token.approve(ALICE, 500e6);     vm.stopPrank();     // Blacklist Alice     vm.prank(COMPLIANCE\_ADMIN);     accessControl.blacklistAddress(ALICE);     vm.startPrank(ALICE);     vm.expectRevert(); // transfer blocked     token.transfer(BOB, 100e6);     token.burnFrom(ALICE, 200e6); // SUCCEEDS despite blacklist     vm.stopPrank();     assertEq(token.balanceOf(ALICE), 300e6); } |
| :---- |


  

  **PoC 3 — Subscription DOS** (`test_BurnFrom_DOS_Subscriptions`):

| function test\_BurnFrom\_DOS\_Subscriptions() public {     TradeToken token \= \_createToken(1000e6, 1e6);     stablecoin.mint(ALICE, 600e6);     vm.startPrank(ALICE);     stablecoin.approve(address(subscriptionContract), type(uint256).max);     subscriptionContract.subscribe(address(token), 600e6);     token.approve(ALICE, 200e6);     token.burnFrom(ALICE, 200e6);     vm.stopPrank();     uint256 availableTokens \= subscriptionContract.getAvailableTokens(address(token));     uint256 contractBalance \= token.balanceOf(address(subscriptionContract));     assertEq(contractBalance, 400e6, "Contract holds 400");     assertEq(availableTokens, 200e6, "Only 200 available — 200 phantom locked");     // Bob cannot subscribe for more than 200 even though contract holds 400     stablecoin.mint(BOB, 400e6);     vm.startPrank(BOB);     stablecoin.approve(address(subscriptionContract), type(uint256).max);     vm.expectRevert();     subscriptionContract.subscribe(address(token), 201e6);     subscriptionContract.subscribe(address(token), 200e6); // max allowed     vm.stopPrank(); } |
| :---- |


  


  **PoC 4 — Underflow DOS** (`test_GetAvailableTokens_Reverts_After_BurnFrom`):

| function test\_GetAvailableTokens\_Reverts\_After\_BurnFrom() public {     TradeToken token \= \_createToken(1000e6, 1e6);     stablecoin.mint(ALICE, 1000e6);     vm.startPrank(ALICE);     stablecoin.approve(address(subscriptionContract), type(uint256).max);     subscriptionContract.subscribe(address(token), 1000e6);     // Burn 1 token: totalSupply \= 999e6, tokensDistributed \= 1000e6     token.approve(ALICE, 1e6);     token.burnFrom(ALICE, 1e6);     vm.stopPrank();     // getAvailableTokens underflows: 999e6 \- 1000e6     vm.expectRevert(); // arithmetic underflow     subscriptionContract.getAvailableTokens(address(token)); } |
| :---- |


  

  **Recommendations**

- Override `burnFrom` in `TradeToken` to unconditionally revert:


  function burnFrom(address, uint256) public pure override {


      revert ITradeToken.TradeToken\_UnauthorizedBurner();


  }


- Override `_update()` in `TradeToken` to invoke `_checkTransferRestrictions` on all calls including burns, ensuring blacklist enforcement cannot be bypassed through any code path.  
- Cross-remediate with H-01: after implementing the `burnFrom` fix, confirm that `getAvailableTokens()` arithmetic is consistent with the corrected supply denominator.

### 

3. **Zero-Cost Subscription Acquisition**  
   Severity: High   
   Status: **Fixed**   
   Location: `src/funding/SubscriptionContract.sol:101–102`   
   Category: Arithmetic / Missing Validation

   **Description**  
   `subscribe()` computes the stablecoin cost as:  
     
   // SubscriptionContract.sol:101–102  
     
   uint256 requiredStablecoinAmount \=  
     
       (tokenAmount \* tokenPrice) / (10 \*\* uint256(IERC20Metadata(tradeToken).decimals()));  
     
   No guard checks `requiredStablecoinAmount > 0`. When `tokenAmount * tokenPrice < 10^decimals`, integer division truncates to zero. `subscribe()` proceeds to transfer `tokenAmount` tokens to the caller at zero cost: `fee = 0`, `netAmount = 0`, `totalRaised += 0`.  
     
   The trigger window is not limited to pathological configurations. For USDC (6 decimals) at `tokenPrice = 5e5` ($0.50 per token): `1 * 5e5 / 1e6 = 0`. Any `tokenAmount` below 2 acquires tokens free. A loop over `subscribe(tradeToken, 1)` acquires the entire token supply without paying any stablecoin. On L2 networks (Arbitrum, Base, Optimism) where gas costs are negligible, acquiring the full supply at zero capital cost is economically viable. The decimal-fix in this branch adjusts the denominator but does not add the missing zero-cost guard — the defect is present after the branch's changes.

   **Impact**  
   Zero-Cost Token Acquisition  
     
- At any token price below $1 per smallest stablecoin unit, attackers acquire tokens with no capital outlay.  
- Looping the call on a low-gas chain costs a few dollars while draining the full token supply.  
    
  Settlement Pool Dilution  
    
- Zero-cost tokens share pro-rata in settlement redemption, reducing payout to honest investors who paid full price.  
    
    
  Accounting Insolvency  
    
- `totalRaised` understates actual capital; `withdrawInvestorFunds` transfers based on `totalRaised`, which becomes overstated relative to the actual USDC balance. The shortfall creates permanent insolvency.

  **Proof of Concept**  
  **File:** `test_F1_ZeroCostTokens_RealisticPrice_50Cents`

| function test\_F1\_ZeroCostTokens\_RealisticPrice\_50Cents() public {     // $0.50 per token \-- a perfectly reasonable price     TradeToken token \= \_createToken(1000e6, 500000);     stablecoin.mint(ATTACKER, 1e6);     vm.startPrank(ATTACKER);     stablecoin.approve(address(subscriptionContract), type(uint256).max);     // tokenAmount \= 1 (smallest unit). cost \= (1 \* 500000\) / 1e6 \= 0     uint256 freeTokensTotal \= 0;     for (uint256 i \= 0; i \< 100; i++) {         subscriptionContract.subscribe(address(token), 1);         freeTokensTotal \+= 1;     }     vm.stopPrank();     uint256 stablecoinSpent \= 1e6 \- stablecoin.balanceOf(ATTACKER);     assertEq(freeTokensTotal, 100);     assertEq(stablecoinSpent, 0, "Attacker paid nothing"); } |
| :---- |


  **Result:** 100 token units acquired at $0.50/token, 0 USDC paid. Repeat until supply is exhausted.


  **Additional PoCs:**


- `test_F1_ZeroCostTokens_RealisticPrice_10Cents` — 9 free token units at $0.10/token  
- `test_POC_ZeroCostTokenAcquisition` — 999,999 free tokens at `tokenPrice=1`

  **Recommendations**  
- Add a zero-cost guard immediately after computing `requiredStablecoinAmount`:


  if (requiredStablecoinAmount \== 0\) {


      revert SubscriptionContract\_ZeroAmount();


  }


- Consider enforcing a minimum `tokenAmount` floor (`tokenAmount >= minTokenSubscription`) to prevent sub-unit subscriptions at any price point.

### 

4. **Unsafe ERC-20 Transfer Pattern**  
   Severity: High   
   Status: **Fixed**   
   Location: `StratumAccessControl.sol:261,299` · `SubscriptionContract.sol:111,116,122,302,308` · `RedemptionContract.sol:90,290,296`   
   Category: ERC-20 Compatibility / Interface Mismatch

   **Description**  
   All stablecoin transfers across the protocol use the raw `IERC20` interface with explicit boolean return checks:  
     
   // Representative pattern — StratumAccessControl.sol:261  
     
   if (\!IERC20(stablecoin).transfer(treasury, amount)) {  
     
       revert StratumAccessControl\_TransferFailed();  
     
   }  
     
   USDT on Ethereum mainnet (`0xdAC17F958D2ee523a2206206994597C13D831ec7`) does not return a `bool` from `transfer()` or `transferFrom()`. The Solidity ABI decoder, upon receiving empty return data from a call declared as returning `(bool)`, reverts with a decoding error. All eleven transfer call sites listed above become non-functional when invoked against USDT on mainnet.  
     
   The documentation explicitly identifies USDT as a supported stablecoin. The branch name `fix-subscribe-stablecoin-decimals` reflects prior USDT-specific decimal work. Tests use `MockUSDT`, a custom ERC-20 that correctly returns `bool` — masking the incompatibility across the entire test suite. The defect would not surface until the protocol is deployed against live USDT.  
     
   Fee-on-transfer tokens present a secondary concern: the recorded transfer amount differs from the amount actually received. `totalRaised`, `settlementPools`, and fee accounting all assume the recorded value equals the received value. With fee-on-transfer stablecoins, the contract accrues liabilities exceeding actual holdings, producing an insolvency that grows with each operation.

     
   **Impact**  
   Mainnet USDT Non-Functionality  
     
- Every fund-moving operation (`subscribe`, `withdrawInvestorFunds`, `depositSettlement`, `redeem`, `withdrawFees`, `handleUnredeemedTokens`) reverts against USDT on Ethereum mainnet. The protocol is entirely non-functional with its documented primary stablecoin.  
    
  Fee-on-Transfer Accounting Insolvency  
    
- Each subscription or redemption with a fee-on-transfer token creates an accounting deficit that accumulates. The last participant to execute an outbound transfer in the accounting cycle absorbs all accumulated losses.  
    
  Silent Test Coverage Gap  
    
- `MockUSDT` masking means the defect is invisible in the existing 269-test suite; it would only surface at deployment.  
    
  **Code Affected:**

| File | Line | Function |
| :---- | :---- | :---- |
| `StratumAccessControl.sol` | 261 | `withdrawFees` |
| `StratumAccessControl.sol` | 299 | `withdrawAllFees` |
| `SubscriptionContract.sol` | 111 | `subscribe` (transferFrom) |
| `SubscriptionContract.sol` | 116 | `subscribe` (fee transfer) |
| `SubscriptionContract.sol` | 122 | `subscribe` (token transfer) |
| `SubscriptionContract.sol` | 302 | `withdrawInvestorFunds` (fee) |
| `SubscriptionContract.sol` | 308 | `withdrawInvestorFunds` (net) |
| `RedemptionContract.sol` | 90 | `depositSettlement` |
| `RedemptionContract.sol` | 290 | `redeem` (fee) |
| `RedemptionContract.sol` | 296 | `redeem` (payout) |
| `RedemptionContract.sol` | 346 | `handleUnredeemedTokens` |

    
  **Recommendations**

- Import and apply `SafeERC20` throughout all contracts:


  import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";


  using SafeERC20 for IERC20;


  IERC20(stablecoin).safeTransfer(recipient, amount);


  IERC20(stablecoin).safeTransferFrom(msg.sender, address(this), amount);


- Remove all `if (!...) revert` wrappers around transfer calls; `safeTransfer` handles revert internally.  
- For inbound fee-on-transfer protection, measure actual received amounts via balance-difference:


  uint256 balBefore \= IERC20(stablecoin).balanceOf(address(this));


  IERC20(stablecoin).safeTransferFrom(msg.sender, address(this), amount);


  uint256 received \= IERC20(stablecoin).balanceOf(address(this)) \- balBefore;

### 

5. **Dust Settlement Rate Lock**  
   Severity: High   
   Status: **Partially Fixed**   
   Location: `src/redemption/RedemptionContract.sol:58–115` (depositSettlement) · `src/redemption/RedemptionContract.sol:265` (redeem)   
   Category: Missing Validation / Economic Attack / Arithmetic

   **Description**  
   This finding describes two interacting defects that are low-impact in isolation but compose into a complete investor capital loss path when a semi-trusted trader (off-chain enforcement only) acts adversarially.  
     
   Phase 1 — Dust deposit permanently locks a garbage settlement rate.  
     
   `depositSettlement` enforces only that `settlementAmount > 0` (line 61). No minimum threshold relative to `totalRaised`, `totalSupply`, or any economic parameter is required. A trader calling `depositSettlement(tradeToken, 1)` — 1 wei of stablecoin — triggers two irreversible consequences within a single transaction, before any admin action:  
     
   Consequence A (line 105): `redemptionValuePerToken[tradeToken]` is set to `(1 * 1e18) / totalSupply ≈ 0`. The reentrancy guard at line 86–88 checks `if ($.redemptionValuePerToken[tradeToken] != 0) revert SettlementAlreadyDeposited`, preventing any corrective re-deposit. No admin role can reset this value.  
     
   Consequence B (line 111–114): `ISubscriptionContract(subscriptionContract).closeFundingOnSettlement(tradeToken)` fires unconditionally. New investor subscriptions are permanently blocked from the moment of deposit — before `confirmSettlement` or `openRedemption` is called.  
     
   Phase 2 — Garbage rate causes zero-output redemption for every investor.  
     
   `redeem()` computes stablecoin payout at line 265:  
     
   // RedemptionContract.sol:265  
     
   uint256 stablecoinAmount \= (tokenAmount \* redemptionValuePerToken) / 1e18;  
     
   With `redemptionValuePerToken ≈ 0` established in Phase 1, `stablecoinAmount = 0` for any practical `tokenAmount`. No zero-output guard exists. The function burns the caller's tokens and transfers 0 stablecoins silently — the investor permanently loses their entire position.  
     
   Phase 2 requires OPERATIONS\_ADMIN to execute `confirmSettlement` and `openRedemption` after the dust deposit. An informed admin would not do so for a 1-wei deposit; the finding is High rather than Critical because Phase 2 is admin-gated. Phase 1's harms — permanent funding closure and garbage rate lock — are not admin-gated and constitute standalone High-severity impact.

   **Impact**  
   Irreversible Settlement State  
     
- A single 1-wei transaction permanently closes the funding phase and locks `redemptionValuePerToken` to a near-zero value with no on-chain recovery path for either the admin or investors.  
    
  Complete Investor Capital Loss  
    
- If OPERATIONS\_ADMIN confirms the garbage settlement under duress or administrative error, every investor burns their tokens for 0 stablecoins. The entire settlement pool remains undistributed.  
    
  No Admin Recovery Path  
    
- Post-deposit choices: confirm garbage settlement (investors receive nothing) or leave redemption closed (investor tokens permanently frozen). Neither path allows investor capital recovery without a contract upgrade.

  **Proof of Concept**  
  **File:** `test_ZeroValueRedemption`  
    
  This test covers both Phase 1 (dust deposit sets near-zero `redemptionValuePerToken`) and Phase 2 (zero-output redemption burns tokens silently):


| function test\_ZeroValueRedemption() public {     TradeToken token \= \_createToken(1000e6, 1e6);     stablecoin.mint(ALICE, 1000e6);     vm.startPrank(ALICE);     stablecoin.approve(address(subscriptionContract), type(uint256).max);     subscriptionContract.subscribe(address(token), 1000e6);     vm.stopPrank();     // Phase 1: Trader deposits tiny settlement — just 1 wei USDC     stablecoin.mint(TRADER, 1);     vm.startPrank(TRADER);     stablecoin.approve(address(redemptionContract), type(uint256).max);     redemptionContract.depositSettlement(address(token), 1);     vm.stopPrank();     vm.startPrank(OPERATIONS\_ADMIN);     redemptionContract.confirmSettlement(address(token));     redemptionContract.openRedemption(address(token), 30 days);     vm.stopPrank();     // Phase 2: redemptionValuePerToken \= (1 \* 1e18) / 1000e6 \= 1e12     // Redeeming 1 token unit: stablecoinAmount \= (1 \* 1e12) / 1e18 \= 0     uint256 aliceBefore \= stablecoin.balanceOf(ALICE);     uint256 tokensBefore \= token.balanceOf(ALICE);     vm.startPrank(ALICE);     token.approve(address(redemptionContract), type(uint256).max);     redemptionContract.redeem(address(token), 1); // Redeem 1 token unit     vm.stopPrank();     uint256 received \= stablecoin.balanceOf(ALICE) \- aliceBefore;     uint256 tokensBurned \= tokensBefore \- token.balanceOf(ALICE);     assertEq(tokensBurned, 1, "1 token burned");     assertEq(received, 0, "0 USDC received \-- tokens burned for nothing"); } |
| :---- |


  **Result:** Alice's token is burned; she receives 0 USDC. The settlement pool is not decremented.

  **Recommendations**

- Enforce a minimum settlement amount relative to capital raised:


  uint256 totalRaised \= ISubscriptionContract(subscriptionContract).getTotalRaised(tradeToken);


  if (totalRaised \> 0 && settlementAmount \< (totalRaised \* MIN\_SETTLEMENT\_BPS) / 10000\) {


      revert RedemptionContract\_InsufficientSettlement();


  }


- Add a zero-output guard in `redeem()` (resolves the Phase 2 component independently):


  if (stablecoinAmount \== 0\) revert RedemptionContract\_InsufficientRedemptionValue();


- Gate `closeFundingOnSettlement` on the minimum settlement threshold rather than firing unconditionally on any `settlementAmount > 0` deposit.

### 

6. **Settlement Deposit Front-Running**  
   Severity: High   
   Status: **Fixed**   
   Location: `src/funding/SubscriptionContract.sol:53` (subscribe) · `src/redemption/RedemptionContract.sol:58` (depositSettlement) — no temporal separation between them   
   Category: MEV / Economic Attack

   **Description**  
   No time-lock, funding-close requirement, or commit-reveal scheme prevents a subscription from occurring after `depositSettlement` has been submitted to the mempool. `subscribe()` checks only `!$.fundingClosed[tradeToken]`, and `closeFundingOnSettlement()` is called by `depositSettlement()` only upon execution — after the settlement transaction has been broadcast but before it confirms.  
     
   An attacker monitoring the mempool can observe a pending `depositSettlement(tradeToken, settlementAmount)` transaction, compute that the resulting `redemptionValuePerToken` exceeds the current subscription cost, front-run the deposit with a large `subscribe()` call at higher gas, and redeem for guaranteed profit after settlement confirms.  
     
   The attack is risk-free: if the front-run fails (e.g., funding closes first), the attacker simply holds the tokens. The front-runner's tokens do not change `totalSupply` (tokens are pre-minted), so they do not reduce `redemptionValuePerToken` in the settlement calculation — but they do dilute the original investors' aggregate payout from the fixed settlement pool.

   **Impact**  
   Risk-Free MEV Arbitrage  
     
- Attackers extract guaranteed profit from every publicly visible profitable `depositSettlement` transaction. No technical skill beyond mempool monitoring is required.  
    
  Existing Investor Dilution  
    
- The fixed settlement pool is distributed across a larger token holder base; each pre-existing investor's aggregate payout is reduced in proportion to the attacker's subscription size.  
    
    
  Structural Attack Surface  
    
- The attack is repeatable across every trade. No per-trade defense exists; the protocol's open subscription model is structurally exposed.

  **Proof of Concept**  
  **File:** `test_FrontRunSettlement_Arbitrage`

| function test\_FrontRunSettlement\_Arbitrage() public {     // 1000 tokens at $1. Only 200 sold so far.     TradeToken token \= \_createToken(1000e6, 1e6);     // Legitimate investor Alice subscribes 200 tokens     \_fund(ALICE, 200e6);     vm.prank(ALICE);     subscriptionContract.subscribe(address(token), 200e6);     // Attacker sees trader's depositSettlement(token, 2000e6) in mempool     // Attacker front-runs: subscribes 800 remaining tokens for $800     \_fund(ATTACKER, 800e6);     vm.prank(ATTACKER);     subscriptionContract.subscribe(address(token), 800e6);     // Trader's depositSettlement executes     stablecoin.mint(TRADER, 2000e6);     vm.startPrank(TRADER);     stablecoin.approve(address(redemptionContract), type(uint256).max);     redemptionContract.depositSettlement(address(token), 2000e6);     vm.stopPrank();     // Admin pipeline     vm.startPrank(OPERATIONS\_ADMIN);     redemptionContract.confirmSettlement(address(token));     redemptionContract.openRedemption(address(token), 30 days);     vm.stopPrank();     // Attacker redeems immediately     vm.startPrank(ATTACKER);     token.approve(address(redemptionContract), type(uint256).max);     redemptionContract.redeem(address(token), 800e6);     vm.stopPrank();     uint256 attackerFinal \= stablecoin.balanceOf(ATTACKER);     int256 attackerProfit \= int256(attackerFinal) \- 800e6;     assertTrue(attackerProfit \> 0, "Attacker profits");     assertEq(uint256(attackerProfit), 800e6, "Attacker doubles their money"); } |
| :---- |


  **Result:** Attacker invests $800, receives $1,600 back. $800 risk-free profit confirmed on-chain.


  **Recommendations**

- Require `fundingClosed[tradeToken] == true` as a precondition for `depositSettlement`, enforcing a clean phase separation between subscription and settlement.  
- If continuous or late subscriptions are a design requirement, implement a mandatory delay between `closeFunding()` and the earliest permitted `depositSettlement()` (e.g., `require(block.timestamp >= fundingClosedAt[tradeToken] + SETTLEMENT_DELAY)`).  
- Alternatively, snapshot the token holder set at `closeFunding` time and compute settlement against that snapshot, preventing post-close subscriptions from participating.


## 

## **Medium Severity Issues** {#medium-severity-issues}

1. **Permissionless Request Queue Griefing**  
   Severity: Medium   
   Status: **Fixed**   
   Location: `src/factory/TokenFactory.sol:46–83` (requestTradeToken) · `src/factory/TokenFactory.sol:321–333` (\_removePendingRequestId)   
   Category: Access Control / Denial of Service

   **Description**  
   `requestTradeToken` has no role restriction; any external account can call it:  
     
   // TokenFactory.sol:46  
     
   function requestTradeToken(...) external {  
     
       // No role check  
     
   Each call appends a new entry to `pendingRequestIds`. The removal helper `_removePendingRequestId`, called by `approveTradeToken` for every approval, uses an O(n) linear scan:  
     
   // TokenFactory.sol:321–333  
     
   function \_removePendingRequestId(Storage.Layout storage $, uint256 requestId) internal {  
     
       uint256 len \= $.pendingRequestIds.length;  
     
       for (uint256 i \= 0; i \< len; i++) {  
     
           if ($.pendingRequestIds\[i\] \== requestId) {  
     
               $.pendingRequestIds\[i\] \= $.pendingRequestIds\[len \- 1\];  
     
               $.pendingRequestIds.pop();  
     
               return;  
     
           }  
     
       }  
     
   }  
     
   An attacker can spam `requestTradeToken` with unique `requestId` values to grow `pendingRequestIds` until the linear scan's gas cost during `approveTradeToken` exceeds the block gas limit. At approximately 5,000 pending entries on Ethereum mainnet, the scan gas exceeds 30M gas, making `approveTradeToken` uncallable. No admin function exists to drain the queue.  
     
   A secondary front-running vector compounds this: `requestId` is caller-supplied and must be globally unique. An attacker observing a pending `requestTradeToken(requestId=X)` in the mempool can front-run it with the same `requestId`, causing the legitimate transaction to revert with `TokenFactory_RequestIdAlreadyUsed`. The attacker can sustain this at gas cost only, indefinitely denying any specific ID from being registered by the legitimate caller.

   **Impact**  
   Administrative Denial of Service  
     
- OPERATIONS\_ADMIN cannot approve legitimate trade token requests once the pending queue grows past the block gas threshold.  
- Recovery requires a purpose-built admin function to prune the queue, which does not currently exist.  
    
  Permissionless ID Space Griefing  
    
- Any backend-signed `requestId` can be front-run and poisoned at negligible cost, forcing repeated backend retries with no guarantee of success.

  **Proof of Concept**  
  **File:** `test_F3_PermissionlessRequestSpam`

| function test\_F3\_PermissionlessRequestSpam() public {     // Anyone can spam requestTradeToken \-- no access control, no cost.     vm.startPrank(ATTACKER);     for (uint256 i \= 1; i \<= 500; i++) {         tokenFactory.requestTradeToken(i, 1, 1, address(stablecoin), 0, bytes32(0));     }     vm.stopPrank();     uint256 pending \= tokenFactory.getPendingRequestCount();     assertEq(pending, 500, "500 spam requests with no cost to attacker"); } |
| :---- |


  **Result:** 500 pending entries created by a single attacker. Each subsequent `approveTradeToken` must iterate the full array. At \~5,000 entries the gas cost exceeds block limits.


  **Additional PoC:** `test_POC_PendingRequestQueueDOS` — 100 spam requests demonstrated.

  **Recommendations**

- Restrict `requestTradeToken` to a `TRADER_ROLE` or require an EIP-712 signature from an authorized backend key, eliminating the zero-cost griefing vector.  
- Replace the O(n) scan in `_removePendingRequestId` with O(1) removal using an index mapping:


  mapping(uint256 \=\> uint256) pendingRequestIdIndex; // requestId → array index


- Add an emergency admin function to bulk-remove pending entries (e.g., `clearPendingRequests(uint256[] calldata ids)`) callable by `OPERATIONS_ADMIN`.

### 

2. **Irrecoverable Direct Transfer Funds**  
   Severity: Medium   
   Status: **Acknowledged**   
   Location: `SubscriptionContract`, `RedemptionContract`, `StratumAccessControl`, `TokenFactory`, `TradeToken` — all lack rescue functions   
   Category: Missing Functionality / Fund Recovery

   **Description**  
   Stablecoins or trade tokens sent directly to any protocol contract via `ERC20.transfer()` — rather than through the intended protocol entry points — are permanently unrecoverable. No contract in scope implements a `sweep`, `rescue`, or `emergencyWithdraw` function.  
     
   The affected transfer paths include: user error (investor sends stablecoins to `SubscriptionContract.address` instead of calling `subscribe()`), integration error (external contract routes stablecoins directly to a protocol address), and accounting leakage (any direct transfer to `StratumAccessControl` outside `receiveFee` inflates the raw balance without updating `accumulatedFees[stablecoin]`, making `withdrawFees` unable to account for it).  
     
   For `SubscriptionContract`, a direct transfer increases the contract's USDC balance without incrementing `totalRaised`, creating a balance surplus with no on-chain claim. For `StratumAccessControl`, `withdrawFees` reads from `accumulatedFees[stablecoin]`, not from the raw balance — directly transferred tokens cannot be withdrawn by any existing function, even by `SUPER_ADMIN`.

   **Impact**  
   Permanent Fund Loss  
     
- Funds sent outside of protocol functions are irrecoverable without deploying a new contract implementation specifically to perform recovery — a significant operational burden.  
    
  Accounting Discrepancy  
    
- Direct transfers create a divergence between a contract's actual token balance and its internal accounting state that persists indefinitely and can mislead on-chain tooling.

    
  **Proof of Concept**  
  **File:** `test_DirectTransfer_LocksFunds`

| function test\_DirectTransfer\_LocksFunds() public {     TradeToken token \= \_createToken(1000e6, 1e6);     // Someone accidentally sends 500 USDC directly to SubscriptionContract     stablecoin.mint(ATTACKER, 500e6);     vm.prank(ATTACKER);     stablecoin.transfer(address(subscriptionContract), 500e6);     // Those funds are now permanently locked     uint256 subBal \= stablecoin.balanceOf(address(subscriptionContract));     uint256 totalRaised \= subscriptionContract.getTotalRaised(address(token));     uint256 available \= subscriptionContract.getAvailableFunds(address(token));     assertEq(subBal, 500e6, "Contract holds 500 USDC");     assertEq(totalRaised, 0, "Accounting says 0");     assertEq(available, 0, "Nothing withdrawable \- funds LOCKED"); } |
| :---- |


  **Result:** 500 USDC held by contract, 0 accounted, 0 withdrawable. Permanently irrecoverable.

  **Recommendations**

- Add a role-gated `rescueERC20(address token, address to, uint256 amount)` function to each contract, callable only by `SUPER_ADMIN` or `DEFAULT_ADMIN`.  
- Constrain the rescue function to exclude tokens that are part of active accounting (e.g., block rescuing from `settlementPools` or active subscription escrow balances) to prevent misuse.

## **Low Severity Issues** {#low-severity-issues}

1. **CEI Ordering Pattern Violation**  
   Severity: Low   
   Status: **Fixed**  
   Location: `src/redemption/RedemptionContract.sol:58–115` (depositSettlement) · `src/redemption/RedemptionContract.sol:316–360` (handleUnredeemedTokens)   
   Category: Reentrancy Hygiene / Code Quality

   **Description**  
   Both `depositSettlement` and `handleUnredeemedTokens` violate the Checks-Effects-Interactions (CEI) pattern and are missing `nonReentrant` guards.  
     
   In `depositSettlement` (lines 58–115), the external `IERC20.transferFrom` call (line 90\) precedes all state writes (lines 101–108). In `handleUnredeemedTokens` (lines 316–360), the external `IERC20.transfer` to `$.accessControl` (line 346\) and the `IStratumAccessControl.receiveFee` call (line 349\) both precede `$.settlementPools[tradeToken] = 0` (line 352).  
     
   Under the protocol's supported stablecoin set (USDC, USDT), neither token issues callbacks on `transfer` or `transferFrom`. The reentrancy path for `depositSettlement` requires an ERC-777 stablecoin (not in scope); the resulting exploit is self-defeating (attacker loses funds). The reentrancy path for `handleUnredeemedTokens` requires a maliciously upgraded `$.accessControl`, which falls outside the trust model. Neither is practically exploitable as deployed.  
     
   The finding is Low because the CEI fix is mandatory hygiene for production DeFi contracts: the absence of `nonReentrant` creates latent risk that activates if a non-standard stablecoin is approved in future or if the admin key is compromised.

   **Recommendations**  
- Restructure `depositSettlement` as CEI: move all `$.settlementPools`, `$.redemptionValuePerToken`, and related storage writes before the `transferFrom` and `closeFundingOnSettlement` calls.  
- Restructure `handleUnredeemedTokens` as CEI: zero `$.settlementPools[tradeToken]` before the outbound `IERC20.transfer` and `receiveFee` calls.  
- Add `nonReentrant` to both functions regardless of CEI reorder, as defense-in-depth against future stablecoin additions.

2. **Fee Rate Constant Mismatch**  
   Severity: Low   
   Status: **Fixed**   
   Location: `src/access/StratumAccessControl.sol:44`   
   Category: Specification Inconsistency

   **Description**  
   `MAX_FEE_RATE` is defined as `10000` (100% in basis points):  
     
   // StratumAccessControl.sol:44  
     
   uint256 public constant MAX\_FEE\_RATE \= 10000;  
     
   All NatSpec comments on the three fee setters — `setInvestFeeRate`, `setCashoutFeeRate`, `setTraderWithdrawalFeeRate` — across `StratumAccessControl`, `SubscriptionContract`, and `RedemptionContract` state `max 1000 = 10%`. The enforced ceiling is 10× higher than the documented ceiling. Setting any fee to 10000 produces `netAmount = 0` for the affected participant class via the formula `(amount * 10000) / 10000 = amount` (full extraction). This requires `SUPER_ADMIN` to set the rate — a trusted role — but the constant value does not match the protocol's own specification.

   **Proof of Concept**  
   **File:** `test_F5_FeeCapMismatch`

| function test\_F5\_FeeCapMismatch() public {     // Code allows 100% \-- docs say max 10%     vm.prank(SUPER\_ADMIN);     accessControl.setInvestFeeRate(10000); // 100% succeeds     vm.prank(SUPER\_ADMIN);     vm.expectRevert(); // Only \>100% reverts     accessControl.setInvestFeeRate(10001); } |
| :---- |

   

   

   **Additional PoC:** `test_POC_100PercentFeeRate`

| function test\_POC\_100PercentFeeRate() public {     TradeToken token \= \_createToken(1000e6, 1e6);     vm.prank(SUPER\_ADMIN);     accessControl.setInvestFeeRate(10000); // 100%     stablecoin.mint(ALICE, 1000e6);     vm.startPrank(ALICE);     stablecoin.approve(address(subscriptionContract), type(uint256).max);     subscriptionContract.subscribe(address(token), 100e6);     vm.stopPrank();     uint256 totalRaised \= subscriptionContract.getTotalRaised(address(token));     uint256 feeBalance \= stablecoin.balanceOf(address(accessControl));     assertEq(totalRaised, 0, "Trader gets nothing \- all went to fees");     assertEq(feeBalance, 100e6, "All funds captured as fees"); } |
| :---- |

   

   **Result:** `setInvestFeeRate(10000)` passes validation. At 100% fee rate, investor pays $100 but `totalRaised = 0`.

   **Recommendations**

- Change `MAX_FEE_RATE` from `10000` to `1000` in `StratumAccessControl.sol`.  
- Remove the hardcoded literal `10000` checks in `SubscriptionContract` and `RedemptionContract`; import and reference `IStratumAccessControl.MAX_FEE_RATE` so any future correction propagates automatically.

### 

3. **Inline Role Hash Computation**  
   Severity: Low   
   Status: **Fixed**   
   Location: `TokenFactory.sol:98,344` · `SubscriptionContract.sol:145` · `RedemptionContract.sol:125,154,193,215,322`   
   Category: Code Quality / Typo Risk

   **Description**  
   Role identifiers are computed at each use site via `keccak256(abi.encodePacked("OPERATIONS_ADMIN_ROLE"))` rather than referencing the constant exported by `StratumAccessControl`. With seven-plus inline occurrences across three contracts, a single character typo in any one instance silently produces a different bytes32 hash. The affected function's `hasRole(wrongHash, msg.sender)` check returns false, causing the restriction to behave as if no access control exists — the function becomes unguarded without emitting any error. `StratumAccessControl` correctly declares `OPERATIONS_ADMIN_ROLE` as a `public constant bytes32`; the other contracts do not import it.

   **Code Affected:**  
   // Typo introduced in RedemptionContract.sol:125  
     
   bytes32 role \= keccak256(abi.encodePacked("OPERATION\_ADMIN\_ROLE"));  // missing 'S'  
     
   // hasRole(role, caller) always false for any caller  
     
   // Function proceeds as if unrestricted

   **Recommendations**  
- Import `IStratumAccessControl` and reference the exported `OPERATIONS_ADMIN_ROLE` constant at every call site, or declare a shared `Roles` library imported by all three contracts.

### 

4. **Redemption Rounding Remainder**  
   Severity: Low   
   Status: **Acknowledged**   
   Location: `src/redemption/RedemptionContract.sol:99,265`   
   Category: Arithmetic / Dust

   **Description**  
   The redemption calculation involves two sequential integer divisions: `redemptionValuePerToken = (settlementAmount * 1e18) / totalSupply` (line 99\) and `stablecoinAmount = (tokenAmount * redemptionValuePerToken) / 1e18` (line 265). Integer truncation at each step ensures that the sum of all stablecoins paid out across all redemptions is strictly less than `settlementAmount`. A remainder of 1–2 wei remains in `settlementPools` after all investors have redeemed. This remainder is swept to `StratumAccessControl` by `handleUnredeemedTokens()`.  
     
   The economic impact per trade is negligible (at most a few wei). The behavioral note is that `settlementPools[tradeToken]` never reaches exactly zero through redemptions alone; a small residue always remains.

   **Proof of Concept**  
   **File:** `test_F4_RedemptionDust`

| function test\_F4\_RedemptionDust() public {     // 7 tokens, 10 USDC settlement \-- indivisible     TradeToken token \= \_createToken(7e6, 1e6);     stablecoin.mint(ALICE, 100e6);     vm.startPrank(ALICE);     stablecoin.approve(address(subscriptionContract), type(uint256).max);     subscriptionContract.subscribe(address(token), 7e6);     vm.stopPrank();     stablecoin.mint(TRADER, 10e6);     vm.startPrank(TRADER);     stablecoin.approve(address(redemptionContract), type(uint256).max);     redemptionContract.depositSettlement(address(token), 10e6);     vm.stopPrank();     vm.startPrank(OPERATIONS\_ADMIN);     redemptionContract.confirmSettlement(address(token));     redemptionContract.openRedemption(address(token), 30 days);     vm.stopPrank();     uint256 balBefore \= stablecoin.balanceOf(ALICE);     vm.startPrank(ALICE);     token.approve(address(redemptionContract), type(uint256).max);     redemptionContract.redeem(address(token), 7e6);     vm.stopPrank();     uint256 received \= stablecoin.balanceOf(ALICE) \- balBefore;     uint256 dust \= redemptionContract.getSettlementPool(address(token));     assertTrue(received \< 10e6, "Alice doesn't get full settlement");     assertTrue(dust \> 0, "Dust remains locked"); } |
| :---- |

   

   **Result:** Settlement \= 10 USDC, 7 tokens. Alice receives 9,999,999 wei. 1 wei dust remains in pool.

   **Recommendations**

- No code change required. Document in the NatSpec of `depositSettlement` and `handleUnredeemedTokens` that a 1–2 wei rounding remainder is expected behavior.

5. **Missing Redemption Finalization Guard**  
   Severity: Low   
   Status: **Acknowledged**   
   Location: `src/redemption/RedemptionContract.sol:316–360`   
   Category: State Guard / Idempotency

   **Description**  
   `handleUnredeemedTokens` can be called multiple times without restriction after the first execution. On the first call, `$.settlementPools[tradeToken]` is read, transferred, and set to zero. On all subsequent calls, `remainingSettlement = 0`, so no transfer occurs — but the function still emits `UnredeemedTokensHandled(tradeToken, 0, block.timestamp)` with a zero-amount payload. Any event indexer or off-chain consumer that monitors this event will observe spurious re-emissions, misrepresenting that the sweep operation was repeated.

   **Recommendations**  
- Add a `$.unredeemedHandled[tradeToken]` boolean flag that is set on the first successful call; subsequent calls revert with a descriptive error.  
- Alternatively, revert early if `remainingSettlement == 0` before emitting any event.

## **Informational** {#informational}

1. **Silent Decimal Fallback**  
   Severity: Informational   
   Status: **Fixed**   
   Location: `src/factory/TokenFactory.sol:171–176`   
   Category: Error Handling

   **Description**  
   The `try/catch` around `IERC20Metadata(request.stablecoin).decimals()` silently falls back to 18 decimals on any revert. A stablecoin whose `decimals()` transiently fails during deployment (e.g., during a proxy upgrade) results in a `TradeToken` with `TOKEN_DECIMALS = 18` instead of the correct value. This is the same root cause that motivated this branch; the fallback reintroduces it through a different path with no event, no warning, and no revert.  
     
   **Recommendation**  
   Remove the `catch` fallback. Let a `decimals()` revert propagate and surface as a token approval failure with a descriptive error from the factory.

2. **Premature Investor Fund Withdrawal**  
   Severity: Informational   
   Status: Acknowledged   
   Location: src/funding/SubscriptionContract.sol:263–313   
   Category: Design Note / Economic Risk

   **Description**  
   withdrawInvestorFunds has no precondition requiring $.fundingClosed\[tradeToken\]. A trader can withdraw subscribed capital while the round is still active. A code comment reads "Can be called at any time", confirming this is a deliberate design decision acknowledged in the pre-review threat model (AUDIT\_PRE\_REVIEW.md, ECON-03). The risk is that investors in an active round are unaware that capital they believe is escrowed has already left the contract. This also forms a precondition in H-05 (trader withdraws capital then deposits 1-wei settlement).  
     
   **Recommendation**  
   Emit a FundsWithdrawnBeforeClosure(tradeToken, amount, block.timestamp) event flagged in user-facing interfaces when withdrawInvestorFunds is called while fundingClosed \== false, to surface the timing to investors and monitoring systems.  
     
3. **Redundant Restriction Check Calls**  
   Severity: Informational   
   Status: **Fixed**   
   Location: `src/tokens/TradeToken.sol:161–174`   
   Category: Gas Optimization

   **Description**  
   `_checkTransferRestrictions` calls `IStratumAccessControl.isBlacklisted(from)` twice in the revert path: once in the `if` condition and once in the ternary within the revert argument. For a blacklisted `from` address, this results in three cross-contract calls to `StratumAccessControl` instead of two. Caching results in local booleans eliminates the redundant external call on every transfer.  
     
   **Recommendation**  
   Cache both blacklist results before the conditional:  
     
   bool fromBlocked \= IStratumAccessControl(accessControl).isBlacklisted(from);  
     
   bool toBlocked   \= IStratumAccessControl(accessControl).isBlacklisted(to);  
     
   if (fromBlocked || toBlocked) {  
     
       revert TradeToken\_ProtocolBlacklisted(fromBlocked ? from : to);  
   }

4. **Non-Unique TradeToken Symbol**  
   Severity: Informational   
   Status: **Fixed**   
   Location: `src/factory/TokenFactory.sol:169`   
   Category: Off-Chain Indexing

   **Description**  
   Every `TradeToken` is deployed with `symbol = "STT"`. Token names include the trade ID (`"Stratum Trade Token 1"`) but symbols do not. Etherscan, DEX aggregators, wallet UIs, and compliance systems use symbol as the primary token identifier; all trade tokens appear identically as "STT". Any tooling that uses symbol as a deduplication key will silently collapse all trade tokens into the same entry.  
     
   **Recommendation**  
   Encode the trade ID in the symbol: `string(abi.encodePacked("STT-", Strings.toString(tradeId)))`.  
5. **Inverted Redemption Error Name**  
   Severity: Informational   
   Status: **Fixed**   
   Location: `src/redemption/RedemptionContract.sol:163–164`   
   Category: Code Quality

   **Description**  
   // RedemptionContract.sol:163–164  
     
   if ($.redemptionOpen\[tradeToken\]) {  
     
       revert RedemptionContract\_RedemptionNotOpen();  // fires when redemption IS open  
     
   }  
     
   The guard correctly prevents double-opening. The error name `RedemptionContract_RedemptionNotOpen` is the logical inverse of the triggering condition — it fires when `redemptionOpen == true`. An integrator or debugger who reads the error name will form an incorrect mental model of the contract state at revert time.  
     
   **Recommendation**  
   Rename to `RedemptionContract_RedemptionAlreadyOpen`.

6. **Absent Minimum Redemption Deadline**  
   Severity: Informational   
   Status: **Acknowledged**   
   Location: `src/redemption/RedemptionContract.sol:187–206`   
   Category: Defense-in-Depth / Governance

   **Description**  
   `setRedemptionDeadline` enforces only `deadline > 0`. OPERATIONS\_ADMIN can set `deadline = 1` (1 second), making `handleUnredeemedTokens` callable in the next block even while the redemption window is nominally open. This requires trusted OPERATIONS\_ADMIN action and falls within the trust model; no access control is bypassed. The note is defense-in-depth guidance only.  
     
   **Recommendation**  
   Enforce `deadline >= MIN_REDEMPTION_DURATION` (e.g., `7 days`) and prevent shortening the deadline below the currently remaining window (`require(absoluteDeadline >= $.redemptionDeadline[tradeToken])`).

7. **Absent On-Chain Token Whitelist**  
   Severity: Informational   
   Status: **Acknowledged**   
   Location: `src/factory/TokenFactory.sol:46–83`   
   Category: Defense-in-Depth

   **Description**  
   The two-step `requestTradeToken` / `approveTradeToken` flow provides a human whitelist: OPERATIONS\_ADMIN reviews each stablecoin before approval. No on-chain registry of permitted stablecoin addresses exists. Future operators, upgraded admin contracts, or automated tooling cannot programmatically verify that a given stablecoin meets protocol eligibility requirements (non-rebasing, non-fee-on-transfer, non-ERC-777, `transfer()` returns `bool`). Sub-vectors requiring OPERATIONS\_ADMIN to knowingly approve a malicious stablecoin fall outside the trust model; the note targets future-proofing and automated governance.  
     
   **Recommendation**  
   Maintain an on-chain `mapping(address => bool) approvedStablecoins` managed by `SUPER_ADMIN`. Reference it as a hard check in `approveTradeToken` before token deployment.  
   

# **Disclaimer** {#disclaimer}

ImmuneBytes’s audit does not provide a security or correctness guarantee of the audited smart contract. Securing smart contracts is a multistep process; therefore, running a bug bounty program complementing this audit is strongly recommended.

Our team does not endorse the platform or its product, nor is this audit investment advice.

Notes:

* Please make sure contracts deployed on the mainnet are the ones audited.  
* Check for code refactoring by the team on critical issues.

  