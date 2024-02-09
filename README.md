This is a fork of [@hxronetwork/dexterity-ts](https://www.npmjs.com/package/@hxronetwork/dexterity-ts) updated and maintained by [PepperDEX](https://pepperdex.xyz/) Contributors

# Getting started

1. **Create a typescript project:**

```bash
mkdir dexterity-project
cd dexterity-project
npm i typescript --save-dev
npx tsc --init
```

1. **Install Dependencies:**

```bash
npm install @solana/web3.js @pepperdex/dexterity-ts @coral-xyz/anchor
```

1. **Create a Solana Wallet and export your private/secret-key:**

[Solflare](https://solflare.com/download)

1. **Send yourself some devnet SOL:**

[SOL Faucet](https://solfaucet.com/)

1. **Send yourself some devnet UXDC:**

[UXDC Faucet](https://uxdc-faucet-api-1srh.vercel.app/)

# **Index**

1. **Create TRG, Get TRGs, Deposit, Withdraw, Place Order**
2. **Get all MPGs and their Products for each**
3. **Fetch, parse & stream Orderbook**
4. **Mark & Index Price**
5. **Account info**
6. **Instructions Bundle**

# 1. Create TRG, Get TRGs, Deposit, Withdraw, Place Order

```jsx
import { Keypair, PublicKey } from '@solana/web3.js';
import dexterityTs from '@pepperdex/dexterity-ts';
import { Wallet } from '@coral-xyz/anchor';
const dexterity = dexterityTs;

// Configure Solana Devnet connection and relevant constants
const rpc = "your-devnet-rpc-url";
const keypair = Keypair.fromSecretKey(new Uint8Array([])); // Replace with your private key
const wallet = new Wallet(keypair);
const pubkey = 'wallet-pubkey'; // Replace with your wallet public key
const MPG = "BRWNCEzQTm8kvEXHsVVY9jpb1VLbpv9B8mkF43nMLCtu"; // Replace with the desired MPG public key
const TRG = new PublicKey("your-created-trg"); // Replace with the TRG public key
const PRODUCT_NAME = 'SOLUSD-PERP'; // Replace with the desired product name

/**
 * Creates a Trader Account (TRG) for a specified MPG.
 * Consumes 0.54 SOL in rent, which is refundable upon account closure.
 */
const createTrg = async () => {
    const manifest = await dexterity.getManifest(rpc, true, wallet);
    const trg = await manifest.createTrg(new PublicKey(MPG));
    console.log('\nTRG: ', trg.toBase58());
};

/**
 * Retrieves all Trader Accounts (TRGs) associated with a specified MPG for a given wallet.
 */
const getTrgs = async () => {
    const manifest = await dexterity.getManifest(rpc, true, wallet);
    const trgArr = await manifest.getTRGsOfWallet(new PublicKey(MPG));
    const TRGs = trgArr.map(trg => trg.pubkey);
    console.log(`\nTRGs:\n`, TRGs);
};

getTrgs();

/**
 * Deposits or withdraws a specified amount to/from a Trader Account (TRG).
 */
const depositOrWithdraw = async (amount: number, type: 'deposit' | 'withdraw') => {
    const manifest = await dexterity.getManifest(rpc, true, wallet);
    const trader = new dexterity.Trader(manifest, TRG);
    const n = dexterity.Fractional.New(amount, 0);

    await trader.connect(NaN, async () => {
        console.log(`\nBALANCE: ${Number(trader.getCashBalance()).toLocaleString()} UXDC`);
    });

    const callbacks = { 
        onGettingBlockHashFn: () => {}, 
        onGotBlockHashFn: () => {}, 
        onTxSentFn: (sig: string) => console.log(`\nSUCCESSFUL ${type.toUpperCase()} OF ${amount.toLocaleString()} UXDC\n${sig ? `SIGNATURE: https://solscan.io/tx/${sig}?cluster=devnet` : ''}`),
    };

    if (type === 'deposit') await trader.deposit(n, callbacks);
    if (type === 'withdraw') await trader.withdraw(n);

    await trader.connect(NaN, async () => {
        console.log(`\nBALANCE: ${Number(trader.getCashBalance()).toLocaleString()} USDC`);
    });
};

depositOrWithdraw(10000, 'deposit');
depositOrWithdraw(5000, 'withdraw');

/**
 * Places a buy or sell order for a specified product in the MPG.
 */
const placeOrder = async (type: 'buy' | 'sell', price: number, size: number) => {
    const manifest = await dexterity.getManifest(rpc, true, wallet);
    const trader = new dexterity.Trader(manifest, TRG);

    await trader.connect(NaN, async () => {
        console.log(`\nBALANCE: ${trader.getCashBalance()} | OPEN ORDERS: ${(await Promise.all(trader.getOpenOrders([PRODUCT_NAME]))).length} | POSITIONS VALUE: ${trader.getPositionValue()} | PNL: ${trader.getPnL()}`);
    });

    // Find the index of the desired product within the MPG
    let ProductIndex;
    for (const [name, { index }] of trader.getProducts()) {
        if (name.trim() === PRODUCT_NAME.trim()) {
            ProductIndex = index;
            break;
        }
    }

    // Prepare order details
    const QUOTE_SIZE = dexterity.Fractional.New(size, 0);
    const priceFractional = dexterity.Fractional.New(price, 0);

    // Define callback functions for order handling
    const callbacks = {
        onGettingBlockHashFn: () => {},
        onGotBlockHashFn: () => {},
        onTxSentFn: (sig: string) => console.log(`\nSUCCESSFULLY PLACED LIMIT ${type.toUpperCase()} ORDER\nSIGNATURE: https://solscan.io/tx/${sig}?cluster=devnet`)
    };

    // Make sure Mark Prices are always up-to-date for orders to go through
    await trader.updateMarkPrices()

    // Place a limit order based on the specified type (buy or sell)
    if (type === 'buy') {
        await trader.newOrder(ProductIndex, true, priceFractional, QUOTE_SIZE, false {/*true = fillOrKill*/}, null, null, null, null, callbacks); // For a buy order
    } else if (type === 'sell') {
        await trader.newOrder(ProductIndex, false, priceFractional, QUOTE_SIZE, false {/*true = fillOrKill*/}, null, null, null, null, callbacks); // For a sell order
    }
};

placeOrder('buy', 30000, 100);
placeOrder('sell', 30000, 50);
```

- **Manifest Object**
    
    ```jsx
    class Manifest {
        fields: ManifestFields;
        base_api_url: string;
        slot: number;
        timestamp: Date;
        constructor(fields: ManifestFields);
        setWallet(wallet: any): void;
        static GetRiskAndFeeSigner(mpg: PublicKey): PublicKey;
        static GetStakePool(): PublicKey;
        getRiskS(marketProductGroup: PublicKey, mpg: any): PublicKey;
        getRiskR(marketProductGroup: PublicKey, mpg: any): PublicKey;
        static GetATAFromMPGObject(mpg: MarketProductGroup, wallet: PublicKey): Promise<web3.PublicKey>;
        accountSubscribe(pk: any, parseDataFn: any, onUpdateFn: any, useCache?: boolean): any;
        static GetMarkPrice(markPrices: MarkPricesArray, productKey: PublicKey): Fractional;
        static GetMarkPriceOracleMinusBookEwma(markPrices: MarkPricesArray, productKey: PublicKey): Fractional;
        static GetIndexPrice(markPrices: MarkPricesArray, productKey: PublicKey): Fractional;
        static FromFastInt(bn: BN): Fractional;
        getMPGFromData(data: any): Promise<MarketProductGroup>;
        static GetMPGFromData(dexProgram: Program, data: any): Promise<MarketProductGroup>;
        getMPG(mpg: PublicKey): Promise<MarketProductGroup>;
        static GetProductsOfMPG(mpg: MarketProductGroup): Map<any, any>;
        static GetActiveProductsOfMPG(mpg: MarketProductGroup): Map<any, any>;
        getDerivativeMetadataFromData(data: any): Promise<DerivativeMetadata>;
        static GetDerivativeMetadataFromData(instrumentsProgram: Program, data: any): Promise<DerivativeMetadata>;
        getDerivativeMetadata(productKey: PublicKey): Promise<DerivativeMetadata>;
        getTRGFromData(data: any): Promise<TraderRiskGroup>;
        getTRG(trg: PublicKey): Promise<TraderRiskGroup>;
        getMarkPricesAccount(marketProductGroup: PublicKey, mpg: any): PublicKey;
        ugh(str: any): web3.PublicKey;
        getMarkPricesFromData(data: any): Promise<MarkPricesArray>;
        getMarkPrices(markPricesAccount: PublicKey): Promise<MarkPricesArray>;
        getVarianceCache(varianceCache: PublicKey): Promise<VarianceCache>;
        getCovarianceMetadata(marketProductGroup: PublicKey, mpg: any): Promise<CovarianceMetadata>;
        getBook(product: any, marketState: any): Promise<{
            bids: any[];
            asks: any[];
        }>;
        static aaobOrderToDexPrice(aaobOrder: LeafNode, tickSize: Fractional, offset: Fractional): Fractional;
        static orderIdToDexPrice(id: BN, tickSize: Fractional, offset: Fractional): Fractional;
        static orderIdIsBid(id: BN): boolean;
        streamBooks(product: any, marketState: any, onBookFn: any, onMarkPricesFn?: any): {
            asksSocket: ReliableWebSocket;
            bidsSocket: ReliableWebSocket;
            markPricesSocket: ReliableWebSocket;
        };
        streamTrades(product: any, marketState: any, onTradesFn: any): ReliableWebSocket;
        streamMPG(mpg: PublicKey, onUpdateFn: any): ReliableWebSocket;
        getTRGsOfOwner(owner: PublicKey, marketProductGroup?: PublicKey): Promise<any[]>;
        getTRGsOfWallet(marketProductGroup?: PublicKey): Promise<any[]>;
        closeTrg(marketProductGroup: PublicKey, traderRiskGroup: PublicKey): Promise<any>;
        createTrg(marketProductGroup: PublicKey): Promise<web3.PublicKey>;
        fetchOrderbooks(marketProductGroup?: PublicKey): Promise<void>;
        fetchOrderbook(orderbook: PublicKey): Promise<any>;
        getFills(productName: string, trg: PublicKey, before: number, after: number): Promise<string | GetFillsResponse>;
        updateOrderbooks(marketProductGroup: PublicKey): Promise<void>;
        updateCovarianceMetadatas(): Promise<void>;
        static GetRiskNumber(data: any, offset: any, size: any, isSigned?: boolean): any;
        getStds(marketProductGroup: PublicKey): Map<any, any>;
    }
    ```
    
- **Trader Object**
    
    ```jsx
     class Trader {
        manifest: Manifest;
        feeAccount: PublicKey;
        feeAccountBump: number;
        marketProductGroup: PublicKey;
        traderRiskGroup: PublicKey;
        riskStateAccount: PublicKey;
        markPricesAccount: PublicKey;
        hardcodedOracle: PublicKey;
        priceOracles: Map<string, PublicKey>;
        mpg: MarketProductGroup;
        trg: TraderRiskGroup;
        varianceCache: VarianceCache;
        markPrices: MarkPricesArray;
        addressLookupTableAccount: AddressLookupTableAccount;
        trgSocket: ReliableWebSocket;
        mpgSocket: ReliableWebSocket;
        riskSocket: ReliableWebSocket;
        markPricesSocket: ReliableWebSocket;
        trgDate: Date;
        mpgDate: Date;
        riskDate: Date;
        markPricesDate: Date;
        trgSlot: Slot;
        mpgSlot: Slot;
        riskSlot: Slot;
        markPricesSlot: Slot;
        isPaused: boolean;
        skipThingsThatRequireWalletConnection: boolean;
        constructor(manifest: Manifest, traderRiskGroup: PublicKey, skipThingsThatRequireWalletConnection?: boolean);
        timeTravelToDate(toDate: Date): Promise<void>;
        getProducts(): Map<any, any>;
        getPositions(): Map<any, any>;
        getNewOrderIx(productIndex: any, isBid: any, limitPrice: Fractional, maxBaseQty: Fractional, isIOC?: boolean, referrerTrg?: any, referrerFeeBps?: any, clientOrderId?: any, matchLimit?: any): web3.TransactionInstruction;
        newOrder(productIndex: any, isBid: any, limitPrice: Fractional, maxBaseQty: Fractional, isIOC: boolean, referrerTrg: any, referrerFeeBps: any, clientOrderId: any, matchLimit: any, callbacks: any): Promise<string>;
        justNewOrder(productIndex: any, isBid: any, limitPrice: Fractional, maxBaseQty: Fractional, isIOC: boolean, referrerTrg: any, referrerFeeBps: any, clientOrderId: any, matchLimit: any, callbacks: any): Promise<string>;
        updateVarianceCache(callbacks?: any): Promise<string>;
        getUpdateVarianceCacheIx(): web3.TransactionInstruction;
        getUpdateMarkPricesIx(products: Array<Product>): web3.TransactionInstruction;
        initializePrintTrade(isBid: any, size: any, price: any, counterparty: PublicKey): Promise<void>;
        getOpenOrders(productNames: any): Set<Order>;
        getOpenOrderIds(productName: any): Set<string>;
        getCancelOrderIx(productIndex: any, orderId: any, noErr?: boolean, clientOrderId?: any): web3.TransactionInstruction;
        cancelOrder(productIndex: any, orderId: any, noErr?: boolean, clientOrderId?: any, callbacks?: any): Promise<string>;
        cancelOrders(productIndex: any, orderIds: any, noErr?: boolean, clientOrderIds?: any, callbacks?: any, maxCancelsPerTx?: number): Promise<any[]>;
        justCancelOrder(productIndex: any, orderId: any, noErr?: boolean, clientOrderId?: any, callbacks?: any): Promise<string>;
        justCancelOrders(productIndex: any, orderIds: any, noErr?: boolean, clientOrderIds?: any, callbacks?: any, maxCancelsPerTx?: number): Promise<any[]>;
        cancelAllOrders(productNames: any, isUseCache?: boolean, justIssueCancels?: boolean, maxCancelsPerTx?: number): Promise<any[]>;
        getDepositIx(usdcAmount: Fractional): Promise<web3.TransactionInstruction>;
        fetchAddressLookupTableAccount(): Promise<void>;
        sendV0Tx(ixs: any, { onGettingBlockHashFn, onGotBlockHashFn, onTxSentFn }?: {
            onGettingBlockHashFn: any;
            onGotBlockHashFn: any;
            onTxSentFn: any;
        }): Promise<string>;
        sendLegacyTx(ixs: any, { onGettingBlockHashFn, onGotBlockHashFn, onTxSentFn }?: {
            onGettingBlockHashFn: any;
            onGotBlockHashFn: any;
            onTxSentFn: any;
        }): Promise<string>;
        sendTx(ixs: any, callbacks: any): Promise<string>;
        deposit(usdcAmount: Fractional, callbacks: any): Promise<string>;
        justDeposit(usdcAmount: Fractional, callbacks: any): Promise<string>;
        updateTraderRiskGroupOwner(newOwner: PublicKey, oldOwner?: PublicKey): Promise<void>;
        withdraw(usdcAmount: Fractional): Promise<void>;
        updateOrderbooks(): Promise<void>;
        disconnect(): void;
        streamUpdates(onUpdateFn: any): void;
        updateRisk(): Promise<void>;
        getRiskNumber(offset: any, size: any, isSigned?: boolean): any;
        getVarianceCacheUpdateSlot(): any;
        getPositionValue(): Fractional;
        getTradedVariance(): Fractional;
        getOpenOrderVariance(): Fractional;
        getNotionalMakerVolume(): Fractional;
        getNotionalTakerVolume(): Fractional;
        getReferredTakersNotionalVolume(): Fractional;
        getReferralFees(): Fractional;
        getCashBalance(): Fractional;
        getPendingCashBalance(): Fractional;
        getNetCash(): Fractional;
        getPortfolioValue(): Fractional;
        getTotalDeposited(): Fractional;
        getTotalWithdrawn(): Fractional;
        getDepositedCollateral(): Fractional;
        getPnL(): Fractional;
        getRequiredMaintenanceMargin(): any;
        getRequiredMaintenanceMarginWithoutOpenOrders(): any;
        getRequiredInitialMargin(): any;
        getRequiredInitialMarginWithoutOpenOrders(): any;
        getExcessMaintenanceMargin(): Fractional;
        getExcessInitialMargin(): Fractional;
        getExcessMaintenanceMarginWithoutOpenOrders(): Fractional;
        getExcessInitialMarginWithoutOpenOrders(): Fractional;
        updateMarkPrices(): Promise<void>;
        update(isUpdateMPG?: boolean): Promise<void>;
        connect(streamUpdatesCallback: any, initialUpdateCallback: any): Promise<void>;
    }
    ```
    

# 2. Get all MPGs and Products for each

Get all the MPGs for dexterity on the selected solana network (mainnet, devenet or testnet) and each of their products:**Code Implementation:**

```jsx
import { Keypair } from '@solana/web3.js';
import dexterityTs from '@pepperdex/dexterity-ts';
import { Wallet } from '@coral-xyz/anchor';
const dexterity = dexterityTs;

// Replace with your private key
const keypair = Keypair.fromSecretKey(new Uint8Array([]));

// Initialize a wallet instance for Dexterity using the keypair
const wallet = new Wallet(keypair);

// Specify your RPC URL here
const rpc = 'rpc-url';

/**
 * Retrieves and logs information about all Market Product Groups (MPGs) and their associated products.
 * Filters out a specific MPG if necessary.
 */
const getMpgs = async () => {
    // Fetch the manifest from Dexterity which contains market information
    const manifest = await dexterity.getManifest(rpc, true, wallet);

    // Convert the MPG Map to an array for easier processing
    const mpgs = Array.from(manifest.fields.mpgs.values());

    // Iterate through each MPG to access its products
    for (const { pubkey, mpg, orderbooks } of mpgs) {
        // Skip a specific MPG if needed (replace "MPG-PUBKEY-HERE" with the actual public key to skip)
        if (pubkey.toBase58() === "MPG-PUBKEY-HERE") continue;

        // Iterate through each product in the MPG
        for (const [_, { index, product }] of dexterity.Manifest.GetProductsOfMPG(mpg)) {
            // Convert the product data to a more readable format
            const meta = dexterity.productToMeta(product);

            // Log the index and name of each product for debugging and information purposes
            console.log('productIndex: ', index);
            console.log('Name: ', dexterity.bytesToString(meta.name).trim());
        }
    }
};

getMpgs();
```

# 3. Orderbook

Get Live Orderbook ASK & BID data for a given product in a given MPG

**Code Implementation:**

```tsx
import { Keypair } from '@solana/web3.js';
import dexterityTs from '@pepperdex/dexterity-ts';
import { Wallet } from '@coral-xyz/anchor';
const dexterity = dexterityTs;

// Initialize your keypair from your private/secret key
const keypair = Keypair.fromSecretKey(new Uint8Array([]));
// When working with a UI you need to use the DexterityWallet type
const wallet = new Wallet(keypair);
const rpc =
    'https://devnet.helius-rpc.com/?api-key=';
// Testnet MPG
const mpgPubkey = 'BRWNCEzQTm8kvEXHsVVY9jpb1VLbpv9B8mkF43nMLCtu';
// Desired Product
const productName = 'SOLUSD-PERP';

/**
 * Asynchronously retrieves the order book for a specific product.
 * 
 * - Fetches the manifest containing market information.
 * - Filters to find the desired Market Product Group (MPG).
 * - Iterates through products to find the specific product and its associated market state.
 * - Initializes streaming of order book data for the selected product.
 */
const getOB = async () => {
    // Fetch the manifest containing market information
    const manifest = await dexterity.getManifest(rpc, true, wallet);

    // Convert MPG map to an array for easier processing
    const mpgs = Array.from(manifest.fields.mpgs.values());

    // Find the MPG with the matching public key
    const selectedMPG = mpgs.filter(
        (value) => value.pubkey.toBase58() === mpgPubkey,
    );

    // Initialize variables to store market state and product details
    let MarketState;
    let PRODUCT;
    let productIndex;

    // Loop through each MPG to find the desired product
    for (const { pubkey, mpg, orderbooks } of selectedMPG) {
        for (const [_, { index, product }] of dexterity.Manifest.GetProductsOfMPG(mpg)) {
            // Extract metadata of the product
            const meta = dexterity.productToMeta(product);

            // Check for the specified product name
            if (dexterity.bytesToString(meta.name).trim() === productName.trim()) {
                // Fetch the current state of the order book for the product
                MarketState = await manifest.fetchOrderbook(meta.orderbook);
                productIndex = index;

                // Determine the type of product and assign it to PRODUCT
                if (product.hasOwnProperty('outright')) {
                    PRODUCT = product.outright.outright;
                } else {
                    PRODUCT = product.combo.combo;
                }

                // Logging for debugging purposes
                console.log('productIndex: ', productIndex);
                console.log('Name: ', dexterity.bytesToString(meta.name).trim());

                // Break out of the loop once the desired product is found
                break;
            }
        }
    }

    // Stream order book data for the selected product
    const { asksSocket, bidsSocket, markPricesSocket } = manifest.streamBooks(
        PRODUCT,
        MarketState,
        aggregateBookL2 // Callback function for handling streamed data
    );
};

getOB();

/**
 * Aggregates and processes Level 2 order book data.
 * @param data The incoming order book data, containing asks and bids.
 */
function aggregateBookL2(data: any) {
    
    // Define the structure for cumulative order data
    interface CumulativeOrder {
        price: number;
        ordersSize: number;
    }

    // Check for the presence of both asks and bids
    if (data.asks.length !== 0 && data.bids.length !== 0) {
        const cumulativeBids = new Map<number, CumulativeOrder>();
        const cumulativeAsks = new Map<number, CumulativeOrder>();

        // Process bids
        data.bids.forEach((order: any) => {
            const price = order.price.toNumber();
            const quantity = order.quantity.toDecimal();

            if (cumulativeBids.has(price)) {
                cumulativeBids.get(price)!.ordersSize += quantity;
            } else {
                cumulativeBids.set(price, { price, ordersSize: quantity });
            }
        });

        // Process asks
        data.asks.forEach((offer: any) => {
            const price = offer.price.toNumber();
            const quantity = offer.quantity.toDecimal();

            if (cumulativeAsks.has(price)) {
                cumulativeAsks.get(price)!.ordersSize += quantity;
            } else {
                cumulativeAsks.set(price, { price, ordersSize: quantity });
            }
        });

        // Convert Maps to sorted arrays and limit to top 10
        const bids = Array.from(cumulativeBids.values()).sort((a, b) => b.price - a.price).slice(0, 10);
        const asks = Array.from(cumulativeAsks.values()).sort((a, b) => a.price - b.price).slice(0, 10);

        // Prepare and output the result
        let output = `Order book update received:\n\nASKS:\n`;
        asks.forEach((ask, index) => {
            output += `#${index + 1} QTY: ${ask.ordersSize} | PRICE: $${ask.price}\n`;
        });

        output += '\nBIDS:\n';
        bids.forEach((bid, index) => {
            output += `#${index + 1} QTY: ${bid.ordersSize} | PRICE: $${bid.price}\n`;
        });

        process.stdout.write(output);
        process.stdout.write('\u001b[?25h'); // Show cursor
    } else {
        // Handle missing data cases
        console.log(`Missing ${data.bids.length === 0 && data.asks.length > 0 ? 'BIDS' : data.asks.length === 0 && data.bids.length > 0 ? 'ASKS' : 'BIDS & ASKS'}`);
    }
}
```

**Example Output:**

```bash
productIndex:  0
Name:  SOLUSD-PERP
Missing ASKS
Order book update received (#2):

ASKS:
#1 QTY: 0 | PRICE: $23
#2 QTY: 0 | PRICE: $23
#3 QTY: 0 | PRICE: $23
#4 QTY: 0 | PRICE: $27
#5 QTY: 0 | PRICE: $25
#6 QTY: 0 | PRICE: $23
#7 QTY: 0 | PRICE: $23
#8 QTY: 0 | PRICE: $23
#9 QTY: 0 | PRICE: $23
#10 QTY: 0 | PRICE: $23

BIDS:
#1 QTY: 0 | PRICE: $23
#2 QTY: 0 | PRICE: $23
#3 QTY: 0 | PRICE: $23
#4 QTY: 0 | PRICE: $23
#5 QTY: 0 | PRICE: $23
#6 QTY: 0 | PRICE: $23
#7 QTY: 0 | PRICE: $23
#8 QTY: 0 | PRICE: $23
#9 QTY: 0 | PRICE: $23
#10 QTY: 0 | PRICE: $23
```

# 4. Mark & Index Price

**Code implementation:**

```ts
import { Keypair, PublicKey } from '@solana/web3.js';
import dexterityTs from '@pepperdex/dexterity-ts';
import { Wallet } from '@coral-xyz/anchor';
const dexterity = dexterityTs;

// Initialize your keypair from your private/secret key
const keypair = Keypair.fromSecretKey(new Uint8Array([]));
// Initialize a wallet instance for Dexterity using the keypair
const wallet = new Wallet(keypair);
const rpc = 'https://devnet.helius-rpc.com/?api-key=';
// Testnet Market Product Group (MPG) public key
const mpgPubkey = 'BRWNCEzQTm8kvEXHsVVY9jpb1VLbpv9B8mkF43nMLCtu';
// Trader Risk Group (TRG) for the selected MPG
const TRG = new PublicKey("your-created-trg")
// Name of the desired product
const productName = 'SOLUSD-PERP';
// Uninitialized account public key
const UNINITIALIZED = new PublicKey('11111111111111111111111111111111');

/**
 * Periodically updates the market prices for products in a specified MPG.
 * - Initializes the Dexterity manifest.
 * - Creates a trader instance and repeatedly updates market prices.
 */
const updatePrices = async () => {
    // Initialize the Dexterity manifest with the RPC endpoint and wallet
    const manifest = await dexterity.getManifest(rpc, true, wallet);

    // Create a trader instance for the specified MPG
    const trader = new dexterity.Trader(manifest, TRG);

    // Set an interval to periodically update market prices
    setInterval(async () => {
        // Update the mark prices for all products in the MPG
        await trader.updateMarkPrices();

        // Iterate through all products in the MPG
        for (const [productName, obj] of dexterity.Manifest.GetProductsOfMPG(trader.mpg)) {
            const { product } = obj;

            // Skip if the product name is empty
            if (!productName.trim()) {
                continue;
            }

            // Get metadata of the product
            const meta = dexterity.productToMeta(product);

            // Skip if the product key is uninitialized
            if (meta.productKey.equals(UNINITIALIZED)) {
                continue;
            }

            // Skip if the product is of type 'combo'
            if (product.combo?.combo) {
                continue;
            }

            // Update mark prices again (consider if this is necessary)
            await trader.updateMarkPrices();

            // Fetch and log the index and mark prices for the product
            const index = Number(dexterity.Manifest.GetIndexPrice(trader.markPrices, meta.productKey));
            const mark = Number(dexterity.Manifest.GetMarkPrice(trader.markPrices, meta.productKey));
            console.log({ index, mark });
        }
    }, 500); // The interval is set to 500 milliseconds
};
```

# 5. Account info

**Code implementation:**

```ts
import { clusterApiUrl, Keypair, PublicKey } from '@solana/web3.js';
import { Wallet } from '@coral-xyz/anchor';
import dexterityTs from '@pepperdex/dexterity-ts';
const dexterity = dexterityTs;

// Solana RPC URL for connecting to the blockchain
const rpc = 'https://example-rpc.com';

// Setting up our wallet using a private key to sign transactions and interact with the blockchain
const keypair = Keypair.fromSecretKey(new Uint8Array([]));
const wallet = new Wallet(keypair);

/**
 * View account information and open orders for a specified trading group (TRG) in Dexterity.
 */
const viewAccount = async () => {
  // Fetch the latest manifest from Dexterity
  const manifest = await dexterity.getManifest(rpc, false, wallet);

  // Specify the Market-Product-Group (MPG) public key
  const MPG = new PublicKey('BRWNCEzQTm8kvEXHsVVY9jpb1VLbpv9B8mkF43nMLCtu');
  const trgPubkey = new PublicKey('your-trg'); // Replace with actual TRG public key

  console.log(`Wallet: ${wallet.publicKey.toBase58()} TRG: ${trgPubkey.toBase58()}`);

  // Define the product name to filter orders
  const PRODUCT_NAME = ['SOLUSD-PERP']; // Array of product names

  // Create a trader instance to interact with Dexterity
  const trader = new dexterity.Trader(manifest, trgPubkey);

  // Function to stream and display account and order information
  const streamAccount = async () => {
    // Retrieve all open orders for specified products
    const orders = await Promise.all(trader.getOpenOrders(PRODUCT_NAME));

    // Format and display open orders
    if (orders.length === 0) {
      console.log('No Open Orders');
    } else {
      orders.forEach((order, index) => {
        console.log(`Index: ${index} | Product: ${order.productName} | Price: $${order.price.m} | Qty: ${order.qty.m.toNumber() / 10 ** 6} | Type: ${order.isBid ? 'Bid' : 'Ask'} | Id: ${order.id.toString()}`);
      });
    }

    // Display account information
    console.log('\nOpen Orders:', orders.length, '\nPortfolio Value:', trader.getPortfolioValue().toString(), 'Position Value:', trader.getPositionValue().toString(), 'Net Cash:', trader.getNetCash().toString(), 'PnL:', trader.getPnL().toString());
  };

  // Connect to the trader account and stream account information
  await trader.connect(streamAccount, NaN);
};

viewAccount();
```

# 6. Instructions Bundles

**Code implementation:**

```ts
import { clusterApiUrl, Keypair, PublicKey } from '@solana/web3.js';
import { Wallet } from '@coral-xyz/anchor';
import dexterityTs from '@pepperdex/dexterity-ts';
const dexterity = dexterityTs;

// Solana RPC URL for connecting to the blockchain
const rpc = 'https://example-rpc.com';

// Set up the wallet using a private key to sign transactions
const keypair = Keypair.fromSecretKey(new Uint8Array([])); // Replace with actual private key
const wallet = new Wallet(keypair);

// Initialize constants like the Trader Account (TRG) and Product Index
const TRG = new PublicKey(''); // Replace with actual TRG public key
const PRODUCT_INDEX = 0; // Index for 'SOLUSD-PERP' or similar product

/**
 * Demonstrates how to efficiently submit multiple instructions as a bundle.
 */
const submitIxBundle = async () => {
    // Fetch the latest manifest from Dexterity
    const manifest = await dexterity.getManifest(rpc, true, wallet);

    // Create a trader instance to interact with Dexterity
    const trader = new dexterity.Trader(manifest, TRG);

    // Connect to the trader account and log account details
    await trader.connect(NaN, async () => {
        console.log(`\nBALANCE: ${trader.getCashBalance()} | OPEN ORDERS: ${(await Promise.all(trader.getOpenOrders([PRODUCT_NAME]))).length} | POSITIONS VALUE: ${trader.getPositionValue()} | PNL: ${trader.getPnL()}`);
    });

    // Prepare order details using fractional values
    const sizeFractional = dexterity.Fractional.New(1, 0); // Replace with desired size
    const priceFractional = dexterity.Fractional.New(65, 0); // Replace with desired price

    // Define callback functions for handling order transactions
    const callbacks = {
        onGettingBlockHashFn: () => {},
        onGotBlockHashFn: () => {},
        onTxSentFn: (sig: string) => console.log(`SIGNATURE: https://solscan.io/tx/${sig}?cluster=devnet`)
    };

    // Generate the instruction for updating market prices
    const products = Array.from(dexterity.Manifest.GetProductsOfMPG(trader.mpg));
    const updateMarkIx = trader.getUpdateMarkPricesIx(products);

    // Generate the instruction for a new limit order
    const orderIx = trader.getNewOrderIx(PRODUCT_INDEX, false, priceFractional, sizeFractional, false, null, null, null, null);

    // Submit the bundled instructions
    await trader.sendTx([updateMarkIx, orderIx], callbacks);
};

submitIxBundle();
```
