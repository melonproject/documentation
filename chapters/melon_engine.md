# Melon Engine: buy-and-burn model

For context, please read Melonomics 2: https://medium.com/melonport-blog/melonomics-part-2-the-melon-engine-48bcb0dae65.

The Melon Engine is a contract that collects fees paid in ETH for asset management gas units by the users, buys MLN at a premium to market with the collected ETH and burns the commensurate quantity of MLN. The Melon Engine mechanisms enable the alignment of the token value with the usage of the network, while avoiding the pitfalls of high-velocity token models.

## Perspective of a user interacting with the Melon protocol

When a user interacts with an amgu (asset management gas unit) payable function on the Melon protocol, it pays asset management gas in ETH to the Melon network. We are still in the process of defining the functions that are amgu payable. As it stands, the following functions are amgu payable: `setupFund`, `requestInvestment`, `executeRequest` and `claimFees`.

A modifier `amguPayable` is added to certain functions within the Melon protocol. That modifier will apply the logic which grants the user the ability to pay asset management gas.

The total asset management gas paid is equal to: Number of amgu consumed * amgu price 

The initial amgu price will be set in MLN terms by Melonport AG before deployment in February. Thereafter, it will then be in the hands of the Melon Council to adjust the amgu price as necessary. According to our modeling, the Melon gas price (amgu price) should not need frequent adjustment. The 2 scenarios where we can expect an adjustment would be: 

- Overnight spike in network usage
- Extreme volatility on MLN/USD 

The Melon gas price should always be kept at levels which make the Melon protocol competitive to other asset management alternatives. At the same time, the costs incurred must be sufficient to incentivize maintainers and developers compensated in MLN tokens.


## Melon Engine mechanics

The ETH from the amgu fees paid by users is sent to the Melon Engine.

The Melon Engine contract is a unidirectional liquidity contract that sells ETH and buys MLN at a premium over the market price (market price is retrieved through Kyber network).

The Melon Engine maintains consecutive 1 month (60\*60\*24\*30 seconds) freeze periods. ETH received by the Melon Engine during a freeze period is held frozen in the Melon Engine until that freeze period ends. After 1 month, the function `thaw()` is publicly callable. Calling that function will render all frozen ETH liquid and available for external actors to buy ETH in exchange for MLN.

As soon as the Melon Engine sells ETH and buys MLN, it burns the acquired MLN.

The premium schedule provided by the Melon Engine is as follows:
- Ether in the Melon Engine < 1 ETH, premium = 0%
- 1≤ Ether in the Melon Engine < 5, premium = 5%
- 5 ≤ Ether in the Melon Engine < 10, premium = 10%
- Ether in the Melon Engine ≥ 10, premium = 15%


## Perspective of an actor interacting with the Melon Engine

The Melon Engine premium is exclusive to Melon funds. Any Melon fund is eligible to call the `sellAndBurnMln` function on the Melon Engine, and may therefore benefit from the premium offered.

**Use case**: As soon as ETH in the Melon Engine are liquid, a Melon fund can discretionarily sell MLN to the Melon Engine for ETH. This is done through the Melon Engine exchange adapter.

Assume a Melon fund owns 10 MLN.
- Through the Melon Engine exchange adapter, the Melon fund calls `sellAndBurnMln(10)`.
- The function retrieves the amgu price from the Melon Engine (i.e. number of ETH the Melon Engine is willing to sell for 1 MLN, premium included).
- The 10 MLN tokens the Melon fund sells are transferred to the Melon Engine contract.
- The Melon Engine sends (10 * amgu price * (1+premium)) ETH to the Melon fund in exchange for the MLN tokens sold by the Melon fund.
- The Melon Engine burns the MLN tokens collected in that transaction.
&nbsp;

## AmguConsumer.sol

#### Description

