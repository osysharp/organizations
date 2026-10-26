# Osysharp.Organizations.ServicePlans

The service each organisation you **serve** gets: the tier it is on, what it is entitled to, the account team that looks
after it, and its VIP people. Your support, sales or success kits route and promise by them. Only your own people set
them, never the organisation's.

It is an add-on to `Osysharp.Organizations`, which holds who belongs to which organisation, the roles they hold there
and the mail domains that place a newcomer. Take this kit as well when the organisations are your **customers**:

```osy
// app.osy
use Osysharp.Identity@0;
use Osysharp.Organizations@0;
use Osysharp.Organizations.ServicePlans@0;
```

```osy
using Osysharp.Identity;
using Osysharp.Organizations;
using Osysharp.Organizations.ServicePlans;

policy ManagesOrganizations => IsStaff;   // the base kit's one question — this kit asks the same

// A customer is a CustomerAccount (itself an Organization) — it has a tier.
entity Company : CustomerAccount {
  [MaxLength(40)] string AccountNumber = "";
  security {
    allow read, create, update when IsStaff;
    allow read when IsAuthenticated where IsMemberOf(this);
  }
}

// Your own teams stay plain organisations: nobody serves them.
entity Team : Organization { security { allow read, create, update when IsStaff; } }

// What a tier (or one company's contract) includes, in your own shape.
entity ResponseTarget : Entitlement {
  int FirstReplyHours = 24;
  security { allow read, create, update, delete when IsStaff; }
}
```

## What it holds

- **`CustomerAccount`** — an organisation you serve, with its `Tier`. Derive your customers from it.
- **`OrganizationTier`** — Standard, Premium, Enterprise, as you name them: rows your organisation managers define,
  ordered by `Rank`, one of them `IsDefault`. Rows rather than an enum, because they are configuration you change
  without a release.
- **`Entitlement`** — what an organisation may have: a channel, support hours, a response target. Attached to a tier
  (everybody on it) or to one organisation (its own contract), for a period. Derive your own kind.
- **`AccountTeamMember`** — your own people looking after an organisation: one `Owner`, a pod (`Member`), and a `Backup`
  who covers while the owner is away. An organisation's people can see who looks after them.
- **`VipMembership`** — a person who is a VIP **at** an organisation: an executive, a key contact. They stop being one
  when their membership is suspended or ends.

## What holds whatever your app grants

- **The tier is yours to set.** An organisation's own people never change the tier they are on, however much else your
  app lets them edit about their company.
- **So is who is a VIP, and who is on the account team.** Nobody makes themself, or a colleague, a VIP.

## The tiers page

`/settings/tiers` joins your settings area by itself, for your organisation managers: each tier, its rank, what it is,
and what everybody on it is entitled to, kept on Save. Write a component called `TierSettings` to replace it, or use
`TierEditor(description: "…")` inside your own page to keep the editor and say it in your app's words.

## What an app calls

| | |
|---|---|
| `DefaultTier()` · `TierOfAccount(org)` | the default tier · the tier an organisation is on, or the default |
| `EntitlementsOf(org)` | what applies to an organisation now — its own contract first, then its tier's |
| `AccountOwnerOf(org)` · `AccountTeamOf(org)` · `SetAccountOwner(org, person)` · `AddToAccountTeam(org, person, role)` | the account team |
| `MarkVip(org, person)` · `UnmarkVip(org, person)` · `IsVipPerson(person)` | VIPs |

`Osysharp.WorkQueues` uses all of it: a customer's work goes to its account owner first, then the backup and the pod, and
its promise is chosen by its tier and whether the person asking is a VIP.

## Its tests

The kit's test app is a supplier's portal: two customer companies on two tiers, a contract of their own, an account
team, and a VIP buyer who may edit her company's details but never its tier, its team or who is a VIP. The tiers page
is driven as a person would use it, wide and on a phone.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.Organizations;
using Osysharp.Organizations.ServicePlans;
using AccountsKitTests;

[Test(Service)]
void an_organisation_is_entitled_to_its_own_contract_first_then_its_tiers() {
  runas (Sam) {
    var acme = Company.Single(c => c.Name == "Acme");
    var all = EntitlementsOf(acme);
    Assert.Equal(2, all.Count);
    Assert.Equal("Named engineer", all[0].Name);
    Assert.Equal("Phone support", all[1].Name);
    Assert.Equal("Standard", DefaultTier()!.Name);
  }
}
```

From [`tests/service.test.osy`](tests/service.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Organizations;
using Osysharp.Organizations.ServicePlans;
using AccountsKitTests;

[Test(Service)]
void a_companys_people_read_what_they_are_entitled_to_and_nothing_of_another() {
  runas (Ada) {
    Assert.Equal(2, SupportEntitlement.Count());
    Assert.False(SupportEntitlement.Any(e => e.Name == "Email support"));
  }
  runas (Gil) {
    Assert.Equal(1, SupportEntitlement.Count());
    Assert.Equal("Email support", SupportEntitlement.Single().Name);
  }
}
```

From [`tests/service.test.osy`](tests/service.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Organizations;
using Osysharp.Organizations.ServicePlans;
using AccountsKitTests;

[Test(Service)]
void the_account_team_has_one_owner_and_a_backup_and_the_companys_people_see_who_looks_after_them() {
  runas (Sam) {
    var acme = Company.Single(c => c.Name == "Acme");
    var sam = Person.Single(p => p.Email == "sam@supplier.test");
    var kim = Person.Single(p => p.Email == "kim@supplier.test");
    SetAccountOwner(acme, sam);
    AddToAccountTeam(acme, kim, AccountTeamRole.Backup);
    UnitOfWork.Commit();
    SetAccountOwner(acme, kim);
    UnitOfWork.Commit();
    Assert.Equal("kim@supplier.test", AccountOwnerOf(acme)!.Email);
    Assert.Equal(1, AccountTeamMember.Count(m => m.Organization == acme && m.Role == AccountTeamRole.Owner));
    Assert.Equal(2, AccountTeamOf(acme).Count);
  }
  runas (Ada) { Assert.Equal(2, AccountTeamMember.Count()); }
  runas (Gil) { Assert.Equal(0, AccountTeamMember.Count()); }
}
```

From [`tests/service.test.osy`](tests/service.test.osy).

## Learn more

- [The Osysharp.Organizations.ServicePlans reference](https://osysharp.com/reference/security/organizations-accounts-kit/)
- [Osysharp.Organizations.ServicePlans in the package catalogue](https://osysharp.com/packages/#osysharp-organizations-serviceplans)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
