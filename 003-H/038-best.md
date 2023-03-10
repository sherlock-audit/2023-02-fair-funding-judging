hickuphh3

high

# Incorrect shares accounting cause liquidations to fail in some cases

## Summary
Accounting mismatch when marking claimable yield against the vault's shares may cause failing liquidations.

## Vulnerability Detail
`withdraw_underlying_to_claim()` distributes `_amount_shares` worth of underlying tokens (WETH) to token holders. Note that this burns the shares held by the vault, but for accounting purposes, the `total_shares` variable isn't updated.

However, if a token holder chooses to liquidate his shares, his `shares_owned` are used entirely in both `alchemist.liquidate()` and `withdrawUnderlying()`. Because the contract no longer has fewer shares as a result of the yield distribution, the liquidation will fail.

## POC
Refer to the `testVaultLiquidationAfterRepayment()` test case below. Note that this requires a fix to be applied for #2 first.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.18;

import "forge-std/Test.sol";
import "../../lib/utils/VyperDeployer.sol";

import "../IVault.sol";
import "../IAlchemistV2.sol";
import "../MintableERC721.sol";
import "openzeppelin/token/ERC20/IERC20.sol";

contract VaultTest is Test {
    ///@notice create a new instance of VyperDeployer
    VyperDeployer vyperDeployer = new VyperDeployer();

    FairFundingToken nft;
    IVault vault;
    address vaultAdd;
    IAlchemistV2 alchemist = IAlchemistV2(0x062Bf725dC4cDF947aa79Ca2aaCCD4F385b13b5c);
    IWhitelist whitelist = IWhitelist(0xA3dfCcbad1333DC69997Da28C961FF8B2879e653);
    address yieldToken = 0xa258C4606Ca8206D8aA700cE2143D7db854D168c;
    IERC20 weth = IERC20(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
    // pranking from big WETH holder
    address admin = 0x2fEb1512183545f48f6b9C5b4EbfCaF49CfCa6F3;
    address user1 = address(0x123);
    address user2 = address(0x456);
    
    function setUp() public {
        vm.startPrank(admin);
        nft = new FairFundingToken();
        /// @notice: I modified vault to take in admin as a parameter
        /// because of pranking issues => setting permissions
        vault = IVault(
            vyperDeployer.deployContract("Vault", abi.encode(address(nft), admin))
        );
        // to avoid having to repeatedly cast to address
        vaultAdd = address(vault);
        vault.set_alchemist(address(alchemist));

        // whitelist vault and users in Alchemist system, otherwise will run into permission issues
        vm.stopPrank();
        vm.startPrank(0x9e2b6378ee8ad2A4A95Fe481d63CAba8FB0EBBF9);
        whitelist.add(vaultAdd);
        whitelist.add(admin);
        whitelist.add(user1);
        whitelist.add(user2);
        vm.stopPrank();

        vm.startPrank(admin);

        // add depositors
        vault.add_depositor(admin);
        vault.add_depositor(user1);
        vault.add_depositor(user2);

        // check yield token is whitelisted
        assert(alchemist.isSupportedYieldToken(yieldToken));

        // mint NFTs to various parties
        nft.mint(admin, 1);
        nft.mint(user1, 2);
        nft.mint(user2, 3);
        

        // give max WETH approval to vault & alchemist
        weth.approve(vaultAdd, type(uint256).max);
        weth.approve(address(alchemist), type(uint256).max);

        // send some WETH to user1 & user2
        weth.transfer(user1, 10e18);
        weth.transfer(user2, 10e18);

        // users give WETH approval to vault and alchemist
        vm.stopPrank();
        vm.startPrank(user1);
        weth.approve(vaultAdd, type(uint256).max);
        weth.approve(address(alchemist), type(uint256).max);
        vm.stopPrank();
        vm.startPrank(user2);
        weth.approve(vaultAdd, type(uint256).max);
        weth.approve(address(alchemist), type(uint256).max);
        vm.stopPrank();

        // by default, msg.sender will be admin
        vm.startPrank(admin);
    }

    function testVaultLiquidationAfterRepayment() public {
        uint256 depositAmt = 1e18;
        // admin does a deposit
        vault.register_deposit(1, depositAmt);
        vm.stopPrank();

        // user1 does a deposit too
        vm.prank(user1);
        vault.register_deposit(2, depositAmt);

        // simulate yield: someone does partial manual repayment
        vm.prank(user2);
        alchemist.repay(address(weth), 0.1e18, vaultAdd);

        // mark it as claimable (burn a little bit more shares because of rounding)
        vault.withdraw_underlying_to_claim(
            alchemist.convertUnderlyingTokensToShares(yieldToken, 0.01e18) + 100,
            0.01e18
        );

        vm.stopPrank();

        // user1 performs liquidation, it's fine
        vm.prank(user1);
        vault.liquidate(2, 0);

        // assert that admin has more shares than what the vault holds
        (uint256 shares, ) = alchemist.positions(vaultAdd, yieldToken);
        IVault.Position memory adminPosition = vault.positions(1);
        assertGt(adminPosition.sharesOwned, shares);

        vm.prank(admin);
        // now admin is unable to liquidate because of contract doesn't hold sufficient shares
        // expect Arithmetic over/underflow error
        vm.expectRevert(stdError.arithmeticError);
        vault.liquidate(1, 0);
    }
}
```

## Impact
Failing liquidations as the contract attempts to burn more shares than it holds.

## Code Snippet
https://github.com/sherlock-audit/2023-02-fair-funding/blob/main/fair-funding/contracts/Vault.vy#L341-L349
https://github.com/sherlock-audit/2023-02-fair-funding/blob/main/fair-funding/contracts/Vault.vy#L393-L404

## Tool used
Foundry, Mainnet Forking, Manual Review

## Recommendation
For the `shares_to_liquidate` and `amount_to_withdraw` variables, check against the vault's current shares and take the minimum of the 2.

The better fix would be to switch from marking yield claims with withdrawing WETH collateral to minting debt (alETH) tokens.