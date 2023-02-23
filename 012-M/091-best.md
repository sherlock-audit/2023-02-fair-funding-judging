hickuphh3

medium

# Migrator contract lacks sufficient permissions over vault positions

## Summary
The migrator contract lacks sufficient permissions over vault shares to successfully perform migration.

## Vulnerability Detail
> Since vault potentially holds an Alchemix position over a long time during which changes at Alchemix could happen, the `migration_admin` has complete control over the vault and its position after giving depositors a 30 day window to liquidate (or transfer with a flashloan) their position if they're not comfortable with the migration.

We see that all that `migrate()` does is to trigger the `migrate()` function on the migration contract. However, no permissions over the vault's shares were given to the migration contract to enable it to say, liquidate to underlying or yield tokens. It also goes against what was intended, that is, _"complete control over the vault and its position"_.

```vyper
@external
def migrate():
    """
    @notice
        Calls migrate function on the set migrator contract.
        This is just in case there are severe changes in Alchemix that
        require a full migration of the existing position.
    """
    assert self.migration_active <= block.timestamp, "migration not active"
    assert self.migration_executed == False, "migration already executed"
    self.migration_executed = True
    Migrator(self.migrator).migrate()
```

## Impact
Funds / positions cannot be successfully migrated due to lacking permissions.

## Code Snippet
https://github.com/sherlock-audit/2023-02-fair-funding/blob/main/fair-funding/contracts/Vault.vy#L545-L556

## Tool used
Manual Review

## Recommendation
In addition to invoking the `migrate()` function, consider calling `approveWithdraw()` on the migrator contract for all of the vault's shares.
https://alchemix-finance.gitbook.io/v2/docs/alchemistv2#approvewithdraw

Also consider using `raw_call()` for this function call because the current `alchemist` possibly reverts, bricking the migration process entirely.