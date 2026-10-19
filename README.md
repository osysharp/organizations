# Osysharp.Organizations

The groups people belong to — the **companies** an app's customers work for, and the **teams** its own people are in:
who belongs to which, the roles they hold **there**, and the mail domains that put a newcomer in the right company.

- **`Organization`** — a base the app derives from (`Company : Organization`, `Team : Organization`) or uses as-is.
  `Parent` makes a group of companies or a department of teams.
- **`Membership`** — a person's place in one, with its `Source` (added here, a mail domain, an invitation, single sign-on,
  SCIM) and `ExternalId`, so an identity provider later keeps in step only the memberships it made.
- **`MembershipRole`** — a role held **within** one organisation: a member of the app's own `[Role]` enum, held there and
  nowhere else.
- **`OrganizationDomain`** — `acme.com` belongs to Acme: a person whose address on it is **proven** joins Acme when they
  sign in.

## Use it

```osy
// app.osy
use Osysharp.Identity@0;
use Osysharp.Organizations@0;
```

```osy
using Osysharp.Identity;
using Osysharp.Organizations;

[Role] enum AppRole { Authenticator, Staff, Buyer, Viewer }
policy IsStaff => RoleGrant.Any(g => g.Member == user && g.Role == AppRole.Staff);

// The kit's one question: who looks after the organisations.
policy ManagesOrganizations => IsStaff;

entity Company : Organization {
  [MaxLength(40)] string AccountNumber = "";
  security {
    allow read, create, update when IsStaff;
    allow read when IsAuthenticated where IsMemberOf(this);      // its own people see it
  }
}

entity PurchaseOrder {
  [Required("Say which company the order is for.")] Company Company;
  security {
    allow read where IsMemberOf(Company);
    allow create where HoldsRoleIn(Company, AppRole.Buyer);     // a buyer AT THIS company
  }
}
```

Put a newcomer in their company by their proven address, at every sign-in:

```osy
public class Signins : AccountEvents {
  public override void SignedIn(User account, DateTime at) { JoinByDomain(account); }
}
```

## What holds whatever the app grants

- **A role held at Acme is held at Acme.** It is a row of its own table, never a `RoleGrant`, so no policy over the app's
  roles can read it as an app-wide role.
- **Nobody grants themself a role in an organisation** — not an organisation's own admin, not the app's staff — and the
  sign-in flow's `Authenticator` is never one. A role row is made or removed, never edited into another.
- **A mail domain joins only a PROVEN address.** An address typed at sign-up proves nothing; a membership made by a domain
  is refused for an account whose address was not proven, whoever tries to make it.
- **Which domain belongs to which organisation is the app's to say** — its organisation managers', never an
  organisation's own people's — one domain belongs to one organisation, and a public mail service's domain (`gmail.com`,
  `outlook.com`, …) is refused.
- A membership is one person in one organisation; it is never re-pointed at another of either. A suspended membership
  (`IsActive = false`, what a directory sends for a person who left) holds nothing.

## The service a customer gets — the Accounts add-on

What a vendor keeps about the organisations it **serves** — the tier each is on, what it is entitled to, the account team
looking after it and its VIP people — is a separate kit, `Osysharp.Organizations.ServicePlans`, that uses this one. An app
that only needs "these people belong to this organisation" takes this kit alone.

## What an app calls

| | |
|---|---|
| `IsMemberOf(org)` · `HoldsRoleIn(org, role)` | policies — the caller's membership and roles, for rules |
| `Join(org, person)` · `Leave(org, person)` | add (or wake a suspended membership) · the bare removal, unrecorded |
| `RemoveMember(org, person)` | take a person out — or leave, as the caller: recorded (`MembershipRemoval`), their roles and membership gone, the app told (`OrganizationsSetup.Events`), the person mailed; the last holder of `OrganizationsSetup.KeepOneOf` refused by name; their account untouched |
| `GrantIn(org, person, role)` · `RevokeIn(org, person, role)` · `HoldsIn(person, org, role)` | roles within one organisation |
| `AddDomain(org, "acme.com", joinRole?)` · `OrganizationByDomain(email)` · `JoinByDomain(person)` | mail domains |
| `OrganizationsOf(person)` · `DomainOf(email)` · `IsPublicMailDomain(domain)` | |

Invitations into an organisation are `Osysharp.Invitations`; the people list that shows and changes memberships is
`Osysharp.UserManagement`.