This contract defines a function modifier which is used with functions where AMG (Asset Management Gas) is designed to be charged. Placing the modifier on a function affects the calculation of the functions gas usage and resulting AMGU charge. The resulting payment and excess refund to the sender is also handled by the functionality of the modifier.  The contract also defines three helper functions to retrieve the contract addresses of the Melon Engine, MLN token and the price source.
&nbsp;

#### Inherits from

DSMath (link)

&nbsp;

#### On Construction

None.

&nbsp;

#### Structs

None.

&nbsp;

#### Enums

None.

&nbsp;

#### Modifiers

`modifier amguPayable(bool deductIncentive)`

This modifier determines the initial gas amount. The functionality in the function implementing this modifier is then executed. Thereafter, the AMGU (Asset Management Gas Unit) price and the MLN token market price is retrieved from the Melon Engine and the price feed, respectively. The modifier calculates the amount of ETH required for the payment of AMGU. Transactions with insufficient ETH amounts to pay the AMGU are reverted. ETH due as AMGU is paid to the Melon Engine and any excess ETH above the Ethereum gas costs and AMGU costs are refunded to the `msg.sender`.
&nbsp;

#### Events

None.

&nbsp;

#### Public State Variables

None.

&nbsp;

#### Public Functions

`function engine() view returns (address)`

This public view function returns the address of the Melon Engine contract.
&nbsp;

`function mlnToken() view returns (address)`

This public view function returns the address of the MLN token contract.
&nbsp;

`function priceSource() view returns (address)`

This public view function returns the address of the price feed contract.
&nbsp;

`function registry() view returns (address)`

This public view function returns the address of the registry contract.
&nbsp;

## Engine.sol

#### Description

This contract is a Melon fund exclusive, unidirectional exchange for Melon funds to exchange MLN tokens for ETH at a premium to market prices. The Melon Engine then reduces the total supply of the MLN token by burning all received MLN tokens.
&nbsp;

#### Inherits from

DSMath, DSAuth (link)

&nbsp;

#### On Construction

The constructor for this contract requires the parameters `_delay` and  `_postDeployOwner`.  The parameter `_delay`  specifies the number of delay seconds, setting the `THAWING_DELAY` state variable to this value. The constructor sets the `lastThaw` state variable to the current `block.timestamp` value. Finally, the engine contract's owner is set to the `_postDeployOwner` address provided.
&nbsp;

#### Structs

None.

&nbsp;

#### Enums

None.

&nbsp;

#### Modifiers

None.

&nbsp;

#### Events

`event RegistryChange(address registry)`

This event is triggered when the Version's Registry is set through the `setRegistry()` function, logging the `Registry` address.
&nbsp;

`event SetAmguPrice(uint amguPrice)`

This event is triggered when the `setAmguPrice()` function is called, logging the modified `amguPrice`.
&nbsp;

`event Thaw(uint amount)`

This event is triggered when the `thaw()` function is called, logging the thawed ETH `amount`.
&nbsp;

event AmguPaid(uint amount)

This event is triggered when the `payAmguInEther()` function is called, logging the ETH `amount`.
&nbsp;

event Burn(uint amount)

This event is triggered when the `sellAndBurnMln()` function is called, logging the burned MLN `amount`.
&nbsp;

#### Public State Variables

`uint public frozenEther`

This public state variable represents the quantity of ETH held by the Melon Engine which is currently frozen.
&nbsp;

`uint public liquidEther`

This public state variable represents the quantity of ETH held by the Melon Engine which is not currently frozen.
&nbsp;

`uint public lastThaw`

This public state variable represents the timestamp of the last, previous frozen ETH thaw.
&nbsp;


`uint public THAWING_DELAY`

This public state variable represents the number of seconds required before frozen ETH is thawed.
&nbsp;

`BurnableToken public mlnToken`

This public state variable represents MLN token contract which implements token burn functionality.
&nbsp;

`PriceSourceInterface public priceSource`

This public state variable represents the Version's PriceSource contract.
&nbsp;

`Registry public registry`

This public state variable represents the Version's Registry contract.
&nbsp;

`uint public MLN_DECIMALS = 18`

