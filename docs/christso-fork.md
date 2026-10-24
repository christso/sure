# Christso Sure fork

Repository: https://github.com/christso/sure

## Branches

- `main` mirrors `we-promise/sure:main`. Keep it free of fork-specific commits.
- `christso/main` is the default branch and holds reviewed local patches. It
  starts at the deployed `v0.7.5-alpha.9` source commit,
  `ab457b47370ae69964a835acea96083edd4b13d9`, to keep this repair separate from
  an upstream version upgrade.
- `upstream` is the local Git remote for `https://github.com/we-promise/sure.git`.

The **Sync upstream main** workflow is scheduled every six hours. It checks that
`main` has no fork-only commits, uses GitHub's fork-sync API with the built-in
token, and verifies that the result matches upstream. It never changes
`christso/main`; upstream upgrades on that branch remain deliberate and tested.
Run the workflow manually for an immediate refresh. GitHub may disable scheduled
workflows on inactive public repositories, so verify its status before upgrades.

**Christso CI** runs the existing upstream CI suite on pushes to `christso/main`.
Upstream release and publishing workflows remain disabled in this fork; a source
push does not publish a release or deploy an application.

## Loan-transfer repair

The first patch backports upstream [PR #3714](https://github.com/we-promise/sure/pull/3714),
revision `492a9829c1e4ed86b427c2d72f695a86ea75ca6b`:

- Provider imports preserve the kind of transactions already paired by a transfer.
- A data migration resets stale `loan_payment` kinds on transfer inflows to
  `funds_movement`. Bank-side repayments keep their existing budget treatment.
- Regression coverage includes the upstream SimpleFIN/Up scenarios and the
  actual Redbark transaction processor, with pending and confirmed transfer links.
- The repair also touches the repaired entries' timestamps to invalidate cached
  financial reports. A warm-cache regression covers this addition to the upstream patch.

The migration covers existing transfer inflows across the database, including
pending links. It changes transaction kinds and repaired entry timestamps;
categories, amounts, balances and transfer links are preserved. It is idempotent
and its down migration is irreversible.
Review its affected set on a protected database copy and verify a recoverable
database backup before production rollout. Rolling back the image does not undo
this data repair.

Source preparation and CI do not authorize or perform a production migration.
Build and deploy a pinned image only after validating the target database and
the web/worker rollout together. Keep production data and credentials out of
this public source repository and GitHub Actions.

## Updating

```sh
git fetch upstream main
git switch main
git merge --ff-only upstream/main
git push origin main
```

For a deliberate application upgrade, merge the reviewed upstream revision into
`christso/main`, resolve patch conflicts, run the full suite, and verify the
migration path before publishing an image. Once upstream contains an equivalent
loan-transfer fix, reconcile the local migration's identity and remove redundant
patches rather than applying a second repair blindly.
