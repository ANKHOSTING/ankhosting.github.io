# Free setup: hosting, AdMob, RevenueCat (Urban Sprout + TextCraft)

Nothing here has been pushed or created. Everything below is for Angel to run.

## 1. Free privacy-policy + delete-account links (GitHub Pages)
Cost: $0 (public repo). Needs the repo to be named exactly `ankhosting.github.io`.
1. github.com/new → owner `ANKHOSTING`, name `ankhosting.github.io`, **Public**.
2. Upload the contents of this folder (index.html, assets/, urban-sprout/, textcraft/, .nojekyll) to the repo root on `main`.
   CLI: `cd /workspace/ankhosting-pages && git init -b main && git add -A && git commit -m "site" && git remote add origin https://github.com/ANKHOSTING/ankhosting.github.io.git && git push -u origin main`
3. Repo → Settings → Pages → Source "Deploy from a branch" → `main` / `/ (root)` → Save. Live in ~1 min.
4. Final URLs:
   - https://ankhosting.github.io/
   - https://ankhosting.github.io/urban-sprout/privacy.html
   - https://ankhosting.github.io/urban-sprout/delete-account.html
   - https://ankhosting.github.io/textcraft/privacy.html
   - https://ankhosting.github.io/textcraft/delete-account.html
5. Open each URL in a private window to confirm it loads without login (Play requires public pages).
6. Use them:
   - Play Console → each app → App content → Privacy policy: the privacy URL.
   - Play Console → Data safety → "Delete account URL" (Urban Sprout): the delete-account URL. TextCraft: answer "no account creation"; the delete page is optional but harmless.
   - Urban Sprout code: `app/(tabs)/account.tsx` → `const PRIVACY_URL = 'https://ankhosting.github.io/urban-sprout/privacy.html'`.
   - TextCraft code: env `EXPO_PUBLIC_PRIVACY_POLICY_URL=https://ankhosting.github.io/textcraft/privacy.html` (`app/(tabs)/settings.tsx` reads it).
   - Play listing "Website": https://ankhosting.github.io/

### REQUIRED before publishing TextCraft's pages: 30-day purge
The pages promise TextCraft's `ai_usage` rows (random device id + day + count) are deleted after 30 days, but the code has no purge job. Run once in Supabase → SQL editor (pg_cron is free on Supabase; enable it under Database → Extensions):
```sql
create extension if not exists pg_cron;
select cron.schedule('purge-ai-usage','15 3 * * *', $$delete from public.ai_usage where day < current_date - 30$$);
```
(Use the TextCraft project. If you'd rather not, change "deleted after 30 days" in textcraft/privacy.html and delete-account.html.)

## 2. Recommended monetization (what the code already supports)
- Ads: **one adaptive Banner unit per app**, non-personalized requests (already in code), shown to free users only. No interstitials/rewarded: simplest for Play review and consistent with the Data safety answers.
- Subscriptions: **Pro, entitlement `pro`**, two products per app: `pro_monthly` $2.99/month and `pro_yearly` $14.99/year (~58% off monthly). Optional: add a 7-day free trial offer on the yearly base plan in Play.
- Product IDs can be the same in both apps (Play scopes them per app).

## 3. AdMob (free), per app
1. admob.google.com → sign in → Apps → Add app → Android → "Not listed on Google Play yet" is fine. Create two apps: Urban Sprout, TextCraft. Each gives an **App ID** `ca-app-pub-XXXX~YYYY`.
2. In each app → Ad units → Add ad unit → **Banner** → name "main-banner" → gives a **Unit ID** `ca-app-pub-XXXX/ZZZZ`.
3. Put the IDs in:
   | Repo | App ID (replace test id) | Banner unit ID |
   |---|---|---|
   | Urban-Sprout | `app.json` → plugins → `react-native-google-mobile-ads` → `androidAppId` (currently test `ca-app-pub-3940256099942544~3347511713`) | `.env` and EAS env: `EXPO_PUBLIC_ADMOB_BANNER_ANDROID` |
   | TextCounter | same path in its `app.json` | same env key |
   Optional kill-switch: `EXPO_PUBLIC_ADS_ENABLED=false`.
4. Release builds only use the real unit when the env var is set (dev builds always use Google test ads, so you can't get flagged by clicking your own ads).
5. After the apps are live on Play: AdMob → App settings → link to the Play listing. Also add `app-ads.txt` to the site root (AdMob → Apps → View app-ads.txt shows the exact line; set the Play listing website to https://ankhosting.github.io/).
6. Payments: AdMob → Payments → add tax info + address (free).

## 4. RevenueCat (free under $2.5k/mo tracked revenue), per app
1. app.revenuecat.com → create a project per app (or one project, two Play apps).
2. Project → Apps → add **Play Store** app with package name `com.ankhosting.urbansprout` / `com.textcounter.app`.
3. Play Console → Setup → API access → create a service account, grant it financial permissions, download the JSON → upload in RevenueCat (Play Store app → Service credentials). Allow up to ~36 h for it to validate.
4. Play Console → each app → Monetize → Subscriptions → create `pro_monthly` (base plan monthly, $2.99) and `pro_yearly` (base plan yearly, $14.99); activate both. (An app release must be uploaded to a Play track first.)
5. RevenueCat → Products → import/add both products. Entitlements → create **`pro`** → attach both products.
6. Offerings → open the **default/current** offering → add packages: Monthly (`$rc_monthly` → pro_monthly), Annual (`$rc_annual` → pro_yearly). The apps read `offerings.current.availablePackages`, so the offering must be marked Current.
7. API keys → copy the Public **Google/Android SDK key** (`goog_...`) → put in each repo's `.env` and EAS env: `EXPO_PUBLIC_REVENUECAT_ANDROID_KEY`.
8. Server-side Pro checks:
   - Urban Sprout (webhook): RevenueCat → Integrations → Webhooks → URL `https://<SUPABASE-REF>.functions.supabase.co/revenuecat-webhook`, Authorization header `Bearer <random-secret>`; then `supabase secrets set REVENUECAT_WEBHOOK_AUTH=<random-secret>` and `supabase functions deploy revenuecat-webhook`.
   - TextCraft (REST check): RevenueCat → API keys → copy a **Secret** key (`sk_...`) → `supabase secrets set REVENUECAT_SECRET_KEY=sk_...` (optional `REVENUECAT_ENTITLEMENT=pro`) → `supabase functions deploy writing-check`.
9. Test: add your Gmail as a Play license tester, install an internal-track build, buy (test card, charged nothing), check `pro` becomes active.

## 5. Env keys per repo (all in `.env` locally AND `eas env:create` / EAS dashboard for cloud builds; eas.json has no env block)
- Both: `EXPO_PUBLIC_SUPABASE_URL`, `EXPO_PUBLIC_SUPABASE_ANON_KEY`, `EXPO_PUBLIC_REVENUECAT_ANDROID_KEY`, `EXPO_PUBLIC_ADMOB_BANNER_ANDROID`, `EXPO_PUBLIC_ADS_ENABLED` (optional).
- TextCounter also: `EXPO_PUBLIC_PRIVACY_POLICY_URL`.
- Supabase secrets: both `AI_API_KEY` (+ optional `AI_BASE_URL`, `AI_MODEL`, `FREE_DAILY_LIMIT`); Urban Sprout `REVENUECAT_WEBHOOK_AUTH`; TextCounter `REVENUECAT_SECRET_KEY`.