This public state variable defines the decimal precision of the MLN token. It is set to the constant, 18.
&nbsp;

`uint public amguPrice`

A public state variable specifying the current setting for the Asset Management Gas Unit (AMGU) price. This setting can be changed at the discretion of the Melon Technical Council (MTC).

`uint public totalEtherConsumed`

This public state variable keeps a running total of all ETH sent to the engine contract.
&nbsp;

`uint public totalAmguConsumed`

This public state variable keeps a running total of all amgu consumed by the engine contract.
&nbsp;

`uint public totalMlnBurned`

This public state variable keeps a running total of all MLN burned by the engine contract.
&nbsp;

#### Public Functions

`function setRegistry(address _registry) external auth`

This external function requires that the caller is the `owner` or the current contract. The function sets the `registry` state variable to the contract with the address specified by the parameter provided and further sets the `priceSource` and `mlnToken` state variables. Finally the function emits the `RegistryChange()` event, logging `registry`.
&nbsp;

`function setAmguPrice(uint _price) external auth`

This external function requires that the caller is the `owner` or the current contract. The function sets the `amguPrice` state variable to the value specified by the parameter provided. Finally the function emits the `SetAmguPrice()` event, logging `amguPrice`.
&nbsp;

`function getAmguPrice() public view returns (uint)`

This public view function returns the value of the `amguPrice` state variable.
&nbsp;

`function premiumPercent() public view returns (uint)`

This public view function determines the market price premium to be offered when the Melon Engine buys MLN tokens. Please see the premium schedule above. The function returns an integer representing the premium percentage, e.g. a return value of "5" represents "5%".
&nbsp;

`function payAmguInEther() external payable`

This external payable function is called while sending ETH. The function requires that the function caller must either be the fund managed by msg.sender or the fund factory used in the creation of the manager's fund. The function accepts the ETH amgu payment and increments the `frozenEther` state variable with the ETH value sent in the function call transaction. Finally, the function emits the `AmguPaid()` event, logging `amount` paid.
&nbsp;

`function thaw() external`

This external function allocates frozen ETH (`frozenEther`) to liquid ETH (`liquidEther`) after the specified `THAWING_DELAY`. The function requires that sufficient time (minimally the `THAWING_DELAY`) has passed since the previous thaw. The function then requires that the quantity of frozen ETH is positive. The function then sets the `lastThaw` state variable to the current `block.timestamp`. The `liquidEther` state variable is incremented with the `frozenEther` state variable value, the `Thaw()` event is emitted, logging `frozenEther` value and finally, the `frozenEther` state variable is set to "0". Note: this function is independently and externally called. The `lastThaw` is only reset when this function is called, regardless of whether the previous `THAWING_DELAY` has expired.
&nbsp;

`function enginePrice() public view returns (uint)`

This public view function returns the premium-adjusted ETH/MLN rate based on the quantity of ETH accumulated by the Melon Engine. The function first retrieves the MLN token price denominated in ETH. The calculated ETH-quantity scaled premium is then added to the market price and returned.
&nbsp;

`function ethPayoutForMlnAmount(uint mlnAmount) public view returns (uint)`

This public view function returns the premium-adjusted quantity of ETH to be delivered to the Melon fund in return for selling the `mlnAmount` quantity of the MLN token.
&nbsp;

`function sellAndBurnMln(uint mlnAmount) external`

This external function requires that the fund is the msg.sender, ensuring that only Melon funds may transact with the Melon Engine and benefit from the MLN token premium. The function then requires that approved MLN tokens be transferred from the Melon fund to the Melon Engine. The function calls `ethPayoutForMlnAmount()` to get the current quantity of ETH to send to the transacting Melon fund. This quantity is required to be positive and that a sufficient quantity of liquid ETH is held by the Melon Engine. The Melon Engine transfers the ETH to the transacting Melon fund and burns the received MLN tokens. Finally, the function emits the `Burn()` event, logging `mlnAmount` burned.
&nbsp;
