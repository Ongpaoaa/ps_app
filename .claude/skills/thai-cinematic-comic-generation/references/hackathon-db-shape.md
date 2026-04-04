# Hackathon DB Shape

Use this file when the user wants comic content stored in hackathon activity data.

## Current runtime shape

The current hackathon activity screen reads comics from a `text` content row plus `metadata.panels[]`.

Default safe shape:

```json
{
  "title": "What You'll Walk Away With",
  "instructions": "By the end of Phase 1, you'll know how to find a good problem, validate a real pain point, and define the right target user with our guide.",
  "content_type": "text",
  "content_title": "What You'll Walk Away With",
  "content_body": "By the end of this phase, you'll know how to find a good problem, validate a real pain point, and define the right target user with our guide.",
  "metadata": {
    "variant": "evidence_first",
    "panels": [
      {
        "id": "setup",
        "order": 1,
        "headline": "Thai or English panel headline",
        "body": "Thai or English supporting line",
        "image_key": "portrait-panel-key",
        "accent": "cyan"
      }
    ]
  }
}
```

## Tables to patch

### `hackathon_phase_activities`

Owns:

- `title`
- `instructions`
- `display_order`
- `estimated_minutes`

### `hackathon_phase_activity_content`

Owns:

- `content_type`
- `content_title`
- `content_body`
- `content_url`
- `display_order`
- `metadata`

## Live patch workflow

1. Query the current activity row and nested content rows first.
2. Confirm the actual production IDs.
3. Generate the comic prompts and assets.
4. Update the exact content row.
5. Read it back.
6. Sync the additive seed, destructive phase seed, and preview fallback.

## Readback checklist

- activity `title`
- activity `instructions`
- content `content_type`
- content `content_title`
- content `content_body`
- every `metadata.panels[].headline`
- every `metadata.panels[].body`
- every `metadata.panels[].image_key` or `image_url`
- panel ordering

## Sync files after live patch

- `supabase/seed/hackathon_customer_discovery_phase1.sql`
- `supabase/seed/hackathon_phase1_activities_complete.sql`
- `lib/hackathonProgramPreview.ts`
