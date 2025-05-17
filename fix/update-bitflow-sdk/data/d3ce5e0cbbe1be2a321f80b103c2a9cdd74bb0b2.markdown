# Test info

- Name: Transaction signing >> Contract calls >> that it validates against insufficient funds when performing a contract call
- Location: /home/runner/work/extension/extension/tests/specs/transactions/transactions.spec.ts:27:5

# Error details

```
Error: page.click: Target page, context or browser has been closed
Call log:
  - waiting for locator('text=Debugger')

    at /home/runner/work/extension/extension/tests/specs/transactions/transactions.spec.ts:39:30
```

# Test source

```ts
   1 | import { TokenTransferPayloadWire, deserializeTransaction } from '@stacks/transactions';
   2 | import { TestAppPage } from '@tests/page-object-models/test-app.page';
   3 | import { TransactionRequestPage } from '@tests/page-object-models/transaction-request.page';
   4 | import { OnboardingSelectors } from '@tests/selectors/onboarding.selectors';
   5 |
   6 | import { stxToMicroStx } from '@leather.io/utils';
   7 |
   8 | import { createDelay } from '@shared/utils';
   9 |
   10 | import { test } from '../../fixtures/fixtures';
   11 |
   12 | const delayAnimationDuration = createDelay(2000);
   13 |
   14 | test.describe('Transaction signing', () => {
   15 |   let testAppPage: TestAppPage;
   16 |
   17 |   test.beforeEach(async ({ extensionId, globalPage, onboardingPage, context }) => {
   18 |     await globalPage.setupAndUseApiCalls(extensionId);
   19 |     await onboardingPage.signInWithTestAccount(extensionId);
   20 |     testAppPage = await TestAppPage.openDemoPage(context);
   21 |   });
   22 |
   23 |   // These tests often break if ran in parallel
   24 |   test.describe.configure({ mode: 'serial' });
   25 |
   26 |   test.describe('Contract calls', () => {
   27 |     test('that it validates against insufficient funds when performing a contract call', async ({
   28 |       context,
   29 |     }) => {
   30 |       const newPagePromise = context.waitForEvent('page');
   31 |       await testAppPage.page.getByTestId(OnboardingSelectors.SignUpBtn).click();
   32 |       const accountsPage = await newPagePromise;
   33 |       await accountsPage.getByTestId('switch-account-item-0').click({ force: true });
   34 |       await accountsPage.getByTestId('switch-account-item-1').click({ force: true });
   35 |       await delayAnimationDuration();
   36 |       await accountsPage.getByRole('button').getByText('Confirm').click({ force: true });
   37 |       await delayAnimationDuration();
   38 |       await testAppPage.page.bringToFront();
>  39 |       await testAppPage.page.click('text=Debugger', {
      |                              ^ Error: page.click: Target page, context or browser has been closed
   40 |         timeout: 30000,
   41 |       });
   42 |       await accountsPage.close();
   43 |
   44 |       await testAppPage.clickContractCallButton();
   45 |       const transactionRequestPage = new TransactionRequestPage(await context.waitForEvent('page'));
   46 |       await transactionRequestPage.waitForTransactionRequestPage();
   47 |       const error =
   48 |         await transactionRequestPage.waitForTransactionRequestError('Insufficient balance');
   49 |
   50 |       test.expect(error).toBeTruthy();
   51 |     });
   52 |   });
   53 |
   54 |   test.describe('App initiated STX transfer', () => {
   55 |     test('that it broadcasts correctly with given fee and amount', async ({ context }) => {
   56 |       const newPagePromise = context.waitForEvent('page');
   57 |       await testAppPage.page.getByTestId(OnboardingSelectors.SignUpBtn).click();
   58 |       const accountsPage = await newPagePromise;
   59 |       await delayAnimationDuration();
   60 |       await accountsPage.getByRole('button').getByText('Confirm').click({ force: true });
   61 |       await delayAnimationDuration();
   62 |       await testAppPage.page.bringToFront();
   63 |       await testAppPage.page.click('text=Debugger', {
   64 |         timeout: 30000,
   65 |       });
   66 |       await accountsPage.close();
   67 |
   68 |       await testAppPage.clickStxTransferButton();
   69 |       const transactionRequestPage = new TransactionRequestPage(await context.waitForEvent('page'));
   70 |
   71 |       await transactionRequestPage.waitForTransactionRequestPage();
   72 |       await transactionRequestPage.waitForFee();
   73 |
   74 |       const displayedFee = await transactionRequestPage.getDisplayedFeeValue();
   75 |
   76 |       if (!displayedFee) throw new Error('Cannot pull fee from UI');
   77 |
   78 |       const requestPromise = transactionRequestPage.page.waitForRequest('*/**/v2/transactions');
   79 |
   80 |       await transactionRequestPage.page.route('*/**/v2/transactions', async route => {
   81 |         await route.abort();
   82 |       });
   83 |
   84 |       await transactionRequestPage.clickConfirmTransactionButton();
   85 |
   86 |       const request = await requestPromise;
   87 |       const requestBody = request.postData();
   88 |       if (!requestBody) return;
   89 |
   90 |       const deserializedTx = deserializeTransaction(JSON.parse(requestBody).tx);
   91 |       const payload = deserializedTx.payload as TokenTransferPayloadWire;
   92 |       const amount = Number(payload.amount);
   93 |       const fee = Number(deserializedTx.auth.spendingCondition?.fee);
   94 |       const parsedDisplayedFee = parseFloat(displayedFee.replace(' STX', ''));
   95 |
   96 |       test.expect(fee).toEqual(stxToMicroStx(parsedDisplayedFee).toNumber());
   97 |       test.expect(amount).toEqual(102);
   98 |     });
   99 |   });
  100 | });
  101 |
```

