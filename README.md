# Smart Notes

A notes app with a Draft, Active, Archived workflow, validation, empty and error states, and a layout that works on desktop and phone.

It runs in two modes:
- **Sample mode** (no setup): six sample notes, saved in your browser. This is what you get until you add Supabase settings.
- **Supabase mode**: people create an account, sign in, and their notes are saved in your Supabase database. Each person only sees their own notes.

All data access sits behind six functions (`list`, `get`, `create`, `update`, `remove`, `reset`), so the screen code is the same in both modes.

## 1. Run it with sample data
You need Node.js 18 or newer.

```
npm start
```

Open http://localhost:3000. Add `?empty=1` to see the empty state and `?fail=1` to see the error state. Add `?local` to force sample mode even after Supabase is set up.

## 2. Connect Supabase (database and sign-in)

1. **Create the table.** In Supabase open SQL Editor, paste everything from `supabase/schema.sql`, and run it. It creates the `notes` table and turns on Row Level Security, so each signed-in user can only read and change their own rows.
2. **Allow email sign-in.** Authentication > Providers: Email is on. For quick testing you can turn "Confirm email" off; otherwise new users must click the link in their email first.
3. **Put your settings in `.env`.** Copy `.env.example` to `.env` and fill in:
   - `SUPABASE_URL`: Project Settings > API > Project URL
   - `SUPABASE_ANON_KEY`: the **anon / public** key from the same page

   The anon key is meant to be public, because Row Level Security is what protects the data. Do **not** use the `service_role` key or your personal access token (`sbp_...`) here. `npm run config` refuses both and tells you so without printing them.
4. **Create the app's settings file:**
   ```
   npm run config
   ```
   It reads `.env` and writes `src/config.js`. It prints only "Wrote src/config.js", never the values. If your `.env` lives somewhere else, run it with `ENV_FILE=/path/to/.env npm run config`.
5. **Start the app:** `npm start`. You now see a sign-in screen. Create an account, add a note, and refresh the page: you stay signed in and the note is still there.

## 3. MCP connection (optional, for Claude Code)
The MCP connection lets Claude Code look at and change your Supabase project (for example, run `supabase/schema.sql` for you). The app itself does not need it.

Recommended way (browser login, no token to paste anywhere):

```
claude mcp add --scope project --transport http supabase "https://mcp.supabase.com/mcp?project_ref=YOUR_PROJECT_REF"
```

Then start Claude Code, run `/mcp` (Supabase's docs say to do this in a regular terminal rather than the IDE extension), choose `supabase`, and pick Authenticate. Sign in in the browser and choose the organization that holds your project.

Token way (for unattended setups): copy `.mcp.json.example` to `.mcp.json`. It reads `SUPABASE_PROJECT_REF` and `SUPABASE_ACCESS_TOKEN` from your environment, so the token is never written into the file. Create a token limited to this one project. Do not commit `.mcp.json` with a literal token in it.

Safety habits from Supabase's guidance: scope the server to one project with `project_ref`, add `&read_only=true` once the table exists, keep tool approval switched on, and use a development project rather than one with real customer data.

## 4. Check that a saved record survives a refresh

**In the browser (30 seconds):**
1. Sign in, click New note, type a title, click Save note.
2. Press F5. You should still be signed in and see the note.
3. Open the same address in a private window and sign in with the same account. The note is there too.
4. Sign out. The notes disappear from the screen and the sign-in form comes back.

**With the check script** (it saves a note, reads it back with a brand-new client, edits it, confirms signed-out visitors cannot read or write, then deletes the note):

1. Sign up once in the app with a throwaway email, and confirm it if confirmation is on.
2. Add `VERIFY_EMAIL` and `VERIFY_PASSWORD` for that account to `.env`.
3. Run:
   ```
   npm install
   npm run verify
   ```
4. You should see six PASS lines and "All checks passed". The script prints only PASS or FAIL lines. It hides the URL, key, email and password even inside error messages.

## 5. Publish the frontend on Vercel
1. Put the project on GitHub. `.gitignore` already keeps `.env`, `src/config.js` and `dist/` out.
2. On vercel.com choose Add New > Project and import the repository. `vercel.json` already sets the build command (`npm run build`) and output folder (`dist`).
3. Before deploying, open Settings > Environment Variables and add `SUPABASE_URL` and `SUPABASE_ANON_KEY` (the same two public values).
4. Deploy, then copy your live link.
5. In Supabase open Authentication > URL Configuration, set Site URL to your Vercel link, and add it to Redirect URLs. This makes the confirmation email link come back to your site.
6. Open the live link on your phone and your computer, sign in with the same account, and check the same notes show up.

## 6. Keeping credentials private
- Real values live only in `.env` (ignored by git) and in Vercel's environment variables.
- Only the project URL and the public anon key reach the browser. The personal access token is for MCP only and the app never reads it.
- Do not paste keys or tokens into chat, code, screenshots or commits. If one leaks, rotate it in the Supabase dashboard.
- The scripts print what happened, never the values.

## Tests
```
npm test
```
45 automated tests cover validation, workflow rules, tag suggestions, filtering, the sample-mode data layer, the Supabase data layer (run against a fake database that imitates per-user privacy and the table rules), sign-in checks, and the settings scripts (including refusing secret keys). They need no internet or credentials.

## Sample mode checks
| Try this | You should see |
| --- | --- |
| Open the app with no `src/config.js` | Six sample notes. The Active tab shows three, "Plan the week" pinned on top. |
| New note, then Save note with nothing typed | "Add a title." and nothing is saved. |
| Tags like `bad tag!` or more than 5 tags | A tag message and nothing is saved. |
| Open a Draft note, click Mark active | Badge changes and the list follows to the Active tab. |
| Archive a note | It moves to Archived and Pin disappears. |
| Delete: click twice | First click asks "Tap again to delete". |
| `/?empty=1` | The "No notes yet" empty state. |
| `/?fail=1` | "Notes did not load" with a Try again button. |
| Window narrower than 760px | One column with a back button. |

## Files
- `index.html`, `styles.css`: page and styling
- `src/app.js`: screens; chooses sample mode or Supabase mode
- `src/notesRepository.js`: sample-mode data access (browser storage)
- `src/supabaseRepository.js`: Supabase data access, same six functions
- `src/auth.js`, `src/supabaseClient.js`: sign-in and client setup
- `src/validation.js`, `src/workflow.js`, `src/smart.js`, `src/query.js`: plain logic
- `supabase/schema.sql`: table, security rules
- `scripts/`: `write-config.js` (makes `src/config.js`), `build.js` (Vercel build), `verify.js` (refresh check), `env.js`
- `.env.example`, `.mcp.json.example`, `vercel.json`, `.gitignore`
- `tests/`: automated tests
