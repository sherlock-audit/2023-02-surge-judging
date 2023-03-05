Tricko

high

# Fee-on-transfer ERC20 not supported

## Summary
In the README is stated that any non-rebasing ERC20 token is supported, but if a Fee-on-transfer ERC20 is chosen as collateral ou loan token, it will incur loss to the protocol.

## Vulnerability Detail
Fee-on-transfer tokens work by deducting a percentage of the transaction amount as a fee. However, [Pool](https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol) logic doesn't handle this type of token correctly, leading to loss of funds and wrong protocol functionality. 

For example, consider a Pool with a fee-on-transfer ERC20 as loan token (5% fee). Alice deposits 1000 tokens on the pool, so Alice will receive N `shares` that will be added to her balance. But during `safeTransferFrom()` only 950 tokens will be transferred to the pool due to fees. As we can see, there is a mismatch between the actual value of the `shares` alice received and how much the pool received.

This mismatch will happen on all main protocol funcionalities, when borrowing users will receive less tokens than due. On liquidations, the liquidate user debt balance can be set to 0, but not all his debt will be paid due to fees, leaving bad debt to the Pool

## Impact
Loss of funds as users will be able to withdraw more than they are due, and during liquidations and repayments, a small amount of bad debt will be left to the pool.

## Code Snippet
https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L307-L389

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L394-L398

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L414-L451

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L455-L498

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L504-L547

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Pool.sol#L553-L609

## Tool used
Manual Review

## Recommendation
Consider checking the token receiver balance before and after the transfer (see diff snippet below), and return the actual amount of tokens transferred to be used in the internal logic of `deposit`, `withdraw`, `repay`, `liquidate`, `addCollateral` and `removeCollateral`.

```diff
diff --git a/Pool.sol.orig b/Pool.sol
index 0fb9b27..922575c 100644
--- a/Pool.sol.orig
+++ b/Pool.sol
@@ -80,14 +80,20 @@ contract Pool {
         MAX_RATE = _maxRateMantissa;
     }

-    function safeTransfer(IERC20 token, address to, uint value) internal {
+    function safeTransfer(IERC20 token, address to, uint value) internal returns(uint256) {
+        uint256 balanceBefore = token.balanceOf(to);
         (bool success, bytes memory data) = address(token).call(abi.encodeWithSelector(TRANSFER_SELECTOR, to, value));
         require(success && (data.length == 0 || abi.decode(data, (bool))), 'Pool: TRANSFER_FAILED');
+        uint256 balanceAfter = token.balanceOf(to);
+        return balanceAfter - balanceBefore;
     }

-    function safeTransferFrom(IERC20 token, address from, address to, uint value) internal {
+    function safeTransferFrom(IERC20 token, address from, address to, uint value) internal returns(uint256) {
+        uint256 balanceBefore = token.balanceOf(to);
         (bool success, bytes memory data) = address(token).call(abi.encodeWithSelector(TRANSFER_FROM_SELECTOR, from, to, value));
         require(success && (data.length == 0 || abi.decode(data, (bool))), 'Pool: TRANSFER_FROM_FAILED');
+        uint256 balanceAfter = token.balanceOf(to);
+        return balanceAfter - balanceBefore;
     }

     /// @notice Gets the current state of pool variables based on the current time
```