# Local changes

```diff
diff --git a/.env.example b/.env.example
index 69061d2ea..59806cbdd 100644
--- a/.env.example
+++ b/.env.example
@@ -7,5 +7,6 @@ WALLET_ENVIRONMENT=development
 BESTINSLOT_API_KEY=bestinslotapikey
 BITFLOW_API_HOST=bitflowapihost
 BITFLOW_API_KEY=bitflowapikey
-BITFLOW_STACKS_API_HOST=bitflowstacksapihost
-BITFLOW_READONLY_CALL_API_HOST=bitflowreadonlycallapihost
\ No newline at end of file
+BITFLOW_READONLY_CALL_API_HOST=bitflowreadonlycallapihost
+BITFLOW_KEEPER_API_KEY=bitflowkeeperapikey
+BITFLOW_KEEPER_API_HOST=bitflowkeeperapihost
diff --git a/.github/workflows/build-extension.yml b/.github/workflows/build-extension.yml
index 8ee9d58a1..01c4e5133 100644
--- a/.github/workflows/build-extension.yml
+++ b/.github/workflows/build-extension.yml
@@ -57,8 +57,9 @@ jobs:
           BESTINSLOT_API_KEY: ${{ secrets.BESTINSLOT_API_KEY }}
           BITFLOW_API_HOST: ${{ secrets.BITFLOW_API_HOST }}
           BITFLOW_API_KEY: ${{ secrets.BITFLOW_API_KEY }}
-          BITFLOW_STACKS_API_HOST: ${{ secrets.BITFLOW_STACKS_API_HOST }}
           BITFLOW_READONLY_CALL_API_HOST: ${{ secrets.BITFLOW_READONLY_CALL_API_HOST }}
+          BITFLOW_KEEPER_API_KEY: ${{ secrets.BITFLOW_KEEPER_API_KEY }}
+          BITFLOW_KEEPER_API_HOST: ${{ secrets.BITFLOW_KEEPER_API_HOST }}
           PR_NUMBER: ${{ github.event.number }}
           COMMIT_SHA: ${{ needs.sha-hash.outputs.SHORT_SHA }}
 
diff --git a/.github/workflows/development-extension.yml b/.github/workflows/development-extension.yml
index 83de5a635..02bf45093 100644
--- a/.github/workflows/development-extension.yml
+++ b/.github/workflows/development-extension.yml
@@ -15,8 +15,9 @@ env:
   BESTINSLOT_API_KEY: ${{ secrets.BESTINSLOT_API_KEY }}
   BITFLOW_API_HOST: ${{ secrets.BITFLOW_API_HOST }}
   BITFLOW_API_KEY: ${{ secrets.BITFLOW_API_KEY }}
-  BITFLOW_STACKS_API_HOST: ${{ secrets.BITFLOW_STACKS_API_HOST }}
   BITFLOW_READONLY_CALL_API_HOST: ${{ secrets.BITFLOW_READONLY_CALL_API_HOST }}
+  BITFLOW_KEEPER_API_KEY: ${{ secrets.BITFLOW_KEEPER_API_KEY }}
+  BITFLOW_KEEPER_API_HOST: ${{ secrets.BITFLOW_KEEPER_API_HOST }}
   PREVIEW_RELEASE: true
   WALLET_ENVIRONMENT: preview
 
diff --git a/.github/workflows/playwright.yml b/.github/workflows/playwright.yml
index 356f7348d..61baf3ab6 100644
--- a/.github/workflows/playwright.yml
+++ b/.github/workflows/playwright.yml
@@ -5,8 +5,9 @@ env:
   WALLET_ENVIRONMENT: testing
   BITFLOW_API_HOST: ${{ secrets.BITFLOW_API_HOST }}
   BITFLOW_API_KEY: ${{ secrets.BITFLOW_API_KEY }}
-  BITFLOW_STACKS_API_HOST: ${{ secrets.BITFLOW_STACKS_API_HOST }}
   BITFLOW_READONLY_CALL_API_HOST: ${{ secrets.BITFLOW_READONLY_CALL_API_HOST }}
+  BITFLOW_KEEPER_API_KEY: ${{ secrets.BITFLOW_KEEPER_API_KEY }}
+  BITFLOW_KEEPER_API_HOST: ${{ secrets.BITFLOW_KEEPER_API_HOST }}
 
 on:
   push:
diff --git a/.github/workflows/publish-extensions.yml b/.github/workflows/publish-extensions.yml
index 91d5e7a19..3d8ffcb42 100644
--- a/.github/workflows/publish-extensions.yml
+++ b/.github/workflows/publish-extensions.yml
@@ -15,8 +15,9 @@ env:
   BESTINSLOT_API_KEY: ${{ secrets.BESTINSLOT_API_KEY }}
   BITFLOW_API_HOST: ${{ secrets.BITFLOW_API_HOST }}
   BITFLOW_API_KEY: ${{ secrets.BITFLOW_API_KEY }}
-  BITFLOW_STACKS_API_HOST: ${{ secrets.BITFLOW_STACKS_API_HOST }}
   BITFLOW_READONLY_CALL_API_HOST: ${{ secrets.BITFLOW_READONLY_CALL_API_HOST }}
+  BITFLOW_KEEPER_API_KEY: ${{ secrets.BITFLOW_KEEPER_API_KEY }}
+  BITFLOW_KEEPER_API_HOST: ${{ secrets.BITFLOW_KEEPER_API_HOST }}
   WALLET_ENVIRONMENT: production
   IS_PUBLISHING: true
 
diff --git a/package.json b/package.json
index c2daaaaac..3c0c30ef2 100644
--- a/package.json
+++ b/package.json
@@ -140,6 +140,7 @@
   },
   "dependencies": {
     "@bitcoinerlab/secp256k1": "1.0.2",
+    "@bitflowlabs/core-sdk": "2.2.0",
     "@blockstack/stacks-transactions": "0.7.0",
     "@coinbase/cbpay-js": "2.1.0",
     "@fungible-systems/zone-file": "2.0.0",
@@ -203,7 +204,6 @@
     "bignumber.js": "9.1.2",
     "bitcoin-address-validation": "2.2.1",
     "bitcoinjs-lib": "6.1.5",
-    "bitflow-sdk": "1.6.1",
     "bn.js": "5.2.1",
     "browserify-fs": "1.0.0",
     "c32check": "2.0.0",
diff --git a/pnpm-lock.yaml b/pnpm-lock.yaml
index 635befa01..cfd0c7403 100644
--- a/pnpm-lock.yaml
+++ b/pnpm-lock.yaml
@@ -30,6 +30,9 @@ importers:
       '@bitcoinerlab/secp256k1':
         specifier: 1.0.2
         version: 1.0.2
+      '@bitflowlabs/core-sdk':
+        specifier: 2.2.0
+        version: 2.2.0(encoding@0.1.13)
       '@blockstack/stacks-transactions':
         specifier: 0.7.0
         version: 0.7.0(encoding@0.1.13)
@@ -219,9 +222,6 @@ importers:
       bitcoinjs-lib:
         specifier: 6.1.5
         version: 6.1.5
-      bitflow-sdk:
-        specifier: 1.6.1
-        version: 1.6.1(encoding@0.1.13)
       bn.js:
         specifier: 5.2.1
         version: 5.2.1
@@ -1597,6 +1597,10 @@ packages:
   '@bitcoinerlab/secp256k1@1.2.0':
     resolution: {integrity: sha512-jeujZSzb3JOZfmJYI0ph1PVpCRV5oaexCgy+RvCXV8XlY+XFB/2n3WOcvBsKLsOw78KYgnQrQWb2HrKE4be88Q==}
 
+  '@bitflowlabs/core-sdk@2.2.0':
+    resolution: {integrity: sha512-i/gcnVyM5A+c98SPgQcKoVsRCKW9q7KYKuls/LHYkEo0sGkPH83qQn6d1dMe6ALsHvkixlJmfosT+qG6gM4hhg==}
+    engines: {node: '>=20.0.0'}
+
   '@blockstack/stacks-transactions@0.7.0':
     resolution: {integrity: sha512-9IUR641PJpigYDCWdtKFgBIk0Oxyoc2M2XVdyaEOEcRhI+4lcacDhsuseAmGbkB9FhJgJcNicWPujUPNFl1XCw==}
 
@@ -2587,7 +2591,7 @@ packages:
 
   '@expo/bunyan@4.0.1':
     resolution: {integrity: sha512-+Lla7nYSiHZirgK+U/uYzsLv/X+HaJienbD5AKX1UQZHYfWaP+9uuQluRB4GrEVWF0GZ7vEVp/jzaOT9k/SQlg==}
-    engines: {node: '>=0.10.0'}
+    engines: {'0': node >=0.10.0}
 
   '@expo/cli@0.18.28':
     resolution: {integrity: sha512-fvbVPId6s6etindzP6Nzos/CS1NurMVy4JKozjebArHr63tBid5i/UY5Pp+4wTCAM20gB2SjRdwcwoL6HFC4Iw==}
@@ -2927,7 +2931,6 @@ packages:
 
   '@ls-lint/ls-lint@2.2.3':
     resolution: {integrity: sha512-ekM12jNm/7O2I/hsRv9HvYkRdfrHpiV1epVuI2NP+eTIcEgdIdKkKCs9KgQydu/8R5YXTov9aHdOgplmCHLupw==}
-    cpu: [x64, arm64, s390x]
     os: [darwin, linux, win32]
     hasBin: true
 
@@ -4993,6 +4996,12 @@ packages:
   '@stacks/connect-ui@6.1.1':
     resolution: {integrity: sha512-iSo57djIynmqt0jGlFkRFu2nHY/Nk0LmXKdRf/Whw1w/YbZD+CQJweHRh77XQOtAVbXZ1+e/klszxABevcPtPg==}
 
+  '@stacks/connect-ui@6.6.0':
+    resolution: {integrity: sha512-uc22RH99umYzB94h5LiKPtGu34IBGrwUb3TfijGb2ZMudaMCiv/Fr1jjZKfQW5MRmexnbAEmGZpFlQKinCcsUA==}
+
+  '@stacks/connect@7.10.2':
+    resolution: {integrity: sha512-fQcdayBgq9XZnX4rqQxa//Gx9c0ycrmrZT9dZ01uHDlIr/ZxwU18d5A3hyYv4F7LQYQQkFr9htpVTlH0RSqWUw==}
+
   '@stacks/connect@7.4.0':
     resolution: {integrity: sha512-2jhTHL6Wi7Y/B1AwUuumUUE5F+/X7AvtbJ3BzsNVP7yB+yswmtjC3ZO3jYEohBcuAay5ysfNWUYdjfiXvp0NDQ==}
 
@@ -6877,10 +6886,6 @@ packages:
     resolution: {integrity: sha512-yuf6xs9QX/E8LWE2aMJPNd0IxGofwfuVOiYdNUESkc+2bHHVKjhJd8qewqapeoolh9fihzHGoDCB5Vkr57RZCQ==}
     engines: {node: '>=8.0.0'}
 
-  bitflow-sdk@1.6.1:
-    resolution: {integrity: sha512-V6TUstTBNorR+WadWSaiKEjNXy25unIAICLtrUXTZMrJtOfUIfUZx7W7hWmoUjBE9k8z/dOfjH1HUbF0wRxWZQ==}
-    engines: {node: '>=14.0.0'}
-
   bl@0.8.2:
     resolution: {integrity: sha512-pfqikmByp+lifZCS0p6j6KreV6kNU6Apzpm2nKOk+94cZb/jvle55+JxWiByUQ0Wo/+XnDXEy5MxxKMb6r0VIw==}
 
@@ -8252,6 +8257,10 @@ packages:
     resolution: {integrity: sha512-ZmdL2rui+eB2YwhsWzjInR8LldtZHGDoQ1ugH85ppHKwpUHL7j7rN0Ti9NCnGiQbhaZ11FpR+7ao1dNsmduNUg==}
     engines: {node: '>=12'}
 
+  dotenv@16.5.0:
+    resolution: {integrity: sha512-m/C+AwOAr9/W1UOIZUo232ejMNnJAJtYQjUbHoNTBNTJSvqzzDh7vnrei3o3r3m9blf6ZoDkvcw0VmozNRFJxg==}
+    engines: {node: '>=12'}
+
   dotenv@8.6.0:
     resolution: {integrity: sha512-IrPdXQsk2BbzvCBGBOTmmSH5SodmqZNt4ERAZDmW4CT+tL8VtvinqywuANaFu4bOMWki16nqf0e4oC0QIaDr/g==}
     engines: {node: '>=10'}
@@ -16156,6 +16165,15 @@ snapshots:
     dependencies:
       '@noble/curves': 1.9.0
 
+  '@bitflowlabs/core-sdk@2.2.0(encoding@0.1.13)':
+    dependencies:
+      '@stacks/connect': 7.10.2(encoding@0.1.13)
+      '@stacks/network': 7.0.2(encoding@0.1.13)
+      '@stacks/transactions': 7.0.5(encoding@0.1.13)
+      dotenv: 16.5.0
+    transitivePeerDependencies:
+      - encoding
+
   '@blockstack/stacks-transactions@0.7.0(encoding@0.1.13)':
     dependencies:
       '@types/bn.js': 4.11.6
@@ -20973,6 +20991,24 @@ snapshots:
     dependencies:
       '@stencil/core': 2.22.3
 
+  '@stacks/connect-ui@6.6.0':
+    dependencies:
+      '@stencil/core': 2.22.3
+
+  '@stacks/connect@7.10.2(encoding@0.1.13)':
+    dependencies:
+      '@stacks/auth': 7.0.2(encoding@0.1.13)
+      '@stacks/common': 7.0.2
+      '@stacks/connect-ui': 6.6.0
+      '@stacks/network': 7.0.2(encoding@0.1.13)
+      '@stacks/network-v6': '@stacks/network@6.17.0(encoding@0.1.13)'
+      '@stacks/profile': 7.0.2(encoding@0.1.13)
+      '@stacks/transactions': 7.0.5(encoding@0.1.13)
+      '@stacks/transactions-v6': '@stacks/transactions@6.17.0(encoding@0.1.13)'
+      jsontokens: 4.0.1
+    transitivePeerDependencies:
+      - encoding
+
   '@stacks/connect@7.4.0(encoding@0.1.13)':
     dependencies:
       '@stacks/auth': 6.17.0(encoding@0.1.13)
@@ -23470,14 +23506,6 @@ snapshots:
       typeforce: 1.18.0
       varuint-bitcoin: 1.1.2
 
-  bitflow-sdk@1.6.1(encoding@0.1.13):
-    dependencies:
-      '@stacks/network': 6.13.0(encoding@0.1.13)
-      '@stacks/transactions': 6.17.0(encoding@0.1.13)
-      dotenv: 16.4.5
-    transitivePeerDependencies:
-      - encoding
-
   bl@0.8.2:
     dependencies:
       readable-stream: 1.0.34
@@ -25023,6 +25051,8 @@ snapshots:
 
   dotenv@16.4.5: {}
 
+  dotenv@16.5.0: {}
+
   dotenv@8.6.0: {}
 
   dset@3.1.4: {}
diff --git a/src/app/pages/swap/components/swap-details/swap-details.utils.ts b/src/app/pages/swap/components/swap-details/swap-details.utils.ts
index f89e2f43d..4ccc7caff 100644
--- a/src/app/pages/swap/components/swap-details/swap-details.utils.ts
+++ b/src/app/pages/swap/components/swap-details/swap-details.utils.ts
@@ -1,5 +1,5 @@
+import type { RouteQuote } from '@bitflowlabs/core-sdk';
 import BigNumber from 'bignumber.js';
-import type { RouteQuote } from 'bitflow-sdk';
 
 import { capitalize, isDefined } from '@leather.io/utils';
 
diff --git a/src/app/pages/swap/hooks/use-bitflow-swappable-assets.tsx b/src/app/pages/swap/hooks/use-bitflow-swappable-assets.tsx
index c94d1dfb7..c98a3616c 100644
--- a/src/app/pages/swap/hooks/use-bitflow-swappable-assets.tsx
+++ b/src/app/pages/swap/hooks/use-bitflow-swappable-assets.tsx
@@ -1,8 +1,8 @@
 import { useCallback } from 'react';
 
+import type { Token } from '@bitflowlabs/core-sdk';
 import { useQuery } from '@tanstack/react-query';
 import BigNumber from 'bignumber.js';
-import type { Token } from 'bitflow-sdk';
 
 import { type Currency, createMarketData, createMarketPair } from '@leather.io/models';
 import { getPrincipalFromAssetString } from '@leather.io/stacks';
diff --git a/src/app/pages/swap/providers/stacks-swap-provider.tsx b/src/app/pages/swap/providers/stacks-swap-provider.tsx
index f5ef0ad33..36bb16da8 100644
--- a/src/app/pages/swap/providers/stacks-swap-provider.tsx
+++ b/src/app/pages/swap/providers/stacks-swap-provider.tsx
@@ -1,7 +1,7 @@
 import { Outlet } from 'react-router-dom';
 
+import type { RouteQuote } from '@bitflowlabs/core-sdk';
 import type { StacksTransactionWire } from '@stacks/transactions';
-import type { RouteQuote } from 'bitflow-sdk';
 
 import { defaultSwapFee } from '@leather.io/query';
 
diff --git a/src/app/pages/swap/providers/use-stacks-swap.tsx b/src/app/pages/swap/providers/use-stacks-swap.tsx
index 505c8e5da..b2a193667 100644
--- a/src/app/pages/swap/providers/use-stacks-swap.tsx
+++ b/src/app/pages/swap/providers/use-stacks-swap.tsx
@@ -1,8 +1,8 @@
 import { useCallback } from 'react';
 import { useNavigate } from 'react-router-dom';
 
+import type { RouteQuote } from '@bitflowlabs/core-sdk';
 import { PostConditionMode, serializeCV } from '@stacks/transactions';
-import type { RouteQuote } from 'bitflow-sdk';
 
 import { TransactionTypes, getPostConditions } from '@leather.io/stacks';
 import { isError, isUndefined } from '@leather.io/utils';
@@ -10,11 +10,7 @@ import { isError, isUndefined } from '@leather.io/utils';
 import { logger } from '@shared/logger';
 import { RouteUrls } from '@shared/route-urls';
 import { bitflow } from '@shared/utils/bitflow-sdk';
-import {
-  type ContractCallPayload,
-  legacyCVToCV,
-  serializeLegacyPostCondition,
-} from '@shared/utils/legacy-requests';
+import { type ContractCallPayload } from '@shared/utils/legacy-requests';
 
 import { LoadingKeys, useLoading } from '@app/common/hooks/use-loading';
 import {
@@ -113,9 +109,9 @@ export function useStacksSwap(nonce: number | string) {
         contractAddress: swapParams.contractAddress,
         contractName: swapParams.contractName,
         functionName: swapParams.functionName,
-        functionArgs: swapParams.functionArgs.map(arg => serializeCV(legacyCVToCV(arg))),
+        functionArgs: swapParams.functionArgs.map(arg => serializeCV(arg)),
         postConditionMode: PostConditionMode.Deny,
-        postConditions: getPostConditions(serializeLegacyPostCondition(swapParams.postConditions)),
+        postConditions: getPostConditions(swapParams.postConditions),
         publicKey: currentAccount?.stxPublicKey,
         sponsored: false,
         network,
diff --git a/src/shared/environment.ts b/src/shared/environment.ts
index d3d4da193..81c6ce9bc 100644
--- a/src/shared/environment.ts
+++ b/src/shared/environment.ts
@@ -16,6 +16,7 @@ export const WALLET_ENVIRONMENT = process.env.WALLET_ENVIRONMENT ?? 'unknown';
 export const SWAP_ENABLED = process.env.SWAP_ENABLED === 'true';
 export const BITFLOW_API_HOST = process.env.BITFLOW_API_HOST ?? '';
 export const BITFLOW_API_KEY = process.env.BITFLOW_API_KEY ?? '';
-export const BITFLOW_STACKS_API_HOST = process.env.BITFLOW_STACKS_API_HOST ?? '';
 export const BITFLOW_READONLY_CALL_API_HOST = process.env.BITFLOW_READONLY_CALL_API_HOST ?? '';
+export const BITFLOW_KEEPER_API_KEY = process.env.BITFLOW_KEEPER_API_KEY ?? '';
+export const BITFLOW_KEEPER_API_HOST = process.env.BITFLOW_KEEPER_API_HOST ?? '';
 export const DEBUG_TX_MONITOR = process.env.DEBUG_TX_MONITOR === 'true';
diff --git a/src/shared/utils/bitflow-sdk.ts b/src/shared/utils/bitflow-sdk.ts
index faa3f54db..6eca5d2e2 100644
--- a/src/shared/utils/bitflow-sdk.ts
+++ b/src/shared/utils/bitflow-sdk.ts
@@ -1,20 +1,22 @@
-import { BitflowSDK } from 'bitflow-sdk';
+import { BitflowSDK } from '@bitflowlabs/core-sdk';
 
 import {
   BITFLOW_API_HOST,
   BITFLOW_API_KEY,
+  BITFLOW_KEEPER_API_HOST,
+  BITFLOW_KEEPER_API_KEY,
   BITFLOW_READONLY_CALL_API_HOST,
-  BITFLOW_STACKS_API_HOST,
 } from '@shared/environment';
 import { logger } from '@shared/logger';
 
 export const bitflow: BitflowSDK = (() => {
   try {
     return new BitflowSDK({
-      API_HOST: BITFLOW_API_HOST,
-      API_KEY: BITFLOW_API_KEY,
-      STACKS_API_HOST: BITFLOW_STACKS_API_HOST,
+      BITFLOW_API_HOST: BITFLOW_API_HOST,
+      BITFLOW_API_KEY: BITFLOW_API_KEY,
       READONLY_CALL_API_HOST: BITFLOW_READONLY_CALL_API_HOST,
+      KEEPER_API_KEY: BITFLOW_KEEPER_API_KEY,
+      KEEPER_API_HOST: BITFLOW_KEEPER_API_HOST,
     });
   } catch (e) {
     logger.error('Bitflow SDK initialization failed');
diff --git a/src/shared/utils/legacy-requests.ts b/src/shared/utils/legacy-requests.ts
index 1888fb041..46067e8e8 100644
--- a/src/shared/utils/legacy-requests.ts
+++ b/src/shared/utils/legacy-requests.ts
@@ -1,4 +1,3 @@
-import { bytesToHex } from '@stacks/common';
 import {
   CommonSignaturePayload as ConnectCommonSignaturePayload,
   ContractCallPayload as ConnectContractCallPayload,
@@ -7,19 +6,7 @@ import {
   STXTransferPayload as ConnectSTXTransferPayload,
 } from '@stacks/connect-jwt';
 import type { StacksNetwork } from '@stacks/network';
-import {
-  Address,
-  Cl,
-  type ClarityValue,
-  type PostCondition,
-  type PostConditionWire,
-} from '@stacks/transactions';
-import {
-  ClarityType as LegacyClarityType,
-  ClarityValue as LegacyClarityValue,
-  PostCondition as LegacyPostCondition,
-  serializePostCondition as legacySerializePostCondition,
-} from '@stacks/transactions-v6';
+import { type PostCondition, type PostConditionWire } from '@stacks/transactions';
 import { decodeToken } from 'jsontokens';
 
 import type { ReplaceTypes } from '@leather.io/models';
@@ -72,51 +59,3 @@ export function getLegacyTransactionPayloadFromToken(requestToken: string) {
   const token = decodeToken(requestToken);
   return token.payload as unknown as TransactionPayload;
 }
-
-export function legacyCVToCV(cv: LegacyClarityValue): ClarityValue {
-  if (typeof cv.type === 'string') return cv;
-
-  switch (cv.type) {
-    case LegacyClarityType.BoolFalse:
-      return Cl.bool(false);
-    case LegacyClarityType.BoolTrue:
-      return Cl.bool(true);
-    case LegacyClarityType.Int:
-      return Cl.int(cv.value);
-    case LegacyClarityType.UInt:
-      return Cl.uint(cv.value);
-    case LegacyClarityType.Buffer:
-      return Cl.buffer(cv.buffer);
-    case LegacyClarityType.StringASCII:
-      return Cl.stringAscii(cv.data);
-    case LegacyClarityType.StringUTF8:
-      return Cl.stringUtf8(cv.data);
-    case LegacyClarityType.List:
-      return Cl.list(cv.list.map(legacyCVToCV));
-    case LegacyClarityType.Tuple:
-      return Cl.tuple(
-        Object.fromEntries(Object.entries(cv.data).map(([k, v]) => [k, legacyCVToCV(v)]))
-      );
-    case LegacyClarityType.OptionalNone:
-      return Cl.none();
-    case LegacyClarityType.OptionalSome:
-      return Cl.some(legacyCVToCV(cv.value));
-    case LegacyClarityType.ResponseErr:
-      return Cl.error(legacyCVToCV(cv.value));
-    case LegacyClarityType.ResponseOk:
-      return Cl.ok(legacyCVToCV(cv.value));
-    case LegacyClarityType.PrincipalContract:
-      return Cl.contractPrincipal(Address.stringify(cv.address), cv.contractName.content);
-    case LegacyClarityType.PrincipalStandard:
-      return Cl.standardPrincipal(Address.stringify(cv.address));
-    default:
-      const _exhaustiveCheck: never = cv;
-      throw new Error(`Unknown clarity type: ${_exhaustiveCheck}`);
-  }
-}
-
-export function serializeLegacyPostCondition(pcs: LegacyPostCondition[]) {
-  return pcs.map(pc => {
-    return bytesToHex(legacySerializePostCondition(pc));
-  });
-}
```