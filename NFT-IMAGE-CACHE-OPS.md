# NFT Image Cache — Ops Runbook

> **This is critical infrastructure documentation.**
> Every agent working on NFT-related projects must read this before touching image pipelines, sync crons, or collection onboarding.

## Architecture Overview

```
Source Images (arweave/IPFS/GCS/project APIs)
        ↓ download
  cache_images.py (pg-crons)
        ↓ convert to 512px WebP q80
  S3: pg-nft-images (ap-southeast-1)
        ↓
  CloudFront → images.pfpvault.com (CDN)
        ↓
  API returns cached_image_url to clients
```

**Full public docs:** https://blockchainsuperheroes.github.io/pg-nft-data-docs/#images

## Key Infrastructure

| Component | Location | Details |
|-----------|----------|---------|
| Cacher script | `pg-crons (18.143.147.51):/var/www/nft-sync/nft-image-cache/cache_images.py` | Main batch cacher |
| S3 bucket | `pg-nft-images` (ap-southeast-1) | All cached images |
| CloudFront | `E3909D5UKHNJZ8` → `images.pfpvault.com` | CDN distribution |
| NFT DB | `pg-db (172.31.46.190):5432` / `pg_nft_db` | `cached_image_url`, `original_image_url` columns |
| Source-of-truth DB | `pg-be-master-v3 (35.225.2.90)` Docker `postgres-master` / `chainguardians` | Canonical NFT data |
| Sync cron | `pg-be-master-v3 (35.225.2.90)` supervisor `nft-sync-cron` | Ownership + metadata sync |
| Custom data | `/var/www/nft-sync/nft-sync-cron/app/custom_nft_data.py` | Per-collection overrides |

## S3 Path Layout

```
s3://pg-nft-images/
├── {chain_id}/{contract_address}/{token_id}.webp    ← CDN-served, optimized
├── originals/{chain_id}/{contract}/{token_id}.{ext}  ← archival full-size (NOT on CDN)
└── history/{chain_id}/{contract}/{token_id}/{ts}.webp ← change snapshots (planned)
```

## Image Spec

- **Format:** WebP
- **Max dimension:** 512px (maintains aspect ratio)
- **Quality:** 80
- **Avg size:** 30–50 KB (down from 1–6 MB originals)
- **Cache headers:** `public, max-age=31536000, immutable`
- **URL pattern:** `https://images.pfpvault.com/{chain_id}/{contract_address}/{token_id}.webp`

## Four-Tier Image Fallback

1. **Cached CDN** — `images.pfpvault.com` WebP (preferred, fastest)
2. **Project-Hosted Thumbnails** — e.g. GCS buckets when projects provide them
3. **On-Chain Metadata** — tokenURI → JSON → image (arweave, IPFS, etc.)
4. **Original Image** — captured at first sync, stored in DB `original_image_url`

## Smart Caching Rules

### Same-Image Collections (CRITICAL)
Many collections (especially BCSH heroes across chains) use **one image for all tokens**. The cacher MUST detect this:
- Query `SELECT COUNT(DISTINCT image) FROM nfts WHERE asset_contract_id = X`
- If 1 unique image: **download once, S3 server-side copy to all token slots**
- Saves hours vs downloading the same image thousands of times

### Variant Collections
Some collections have multiple images (e.g. BCSH OASYS has 3 Setsuko variants):
- Group tokens by source image URL
- Download each unique image once, copy to grouped tokens
- `custom_nft_data.py` must correctly differentiate variants

### Project-Hosted Sources
When on-chain metadata points to slow/unreliable sources but the project has better ones:
- **Killer GF:** On-chain → arweave (5.9MB PNGs, frequent timeouts). Use `https://storage.googleapis.com/kgf-thumbnails/{id}.jpg` instead
- Document alternative sources in this file when discovered

## Adding a New Collection — Checklist

1. **Register contract** in `nft_contract` table (both `chainguardians` and `pg_nft_db`)
2. **Check image source reliability:**
   - Is metadata on arweave/IPFS? Test download speed + timeout rate
   - Does project have a thumbnail server? (GCS, S3, custom API)
   - Is it a same-image collection? (heroes, passes, etc.)
3. **Add to `custom_nft_data.py`** if collection needs special name/image handling
4. **Run initial cache:** `sudo /var/www/nft-sync/nft-sync-cron/venv/bin/python3 cache_images.py {chain_id}`
5. **Verify in S3:** `aws s3 ls s3://pg-nft-images/{chain_id}/{contract}/ --region ap-southeast-1 | wc -l`
6. **Verify in DB:** `SELECT COUNT(cached_image_url) FROM nfts WHERE asset_contract_id = X`
7. **CloudFront invalidation** (only if overwriting existing images):
   ```bash
   aws cloudfront create-invalidation --distribution-id E3909D5UKHNJZ8 \
     --paths "/{chain_id}/{contract}/*"
   ```

## Running the Cacher

```bash
# SSH to pg-crons
ssh ubuntu@18.143.147.51

# Single chain
sudo /var/www/nft-sync/nft-sync-cron/venv/bin/python3 \
  /var/www/nft-sync/nft-image-cache/cache_images.py {chain_id}

# Batch size is 500 per collection per run — run multiple passes for large collections

# Smart BCSH cacher (same-image optimization)
sudo /var/www/nft-sync/nft-sync-cron/venv/bin/python3 /tmp/smart_bcsh_cache.py
```

## Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Arweave 5.9MB PNGs timing out | Large source images, 30s timeout | Use project thumbnail server if available |
| Arweave 404 | Content unpinned or gateway issue | Try alternate gateways, or use project-hosted source |
| All tokens show same image | `custom_nft_data.py` hardcodes base variant | Update variant mapping in custom_nft_data.py |
| Duplicate NFT rows | `store_nft()` didn't check existing | Patched: now checks before insert, updates owner only |
| CloudFront serving stale images | 1-year immutable cache | Must create CloudFront invalidation after S3 overwrite |
| BCSH images all wrong after re-sync | Sync overwrites corrected variant data | `custom_nft_data.py` must differentiate variants |

## DB Fields Reference

```sql
-- pg_nft_db.nfts
image              -- current source URL from on-chain metadata
cached_image_url   -- our CDN WebP URL (NULL = not yet cached)
original_image_url -- source URL at time of caching
```

## AWS Credentials

On pg-crons: `~/.aws/credentials` (IAM user Idon, ap-southeast-1)

## Change Log

- **2026-05-13:** Smart caching for same-image collections, BCSH all chains cached
- **2026-05-13:** KGF switched to GCS thumbnail source
- **2026-05-13:** `store_nft()` patched to prevent duplicate rows
- **2026-05-13:** `custom_nft_data.py` needs update for OASYS variants (TODO)
- **2026-05-13:** Docs updated to four-tier fallback + archival plan
