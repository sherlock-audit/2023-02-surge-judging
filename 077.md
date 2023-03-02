favelanky

high

# Zero address can be set as a fee reciever with a non-zero fee.

## Summary

Zero address can be set as a fee receiver with a non-zero fee.

I mark this issue as high only because `README.md` stated "Unexpected loss of funds due to the operator role should be considered high severity bugs."

## Vulnerability Detail

By design zero address can not receive fees. It is checked in `Factory:setFeeMantissa` function: `if (_feeMantissa > 0) require(feeRecipient != address(0), "Factory: fee recipient is zero address");`. 
But if we set zero address through `Factory:setFeeRecipient` function by previously changing the fee in feeMantissa to a non-zero value, zero address will be `feeRecipient` and `_feeMantissa` will non-zero.
 
```Solidity 
function setFeeRecipient(address _feeRecipient) external {
	require(msg.sender == operator, "Factory: not operator");
	feeRecipient = _feeRecipient;
}
```

```Solidity
function setFeeMantissa(uint _feeMantissa) external {
	require(msg.sender == operator, "Factory: not operator");
	require(_feeMantissa <= MAX_FEE_MANTISSA, "Factory: fee too high");
	if (_feeMantissa > 0) require(feeRecipient != address(0), "Factory: fee recipient is zero address");
	feeMantissa = _feeMantissa;
}
```

### POC
1) Set `feeRecipient` any non zero address through `setFeeRecipient`.
2) Change `_feeMantissa` to needed value through `setFeeMantissa`.
3) Set feeRecipient to zero address  through `setFeeRecipient`.

## Impact

Zero address can receive fees.
Function `getFee` will return (0x0, feeMantissa) but it is prohibited.

## Code Snippet

https://github.com/sherlock-audit/2023-02-surge/blob/main/surge-protocol-v1/src/Factory.sol#L52-L65

## Tool used

Manual Review

## Recommendation

Adding a `require` to check for zero address.

```Solidity 
function setFeeRecipient(address _feeRecipient) external {
	require(msg.sender == operator, "Factory: not operator");
	if(_feeRecipient == address(0)) require(feeMatissa == 0, "Factory: feeMatissa is not 0");
	feeRecipient = _feeRecipient;
}
```
