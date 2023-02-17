hickuphh3

medium

# Broken Operator Mechanism: Just 1 malicious / compromised operator can permanently break functionality

## Summary
Operator access control isn't sufficiently resilient against a malicious or compromised actor.

## Vulnerability Detail
I understand that we can assume all privileged roles to be trusted, but this is about the access control structure for the vault operators. The key thing here is that you can have multiple operators who can add or remove each other. As the saying goes, _"you are as strong as your weakest link"_, so all it required is for 1 malicious or compromised operator to permanently break protocol functionality, with no possible remediation as he's able to kick out all other honest operators, _including himself_

The vault operator can do the following:
1) Set the `alchemist` contract to any address (except null) of his choosing. He can therefore permanently brick the claiming and liquidation process, resulting in the permanent locking of token holders' funds in Alchemix.
2) Steal last auction funds. WETH approval is given to the `alchemist` contract every time `register_deposit` is called, and with the fact that anyone can settle the contract, the malicious operator is able to do the following atomically:
    - set the alchemist contract to a malicious implementation
       - contract returns a no-op + arbitrary `shares_issued` value when the `depositUnderlying()` function is called
    - settle the last auction (assuming it hasn't been)
    - pull auction funds from approval given
3) Do (1) and remove himself as an operator (ie. there are no longer any operators), permanently preventing any possible remediation.

## Impact
DoS / holding the users' funds hostage.

## Code Snippet
https://github.com/sherlock-audit/2023-02-fair-funding/blob/main/fair-funding/contracts/Vault.vy#L292-L300
https://github.com/sherlock-audit/2023-02-fair-funding/blob/main/fair-funding/contracts/Vault.vy#L589-L614

## Tool used
Manual Review

## Recommendation
Add an additional access control layer on top of operators: an `owner` that will be held by a multisig / DAO that's able to add / remove operators. 