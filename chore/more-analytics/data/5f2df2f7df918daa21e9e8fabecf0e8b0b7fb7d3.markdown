# Test info

- Name: Transaction signing >> App initiated STX transfer >> that it broadcasts correctly with given fee and amount
- Location: /home/runner/work/extension/extension/tests/specs/transactions/transactions.spec.ts:55:5

# Error details

```
Error: page.click: Target page, context or browser has been closed
Call log:
  - waiting for locator('text=Debugger')

    at /home/runner/work/extension/extension/tests/specs/transactions/transactions.spec.ts:63:30
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
   39 |       await testAppPage.page.click('text=Debugger', {
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
>  63 |       await testAppPage.page.click('text=Debugger', {
      |                              ^ Error: page.click: Target page, context or browser has been closed
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
diff --git a/src/app/common/transactions/stacks/stacks-broadcast-transaction.ts b/src/app/common/transactions/stacks/stacks-broadcast-transaction.ts
index 52d96397c..e31c171d2 100644
--- a/src/app/common/transactions/stacks/stacks-broadcast-transaction.ts
+++ b/src/app/common/transactions/stacks/stacks-broadcast-transaction.ts
@@ -10,6 +10,7 @@ import { delay, isError } from '@leather.io/utils';
 import { logger } from '@shared/logger';
 import { analytics } from '@shared/utils/analytics';
 
+import { serializeError } from '@app/common/utils';
 import { hiroFetchWrapper } from '@app/query/stacks/stacks-client';
 
 interface StacksBroadcastTransactionArgs {
@@ -40,12 +41,19 @@ export async function stacksBroadcastTransaction({
 
     if ('error' in response) {
       logger.error('Transaction failed to broadcast', response);
-      throw new Error(response.error);
+      const error = new Error(response.error);
+      error.name = response.reason;
+      error.message = response.error;
+      error.cause = response.reason;
+      throw error;
     }
 
     if (!response.txid) {
       logger.error('Transaction failed to broadcast', response);
-      throw new Error('Transaction failed to broadcast');
+      const error = new Error('Transaction broadcast but returned no txid');
+      error.name = 'TransactionBroadcastError';
+      error.message = 'Transaction broadcast but returned no txid';
+      throw error;
     }
 
     logger.info('Transaction broadcast', response);
@@ -62,7 +70,7 @@ export async function stacksBroadcastTransaction({
     logger.error('Transaction error', { error });
     void analytics.untypedTrack('stacks_transaction_broadcast_failed', {
       attemptId,
-      error: isError(error) ? error.message : String(error),
+      ...serializeError(error),
     });
     onError(isError(error) ? error : { name: '', message: '' });
     return;
diff --git a/src/app/common/utils.ts b/src/app/common/utils.ts
index 1968980ee..0fb9e4ba0 100644
--- a/src/app/common/utils.ts
+++ b/src/app/common/utils.ts
@@ -267,3 +267,22 @@ export function removeTrailingNullCharacters(s: string) {
 export function removeMinusSign(value: string) {
   return value.replace('-', '');
 }
+
+export function serializeError(err: unknown) {
+  if (err instanceof Error) {
+    const errorObj: Record<string, any> = {
+      name: err.name,
+      message: err.message,
+      stack: err.stack,
+    };
+
+    // Include any custom enumerable properties
+    for (const key of Object.keys(err)) {
+      errorObj[key] = (err as any)[key];
+    }
+
+    return errorObj;
+  }
+
+  return { message: String(err) };
+}
```