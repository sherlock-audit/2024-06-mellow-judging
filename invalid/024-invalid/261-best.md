Curved Powder Rooster

Medium

# Admin role cannot grant/revoke the operator role

## Summary
Admin role cannot grant/revoke the operator role

## Vulnerability Detail
The admin role cannot grant/revoke operator roles. This is different from the documentation which expects the admin role to be able to [assign/resign operator](https://mellowprotocol.notion.site/Role-model-and-actors-90898e3f3af84094922ca6c79d05ac03)

[link](https://github.com/sherlock-audit/2024-06-mellow/blob/26aa0445ec405a4ad637bddeeedec4efe1eba8d2/mellow-lrt/src/utils/DefaultAccessControl.sol#L23-L26)
```solidity
        _setRoleAdmin(ADMIN_ROLE, ADMIN_ROLE);
        _setRoleAdmin(ADMIN_DELEGATE_ROLE, ADMIN_ROLE);
        _setRoleAdmin(OPERATOR, ADMIN_DELEGATE_ROLE);
    }
```

## Impact
admin role will not be able to assign/resign operators

## Code Snippet

## Tool used
Manual Review

## Recommendation
Correct the documentation or add additional functionality to grant/revoke operator role