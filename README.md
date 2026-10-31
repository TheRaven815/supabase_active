# supabase_active

Supabase Free plan projesinin 7 gunluk hareketsizlik sonrasi duraklatilmasini (pause) engelleyen
minimal GitHub Actions keep-alive.

## Nasil calisir

Her gun 03:17 UTC'de workflow su sirayla dener:

1. `POST /rest/v1/rpc/keepalive` — gercek Postgres sorgusu (`select now()`), anon key ile 200 doner
2. `GET /auth/v1/health` — yedek
3. `GET /storage/v1/version` — son yedek

En az biri `200`/`204` donmezse job fail olur, sessiz basarisizlik olmaz.
`/rest/v1/` kok adresi **kullanilmaz**: o endpoint yalnizca `service_role` key kabul eder ve anon key
ile `401 Only the service_role API key can be used for this endpoint` doner.

## Gerekli secret'lar

`Settings -> Secrets and variables -> Actions`:

| Secret | Deger |
| --- | --- |
| `SUPABASE_URL` | `https://<project-ref>.supabase.co` |
| `SUPABASE_KEY` | anon / publishable key (`sb_publishable_...` veya eski JWT anon key) |

`service_role` key **kullanilmaz**.

## Veritabani tarafi

`public.keepalive()` fonksiyonu bir kez olusturulmus olmali:

```sql
create or replace function public.keepalive()
returns timestamptz
language sql
security definer
set search_path = ''
as $$ select now() $$;

revoke all on function public.keepalive() from public;
grant execute on function public.keepalive() to anon, authenticated;

notify pgrst, 'reload schema';
```

Fonksiyon veri okumaz/yazmaz, sadece sunucu saatini dondurur. RLS'e tabi tablo okumaya gerek yoktur;
projede `anon` rolune tablo `SELECT` grant'i verilmedigi icin tablo ping'leri `401 permission denied`
donuyordu.

Fonksiyon silinirse workflow otomatik olarak `/auth/v1/health` yedegine duser, job yine basarili olur.

## Notlar

- GitHub, **60 gun** boyunca commit/aktivite olmayan repolarda zamanlanmis workflow'lari devre disi
  birakir. Actions sekmesinden kontrol et; gerekirse workflow'u yeniden enable et.
- GitHub cron zamanlari yogun saatlerde dakikalar (bazen 1 saati asan) gecikebilir; gunluk calisma
  bu yuzden 7 gunluk pencereye genis pay birakir.
- Elle test: `Actions -> Supabase Keep-Alive -> Run workflow`.
