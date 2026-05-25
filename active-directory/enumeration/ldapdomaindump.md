# LDAPDomainDump

Dumps all LDAP data (users, computers, groups, trusts) into readable files.

## Command

```sh
python3 -m ldapdomaindump ldaps://<DC-IP> -u 'DOMAIN\user' -p 'Password'
```

Or with plain LDAP:
```sh
python3 -m ldapdomaindump ldap://<DC-IP> -u 'DOMAIN\user' -p 'Password'
```

## Output Files

| File | Contents |
|---|---|
| `domain_users.json` | All domain users and attributes |
| `domain_computers.json` | All domain-joined computers |
| `domain_groups.json` | All groups and memberships |
| `domain_trusts.json` | Domain trust relationships |
| `domain_policy.json` | Password policy, lockout settings |

## Viewing Output

```sh
# Open HTML files in browser
firefox domain_users.html

# Or inspect JSON
cat domain_users.json | python3 -m json.tool
```

## What to Look For

- Users with passwords in description fields
- Service accounts with elevated privileges
- Stale/disabled accounts
- Groups with unexpected members (e.g., who's in Domain Admins)
- Password policy (lockout threshold for spray timing)
