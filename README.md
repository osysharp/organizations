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

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.Organizations;
using OrganizationsKitTests;

[Test(Portal)]
void a_proven_address_on_the_domain_joins_the_company_at_sign_in_with_the_domains_role() {
  string ticket = "";
  runas (AuthBootstrap) { ticket = SignInWithCode("ada@acme.com", "2222"); }
  Assert.NotEqual("", ticket);
  runas (Ada) {
    var mine = Membership.Single(m => m.Member == Ada);
    Assert.Equal("Acme", mine.Organization.Name);
    Assert.Equal(MembershipSource.Domain, mine.Source);
    Assert.True(MembershipRole.Any(r => r.Membership == mine && r.Role == PortalRole.Viewer));
  }
}
```

From [`tests/domains.test.osy`](tests/domains.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Organizations;
using Osysharp.Mail;
using Osysharp.Workflow;
using OrganizationsKitTests;

[Test(Acme)]
void a_manager_takes_a_person_out_and_it_is_recorded_mailed_and_their_access_ends_at_once() {
  // Ada reads her company before — so an answer about her is held, and must not outlive the removal.
  runas (Ada) { Assert.Equal(1, Company.Count()); }
  MembershipRemoval? removal = null;
  runas (Sam) { removal = RemoveMember(Company.Single(c => c.Name == "Acme"), Ada); }
  Workflow.Settle(removal!);
  runas (Sam) {
    Assert.False(Membership.Any(m => m.Member == Ada));
    // The app was told, in the same write: her desk at Acme is gone.
    Assert.False(CompanyDesk.Any(d => d.Holder == Ada));
    var r = MembershipRemoval.Single(x => x.Member == Ada);
    Assert.Equal(MembershipRemovalState.Removed, r.State);
    Assert.Equal("sam@supplier.test", r.RemovedBy.Email);
    Assert.Equal("Acme", r.Organization.Name);
  }
  // Her next request reads nothing of Acme.
  runas (Ada) {
    Assert.Equal(0, Company.Count());
    Assert.Equal(0, Membership.Count());
  }
  // She is told who removed her, and that her account remains.
  runas (AuthBootstrap) {
    var mail = OutgoingMail.Single(m => m.To == "ada@acme.com" && m.Subject == "You were removed from Acme");
    Assert.True(mail.Text!.Contains("Sam removed you from Acme"));
    Assert.True(mail.Text!.Contains("Your account remains"));
  }
  // Removal is not erasure: the account is still there, and still signs in.
  runas (AuthBootstrap) { Assert.NotEqual("", SignInWithCode("ada@acme.com", "2222")); }
}
```

From [`tests/removal.test.osy`](tests/removal.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Organizations;
using OrganizationsKitTests;

[Test(Companies)]
void a_buyer_at_acme_buys_for_acme_and_for_nobody_else() {
  runas (Bea) {
    var acme = Company.Single(c => c.Name == "Acme");
    new PurchaseOrder { Company = acme, Reference = "A-100" };
    UnitOfWork.Commit();
    // Globex is not hers to read, let alone to buy for.
    Assert.False(Company.Any(c => c.Name == "Globex"));
  }
  Company? globex = null;
  runas (Sam) { globex = Company.Single(c => c.Name == "Globex"); }
  runas (Bea) { Assert.Denied(() => new PurchaseOrder { Company = globex!, Reference = "G-200" }); }
}
```

From [`tests/roles.test.osy`](tests/roles.test.osy).

## Learn more

- [The Osysharp.Organizations reference](https://osysharp.com/reference/security/organizations-kit/)
- [Osysharp.Organizations in the package catalogue](https://osysharp.com/packages/#osysharp-organizations)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
