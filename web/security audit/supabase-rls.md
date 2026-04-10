# Supabase RLS & SQL Audit

## What to Scan

1. **Migration files**: `supabase/migrations/*.sql`
2. **Seed files**: `supabase/seed.sql` if present
3. **RLS policies**: Extract from migrations — look for `CREATE POLICY`, `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`
4. **Database functions**: `CREATE FUNCTION` definitions, especially those marked `SECURITY DEFINER`
5. **Supabase client usage**: How the app calls `supabase.from(...)` — does it rely on RLS or bypass it?

## Checklist

### RLS Enablement

- [ ] **Every table with user data has RLS enabled.**
  Look for `ALTER TABLE <name> ENABLE ROW LEVEL SECURITY` in migrations.
  If a table stores user-specific data and lacks this, it's **Critical**.

- [ ] **No table has RLS enabled but zero policies defined.**
  This silently blocks all access (which is safe but usually a bug). Flag as **Info**
  unless the table is clearly meant to be accessed, in which case **Medium**.

- [ ] **Junction/join tables have RLS too.**
  Tables like `user_screenshots`, `team_members` are often forgotten. **High** if missing.

### Policy Quality

- [ ] **SELECT policies filter by `auth.uid()`.**
  A SELECT policy that doesn't reference `auth.uid()` or a function that resolves to it
  means any authenticated user can read all rows. **Critical** for sensitive data,
  **High** for less sensitive data.

- [ ] **INSERT policies validate ownership.**
  Users should only be able to insert rows where they are the owner.
  Check that `WITH CHECK` clauses enforce `auth.uid() = user_id`. **High** if missing.

- [ ] **UPDATE policies use both USING and WITH CHECK.**
  `USING` controls which rows can be targeted; `WITH CHECK` controls what the row
  looks like after the update. Missing `WITH CHECK` means a user could update a row
  to change its `user_id` to someone else's. **Critical**.

- [ ] **DELETE policies are scoped to owner.**
  A DELETE policy without `auth.uid()` filtering lets any user delete any row. **Critical**.

- [ ] **No policy uses `true` or `1=1` as the expression for non-public tables.**
  This effectively disables RLS. **Critical** unless the table is intentionally public
  (and if so, confirm it contains no sensitive data).

- [ ] **Policies don't accidentally grant access to `anon` role on private data.**
  Check `TO` clause. A policy `TO authenticated, anon` on user data is **Critical**.

### Service Role Usage

- [ ] **Service role key is only used server-side.**
  Grep for `supabase.createClient` with `service_role` key. If it appears in any file
  that could be bundled for the client (files without `"use server"`, files in
  `app/` components without server-only imports), it's **Critical**.

- [ ] **Server-side service role calls have their own auth checks.**
  When bypassing RLS via service role, the application code must enforce authorization.
  Check that these calls validate the requesting user's permissions. **High** if missing.

- [ ] **Cron jobs and background tasks use service role appropriately.**
  These legitimately need service role, but verify they don't accept user input that
  could influence which rows are affected. **Medium** if user input flows in unchecked.

### SQL Injection

- [ ] **No raw SQL string concatenation with user input.**
  Look for `.rpc()` calls where arguments are constructed from user input, or any use
  of `supabase.rpc('function_name', { param: userInput })` where the function uses
  the param in dynamic SQL (`EXECUTE format(...)`). **Critical** if found.

- [ ] **Database functions use parameterized queries.**
  Check `CREATE FUNCTION` bodies for `EXECUTE` with string concatenation vs.
  `EXECUTE ... USING` with parameters. **Critical** if concatenating.

- [ ] **`.textSearch()` and `.filter()` calls sanitize input.**
  Supabase client methods generally parameterize, but custom filter strings can be
  injected. **High** if raw user input goes into filter expressions.

### SECURITY DEFINER Functions

- [ ] **Functions marked `SECURITY DEFINER` are minimal and necessary.**
  These run with the privileges of the function creator (usually superuser).
  Each one should be reviewed for what it accesses. **High** if it does more than
  necessary or accepts broad input.

- [ ] **`SECURITY DEFINER` functions have `search_path` set.**
  Without `SET search_path = public`, a malicious user could exploit search path
  injection. Check for `SET search_path` in the function definition. **Medium**.

- [ ] **`SECURITY DEFINER` functions validate `auth.uid()` internally.**
  Just because RLS is bypassed doesn't mean the function should act on behalf of
  any user. **High** if the function doesn't check who's calling it.

### Storage (Supabase Storage Buckets)

- [ ] **Private buckets don't have public policies.**
  Check storage policies in migrations or Supabase dashboard configuration.
  A "private" bucket with a permissive SELECT policy is **High**.

- [ ] **Upload policies restrict file types and sizes.**
  Missing restrictions allow users to upload arbitrary files. **Medium**.

- [ ] **Storage paths include user ID for ownership.**
  Pattern: `storage.objects` policies should enforce that the path starts with
  `auth.uid()`. Without this, users can overwrite each other's files. **High**.

## Common Patterns to Flag

```sql
-- BAD: Overly permissive, any authenticated user can read all rows
CREATE POLICY "select_all" ON screenshots FOR SELECT
  TO authenticated USING (true);

-- GOOD: Scoped to owner
CREATE POLICY "select_own" ON screenshots FOR SELECT
  TO authenticated USING (auth.uid() = user_id);

-- BAD: Missing WITH CHECK on update
CREATE POLICY "update_own" ON screenshots FOR UPDATE
  TO authenticated USING (auth.uid() = user_id);

-- GOOD: Both USING and WITH CHECK
CREATE POLICY "update_own" ON screenshots FOR UPDATE
  TO authenticated
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- BAD: SECURITY DEFINER without search_path
CREATE FUNCTION admin_delete(target_id uuid)
RETURNS void SECURITY DEFINER AS $$
  DELETE FROM screenshots WHERE id = target_id;
$$ LANGUAGE sql;

-- GOOD: Scoped and restricted
CREATE FUNCTION admin_delete(target_id uuid)
RETURNS void
SECURITY DEFINER
SET search_path = public
AS $$
  -- Verify caller is admin
  IF NOT exists(SELECT 1 FROM admins WHERE user_id = auth.uid()) THEN
    RAISE EXCEPTION 'unauthorized';
  END IF;
  DELETE FROM screenshots WHERE id = target_id;
$$ LANGUAGE plpgsql;
```
