# Sanctuary Chorus Listening Bands: Fish Chorus Spectrograms with Ranked Candidate Analysis Bands

## Overview

This dataset contains 949 mel-spectrogram images of 60-second underwater recordings made during logged fish choruses by hydrophones at four sites in Channel Islands and Monterey Bay National Marine Sanctuaries, from 33 deployments recorded between March 2019 and July 2024. Each recording comes with a ranking of six fixed candidate listening bands, from the band in which the chorus period stands out most above the site's chorus-free sound level to the band in which it stands out least.

The recordings come from the Sanctuary Soundscape Monitoring Project (SanctSound) and its continuation by the NOAA Office of National Marine Sanctuaries, and the chorus logs from the NOAA Office of National Marine Sanctuaries. All are U.S. Government works in the public domain. This release contains no raw audio, no timestamps, no locations and none of the chorus-free recordings used to compute the rankings; sites and deployments are identified only by opaque keys.

## Release At A Glance

- Raw files: 9
- Recordings (cases): 949, each at least half inside a logged chorus
- Sites: 4 (two in Channel Islands, two in Monterey Bay); deployments: 33
- Recording length: 60 seconds; spectrogram shape (240, 64)
- Candidate bands: 6, fixed for every recording
- Best band across the release: band 0 in 426 recordings, band 1 in 208, band 2 in 17, band 3 in 10, band 4 in 39, band 5 in 249
- Prepared split: 23 deployments / 636 training recordings, 10 other deployments / 313 test recordings

## Raw File Structure

The uploaded ZIP is flat and contains exactly these files:

- `cases.csv`: one record per recording: `case_id`, `site_id`, `deployment_id`, `ranking`.
- `spectrograms.npz`: one uint8 array of shape (240, 64) per `case_id`.
- `test_deployments.txt`: the `deployment_id` values held out for the prepared test split, one per line.
- `LICENSE`: CC0 1.0 notice.
- `ATTRIBUTION.txt`: source and attribution text.
- `DATASET_CARD.md`: short scope summary.
- `DATASET_DESCRIPTION.md`: this document.
- `source_metadata.json`: provenance and processing parameters.
- `PACKAGE_MANIFEST.sha256`: SHA-256 checksum of every other raw file.

## Columns

### cases.csv

- `case_id` (string): opaque recording identifier, for example `c_3f9c1a7b2e`.
- `site_id` (string): opaque hydrophone-site key, for example `site_0b1c2d3e`.
- `deployment_id` (string): opaque deployment key, for example `dep_91e2c0d4`. A deployment is one continuous recorder placement at a site.
- `ranking` (string): the six candidate band numbers 0 to 5 separated by spaces, best first.

### spectrograms.npz

- One array per `case_id`, shape (240, 64), dtype uint8.
- Rows run along the horizontal axis of the image; row i covers seconds 0.25 i to 0.25 (i + 1).
- Columns are 64 mel bands spaced on the mel scale from 10 Hz to 4 kHz, low to high.
- A value v means log10 mel power = v * 0.04 - 9 (values are rounded and clipped to 0 to 255).

### Candidate bands

Band k covers mel columns 8k to 8k + 15, so neighbouring bands overlap by half:

- band 0: columns 0-15, about 10 to 464 Hz
- band 1: columns 8-23, about 196 to 769 Hz
- band 2: columns 16-31, about 431 to 1,154 Hz
- band 3: columns 24-39, about 727 to 1,639 Hz
- band 4: columns 32-47, about 1,100 to 2,252 Hz
- band 5: columns 40-55, about 1,572 to 3,025 Hz

## How The Spectrograms Are Made

- Audio is read from the archived recordings (48 kHz) at the exact sample where the recording starts and resampled to 16 kHz, mono.
- Power spectra use a 2,048-sample Hann window with a 160-sample hop, are mapped to 64 triangular mel bands (area-normalised) between 10 Hz and 4 kHz, averaged over 25 consecutive hops (0.25 seconds) and converted to log10.
- Each recording then receives a keyed gain offset drawn uniformly between -0.6 and +0.6 in log10 units (-6 to +6 dB), so absolute levels are not comparable between recordings.

## How The Recordings And Rankings Are Made

- Deployments with archived audio and a complete fish chorus log are used (sites CI01, CI04, MB01, MB02). Within each, 60-second recordings are drawn at keyed random times, at least 30 minutes apart, avoiding 10 minutes around every recording released in the Sanctuary Fish Chorus Masks dataset and 12 hours around log rows with a missing time.
- Chorus recordings lie at least half inside a logged chorus of any of the five logged types (bocaccio, plainfin midshipman, white seabass, UF440, UF310). Chorus-free recordings, with no logged chorus within 30 minutes, are drawn from the same deployments and are not released.
- For each chorus recording and each band, the excess is the median over the recording's logged-chorus rows of the band's mean log10 power (before the keyed gain) minus the site's chorus-free level in that band (the median over all chorus-free recordings of the site). The ranking orders bands by decreasing excess. Recordings whose two best bands differ by less than 0.02 log10 units are dropped.
- All draws and identifiers derive by HMAC-SHA256 from a secret held by the creator; the secret, recording times, file names and chorus-free recordings are not released.

## Label Source

The chorus logs are the NOAA Office of National Marine Sanctuaries fish chorus logs. According to their metadata, analysts manually scanned long-term spectral averages in species-specific frequency bands between 10 and 1,000 Hz, defining a chorus as sustained acoustic activity at least 3 dB above ambient for minutes to hours. Every deployment was scanned for all five types.

## Prepared Outputs

`prepare.py` writes a public directory and a private answer directory.

- public `train.csv` (636 rows) and `test.csv` (313 rows): `case_id`, `site_id`, `deployment_id`.
- public `train_labels.csv` (636 rows): `case_id`, `ranking`, all six bands best first.
- public `sample_submission.csv` (313 rows): `case_id`, `ranking` from a band-contrast rule.
- public `spectrograms.npz`: the arrays of every training and test recording.
- public `LICENSE`.
- public `data_manifest.json`: array shape, decoding, candidate bands in mel columns and Hz, split counts and the independent unit.
- private `answers.csv` (313 rows): `case_id`, `ranking` for the test recordings. It has the same columns as `sample_submission.csv`.

The test split is the deployments listed in `test_deployments.txt`. The preparation self-check verifies that deployments do not cross the split, that ids agree across files, that every ranking is a permutation of the six bands and that no public column is constant.

## Known Limitations

- **Four sites.** All recordings come from two Channel Islands and two Monterey Bay sites, and every site appears in both splits; transfer is across deployments, seasons and years.
- **Excess, not chorus alone.** The excess measures the whole sound during a logged chorus against the site's chorus-free level, so wind, rain, vessels or snapping shrimp that coincide with a chorus also move it.
- **Uneven best bands.** Bands 2 and 3 are rarely best; the scoring corrects for the most frequent bands.
- **Chorus-level logs.** Logs mark sustained choruses, not individual calls.

## Provenance And License

Sources: NOAA Office of National Marine Sanctuaries and U.S. Navy, Sanctuary Soundscape Monitoring Project (SanctSound), and NOAA Office of National Marine Sanctuaries passive acoustic recordings and fish chorus logs, archived at NOAA National Centers for Environmental Information. U.S. Government works in the public domain. This derived release is dedicated to the public domain under CC0 1.0.
