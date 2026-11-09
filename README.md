# YouTube Download to Cloud Storage

Download YouTube video or audio to your cloud storage, then retrieve job status and the delivery manifest.

Powered by the [FlashData YouTube Download to Cloud Storage API](https://flashdata.dev/docs/sources/youtube_download?utm_source=github-actions&utm_medium=integration&utm_campaign=intent_gtm_2026q4&utm_content=youtube_download-task). Paid asynchronous job. Requires customer storage. Save the receipt; never submit again to poll. Result JSON expires after 24 hours; media retention depends on storage.

## Quick start

Add a [FlashData API key](https://flashdata.dev/signup?utm_source=github-actions&utm_medium=integration&utm_campaign=intent_gtm_2026q4&utm_content=youtube_download-task) as the repository secret `FLASHDATA_API_KEY`. API calls require FlashData Credits ([pricing](https://flashdata.dev/pricing?utm_source=github-actions&utm_medium=integration&utm_campaign=intent_gtm_2026q4&utm_content=youtube_download-task)). Then add this workflow:

```yaml
name: YouTube Download to Cloud Storage
on:
  workflow_dispatch:
    inputs:
      job_id:
        description: Previously saved FlashData download job ID
        required: true
        type: string
permissions:
  contents: read
jobs:
  collect:
    runs-on: ubuntu-latest
    steps:
      - name: YouTube Download to Cloud Storage
        id: data
        uses: rocosen/flashdata-youtube-download-action@v0.1.2
        with:
          api-key: ${{ secrets.FLASHDATA_API_KEY }}
          operation: result
          job-id: ${{ inputs.job_id }}
      - name: Upload results
        uses: actions/upload-artifact@v4
        with:
          name: youtube-download
          path: ${{ steps.data.outputs.artifact-directory }}
          retention-days: 7
```

Run it from the **Actions** tab. Download the exported files from the run’s **Artifacts**, or use the output paths in later steps. Requires an up-to-date runner with Node 24 support; no dependency installation is needed. For production, pin actions to a reviewed commit SHA.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `api-key` | Yes | FlashData API Key. Pass secrets.FLASHDATA_API_KEY; requests use paid Credits. |
| `video-id` | No | 11-character YouTube video ID. |
| `options-json` | No | Optional JSON object with only source-specific options listed in the README. Never include credentials here. |
| `operation` | No | status or result reads a saved job; submit creates one paid job. Default: `status`. |
| `job-id` | No | Saved job ID for status/result. Do not pass submission inputs when resuming. |
| `allow-submit` | No | Must be exactly true to submit a new paid storage write. Default: `false`. |
| `storage-json` | No | For submit only: customer storage JSON from a GitHub Actions secret. |

### Options

Pass these fields in `options-json`:

`download_type` (audio_video, video, audio), `video_quality` (best, worst, or <=/==/>= plus 144, 360, 480, 720, 1080, 1440, 2160, 4320), `fps` (>=/<=/== and 1–240), `audio_language` (default, original), `audio_quality` (best, 320, 256, 192, 128), `output_format` (video: mp4/webm/mkv; audio: m4a/mp3/opus), `yt_dlp_format`, `upload_metadata` (boolean), `start_at`/`end_at` (HH:MM:SS, end after start, at most 12 hours). Default audio_video/<=720/mp4; audio defaults to m4a/best. Advanced yt_dlp_format overrides video quality, fps and audio language selection. Read operations do not accept options-json.

Invalid options fail before an API request. Read operations require job-id; new submissions require allow-submit and storage-json.

## Outputs

- `response.json`: API response, including available result fields and usage; download lifecycle fields are sanitized.
- `records.jsonl`: job receipt, status or completed delivery manifest.
- `manifest.json`: source, operation, timestamps, record count and export warnings.

Returned API fields: submit: id/status receipt; reads: job lifecycle; completed result: output JSON delivery manifest.

Empty results produce an empty JSONL file and `result-count: 0`. Optional fields may be absent.

| Step output | Description |
| --- | --- |
| `artifact-directory` | Absolute directory with response.json, records.jsonl and manifest.json. |
| `results-file` | Absolute path to the original response JSON (sanitized lifecycle fields for download jobs). |
| `records-file` | Absolute path to JSONL records for downstream steps. Empty file for an empty result. |
| `result-count` | Number of exported records; not the requested result count. |
| `source` | The fixed FlashData source used by this action. |
| `job-id` | Persist this receipt and reuse it for status/result. |
| `job-status` | Current lifecycle status. A successful action does not mean the job has completed. |

## Download a video or audio file

The quick-start example reads an existing job. To submit a new download to your storage:

```yaml
- uses: rocosen/flashdata-youtube-download-action@v0.1.2
  id: submit
  with:
    api-key: ${{ secrets.FLASHDATA_API_KEY }}
    operation: submit
    allow-submit: 'true'
    video-id: YOUR_VIDEO_ID
    options-json: '{"download_type":"audio","output_format":"m4a"}'
    storage-json: ${{ secrets.FLASHDATA_STORAGE_JSON }}
```

Replace `YOUR_VIDEO_ID` with an 11-character ID you may process. Set `FLASHDATA_STORAGE_JSON` to your storage configuration in Actions secrets. Supported providers: S3, S3-compatible, GCS, Azure, OSS, BOS, COS and TOS. [Storage setup and required fields](https://flashdata.dev/docs/sources/youtube_download?utm_source=github-actions&utm_medium=integration&utm_campaign=intent_gtm_2026q4&utm_content=youtube_download-task).

Save `steps.submit.outputs.job-id`. Use it with `operation: status` or `result`, omitting submission inputs. Reads do not create a new download or poll automatically. A pending/running job is not a completed delivery. Result JSON expires after 24 hours; media retention follows your storage policy. Delivery manifests may contain private signed links.

## Pricing and errors

The action is free to install. FlashData API calls use paid Credits; GitHub runner/artifact and cloud storage costs are separate. Media delivery is billed by delivered volume. [Current pricing](https://flashdata.dev/pricing?utm_source=github-actions&utm_medium=integration&utm_campaign=intent_gtm_2026q4&utm_content=youtube_download-task).

401: check the API key. 402: add Credits. 429: rate/quota limit. After a timeout, check [FlashData Jobs](https://flashdata.dev?utm_source=github-actions&utm_medium=integration&utm_campaign=intent_gtm_2026q4&utm_content=youtube_download-task) before rerunning: a request may already have been accepted. Paid POST requests are not retried. Resume downloads using the saved job ID. Requests have a 65-second timeout and a 16-MiB response limit.

## Development

`npm ci` → `npm run check` → `npm test` → `npm run build`. Tests use offline fixtures. Commit the bundle with runtime changes.
