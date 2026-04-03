# Lyrics Plus API Documentation

## Overview

The Lyrics Plus API provides access to synchronized song lyrics with timestamp data from multiple sources including Apple Music, Spotify, Musixmatch, and LyricsPlus.

### Base URLs

The API is available on multiple servers for redundancy and performance:

* `https://lyricsplus.prjktla.workers.dev` (Primary - Cloudflare Workers)
* `https://lyrics-plus-backend.vercel.app` (Vercel)

**Recommended:** Use the primary URL or implement fallback logic to try the alternative server if one is unavailable.

---

## Endpoints

### Get Lyrics

Retrieves synchronized lyrics for a specified song with detailed timing information.

**Endpoint:** `GET /v2/lyrics/get`

#### Query Parameters

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `title` | string | Yes | The song title (URL encoded) |
| `artist` | string | Yes | The artist name(s) (URL encoded, comma-separated for multiple artists) |
| `album` | string | No | The album name (URL encoded) |
| `duration` | number | No | Song duration in seconds (helps with matching accuracy) |
| `source` | string | No | Comma-separated list of sources to query. Options: `apple`, `lyricsplus`, `musixmatch`, `spotify`, `musixmatch-word` |

#### Example Request

```
GET /v2/lyrics/get?title=NOT%20CUTE%20ANYMORE&artist=ILLIT&album=NOT%20CUTE%20ANYMORE&duration=132&source=apple,lyricsplus,musixmatch,spotify,musixmatch-word
```

#### Example Request (Decoded)

```
GET /v2/lyrics/get
  ?title=NOT CUTE ANYMORE
  &artist=ILLIT
  &album=NOT CUTE ANYMORE
  &duration=132
  &source=apple,lyricsplus,musixmatch,spotify,musixmatch-word
```

---

## TypeScript Types

```typescript
interface LyricsResponse {
  KpoeTools: string;
  type: "Word" | "Line";
  metadata: {
    source: string;
    songWriters: string[];
    title: string;
    language: string;
    agents: {
      [key: string]: {
        type: "person";
        name: string;
        alias: string;
      };
    };
    songParts: SongPart[];
    totalDuration: string;
    // Present when source is lyricsplus
    artist?: string;
    album?: string;
    leadingSilence?: string;
  };
  lyrics: LyricLine[];
  cached: "GDrive" | null;
  processingTime: {
    timeElapsed: number;
    lastProcessed: number;
  };
}

interface SongPart {
  name: string;
  time: number;
  duration: number;
}

interface LyricLine {
  time: number;
  duration: number;
  text: string;
  syllabus: SyllabusItem[];
  element: {
    key: string;
    songPartIndex: number;
    singer: string;
  };
  translation?: {
    lang: string;
    text: string;
  };
  transliteration?: {
    lang: string;
    text: string;
    syllabus: SyllabusItem[];
  };
}

interface SyllabusItem {
  time: number;
  duration: number;
  text: string;
  isBackground?: boolean;
}
```

---

## Response Format

### Response Structure

```json
{
  "KpoeTools": "1.7-1-ConvertTTMLtoJSON-DOMParser",
  "type": "Word | Line",
  "metadata": {
    "source": "Apple",
    "songWriters": ["string"],
    "title": "string",
    "language": "string",
    "agents": {
      "v1": {
        "type": "person",
        "name": "string",
        "alias": "string"
      }
    },
    "songParts": [
      {
        "name": "string",
        "time": "number",
        "duration": "number"
      }
    ],
    "totalDuration": "string"
  },
  "lyrics": [
    {
      "time": "number",
      "duration": "number",
      "text": "string",
      "syllabus": [
        {
          "time": "number",
          "duration": "number",
          "text": "string",
          "isBackground": "boolean"
        }
      ],
      "element": {
        "key": "string",
        "songPartIndex": "number",
        "singer": "string"
      },
      "translation": {
        "lang": "string",
        "text": "string"
      },
      "transliteration": {
        "lang": "string",
        "text": "string",
        "syllabus": [
          {
            "time": "number",
            "duration": "number",
            "text": "string"
          }
        ]
      }
    }
  ],
  "cached": "GDrive | null",
  "processingTime": {
    "timeElapsed": "number",
    "lastProcessed": "number"
  }
}
```

### Field Descriptions

#### Root Level

| Field | Type | Description |
| --- | --- | --- |
| `KpoeTools` | string | Version identifier for the conversion tool (e.g. `"1.7-1-ConvertTTMLtoJSON-DOMParser"` for Apple source, `"1.0-LPlusBcknd"` for lyricsplus source) |
| `type` | string | Synchronization type: `"Word"` for word-level timing, `"Line"` for line-level timing |
| `metadata` | object | Song metadata including agents, song structure, and duration |
| `lyrics` | array | Array of lyric line objects with timing data |
| `cached` | string \| null | Cache source if the result was served from cache (e.g. `"GDrive"`), or `null` if freshly fetched |
| `processingTime` | object | Timing information about how long the request took to process |

#### Processing Time Object

| Field | Type | Description |
| --- | --- | --- |
| `timeElapsed` | number | Time taken to process the request in milliseconds |
| `lastProcessed` | number | Unix timestamp (ms) of when the lyrics were last processed |

#### Metadata Object

| Field | Type | Description |
| --- | --- | --- |
| `source` | string | Data source (e.g., `"Apple"`, `"lyricsplus"`) |
| `songWriters` | array | List of song writers/composers |
| `title` | string | Song title |
| `language` | string | Language code (e.g., `"en"`, `"ko"`) |
| `agents` | object | Information about performers/singers (see Agents Object below) |
| `songParts` | array | Ordered list of song sections with timing (see Song Parts Array below) |
| `totalDuration` | string | Total song duration in `MM:SS.mmm` format |
| `artist` | string | *(lyricsplus source only)* Artist name |
| `album` | string | *(lyricsplus source only)* Album name |
| `leadingSilence` | string | *(lyricsplus source only)* Duration of leading silence before lyrics begin (e.g. `"0.000"`) |

#### Agents Object

Lives inside `metadata`. Each agent is keyed by an identifier (e.g., `"v1"`, `"v2"`) and contains:

| Field | Type | Description |
| --- | --- | --- |
| `type` | string | Agent type (typically `"person"`) |
| `name` | string | Performer name (may be an empty string if not provided by source) |
| `alias` | string | Agent identifier used in lyrics |

#### Song Parts Array

Lives inside `metadata.songParts`. Provides a full map of the song's structure with timestamps.

| Field | Type | Description |
| --- | --- | --- |
| `name` | string | Section name (e.g., `"Verse"`, `"Chorus"`, `"Bridge"`, `"PreChorus"`) |
| `time` | number | Start time of this section in milliseconds |
| `duration` | number | Duration of this section in milliseconds |

#### Lyrics Array Item

| Field | Type | Description |
| --- | --- | --- |
| `time` | number | Start time in milliseconds |
| `duration` | number | Duration in milliseconds |
| `text` | string | Full line of lyrics |
| `syllabus` | array | Word/syllable-level timing breakdown |
| `element` | object | Metadata about the lyric line |
| `translation` | object | *(Optional)* Human translation of the line into another language |
| `transliteration` | object | *(Optional)* Romanized version of non-Latin script lyrics |

#### Syllabus Array Item

| Field | Type | Description |
| --- | --- | --- |
| `time` | number | Start time in milliseconds |
| `duration` | number | Duration in milliseconds |
| `text` | string | Individual word or syllable |
| `isBackground` | boolean | *(Optional)* If `true`, indicates background vocals or ad-libs |

#### Element Object

| Field | Type | Description |
| --- | --- | --- |
| `key` | string | Unique line identifier (e.g., `"L1"`, `"L2"`) |
| `songPartIndex` | number | Index into `metadata.songParts` indicating which section this line belongs to |
| `singer` | string | Agent identifier of the singer (e.g., `"v1"`); may be an empty string |

#### Translation Object (Optional)

For songs where a human translation is available (e.g. Korean lyrics translated to English):

| Field | Type | Description |
| --- | --- | --- |
| `lang` | string | Language code of the translation (e.g., `"en-US"`) |
| `text` | string | Translated version of the full line |

> **Note:** `translation` contains only the translated text with no syllabus timing. It is distinct from `transliteration`, which provides romanized pronunciation with word-level timing.

#### Transliteration Object (Optional)

For non-Latin script songs (e.g., Korean, Japanese), provides romanized text with timing:

| Field | Type | Description |
| --- | --- | --- |
| `lang` | string | Language code for transliteration (e.g., `"ko-Latn"` for Korean romanized to Latin script) |
| `text` | string | Romanized version of the full line |
| `syllabus` | array | Word/syllable-level romanized timing breakdown |

---

## Synchronization Types

The API returns different levels of timing precision based on availability:

### Quick Reference

| Feature | Word-Level (`type: "Word"`) | Line-Level (`type: "Line"`) |
| --- | --- | --- |
| **Precision** | Word/syllable timing | Line timing only |
| **`syllabus` array** | Contains word data | Empty `[]` |
| **`isBackground` field** | Available in syllabus items | Not available |
| **Best for** | Karaoke, real-time highlighting | Basic lyric display |
| **File size** | Larger | Smaller |

### Word-Level Sync (`type: "Word"`)

Provides precise timing for each word/syllable in the `syllabus` array. Best for karaoke and real-time highlighting applications.

**Characteristics:**

* `type` field is `"Word"`
* `syllabus` array contains detailed word/syllable timing
* Each syllabus item includes `time`, `duration`, and `text`
* May include `isBackground: true` for backing vocals

**Example:**

```json
{
  "time": 12856,
  "duration": 2692,
  "text": "Tell 'em Kendrick did it, aye, who showed you how to run a blitz?",
  "syllabus": [
    { "time": 12856, "duration": 116, "text": "Tell " },
    { "time": 12972, "duration": 180, "text": "'em " },
    { "time": 13152, "duration": 384, "text": "Kendrick " }
  ],
  "element": {
    "key": "L1",
    "songPartIndex": 0,
    "singer": "v1"
  }
}
```

### Line-Level Sync (`type: "Line"`)

Provides timing only for complete lines. The `syllabus` array will be empty.

**Characteristics:**

* `type` field is `"Line"`
* `syllabus` array is empty `[]`
* Only line-level timing available
* Simpler structure for basic lyric display

**Example:**

```json
{
  "time": 590,
  "duration": 3219,
  "text": "These red roses damn near turn to ashes",
  "syllabus": [],
  "element": {
    "key": "L39608",
    "songPartIndex": 0,
    "singer": ""
  }
}
```

---

## Example Responses

### Word-Level Sync with Background Vocals (Apple)

```json
{
  "KpoeTools": "1.7-1-ConvertTTMLtoJSON-DOMParser",
  "type": "Word",
  "metadata": {
    "source": "Apple",
    "songWriters": [
      "Kendrick Lamar",
      "Jack Antonoff",
      "Hitta J3"
    ],
    "title": "gnx",
    "language": "en",
    "agents": {
      "v1": {
        "type": "person",
        "name": "",
        "alias": "v1"
      }
    },
    "songParts": [
      {
        "name": "Chorus",
        "time": 12856,
        "duration": 19774
      },
      {
        "name": "Verse",
        "time": 33207,
        "duration": 41366
      }
    ],
    "totalDuration": "3:13.538"
  },
  "lyrics": [
    {
      "time": 12856,
      "duration": 2692,
      "text": "Tell 'em Kendrick did it, aye, who showed you how to run a blitz?",
      "syllabus": [
        { "time": 12856, "duration": 116, "text": "Tell " },
        { "time": 12972, "duration": 180, "text": "'em " },
        { "time": 13152, "duration": 384, "text": "Kendrick " }
      ],
      "element": {
        "key": "L1",
        "songPartIndex": 0,
        "singer": "v1"
      }
    },
    {
      "time": 9798,
      "duration": 7698,
      "text": "Never fear the fall (Phoenix)",
      "syllabus": [
        { "time": 9798, "duration": 464, "text": "Never " },
        { "time": 10262, "duration": 848, "text": "fear the " },
        { "time": 11110, "duration": 986, "text": "fall " },
        { "time": 14497, "duration": 237, "text": "(Phoe", "isBackground": true },
        { "time": 14734, "duration": 2762, "text": "nix)", "isBackground": true }
      ],
      "element": {
        "key": "L3",
        "songPartIndex": 0,
        "singer": "v1"
      }
    }
  ],
  "cached": "GDrive",
  "processingTime": {
    "timeElapsed": 1029,
    "lastProcessed": 1775246566262
  }
}
```

### Word-Level Sync with Transliteration and Translation (Korean)

```json
{
  "KpoeTools": "1.7-1-ConvertTTMLtoJSON-DOMParser",
  "type": "Word",
  "metadata": {
    "source": "Apple",
    "songWriters": ["C'SA", "Danke", "ENDS"],
    "title": "Phoenix",
    "language": "ko",
    "agents": {
      "v1": {
        "type": "person",
        "name": "",
        "alias": "v1"
      }
    },
    "songParts": [
      {
        "name": "Verse",
        "time": 28728,
        "duration": 13909
      }
    ],
    "totalDuration": "2:44.000"
  },
  "lyrics": [
    {
      "time": 28728,
      "duration": 2915,
      "text": "한 줌 흙 속에 잠겨",
      "syllabus": [
        { "time": 28728, "duration": 363, "text": "한 " },
        { "time": 29091, "duration": 424, "text": "줌 " },
        { "time": 30171, "duration": 595, "text": "흙 속에 " },
        { "time": 30766, "duration": 877, "text": "잠겨" }
      ],
      "element": {
        "key": "L7",
        "songPartIndex": 0,
        "singer": "v1"
      },
      "translation": {
        "lang": "en-US",
        "text": "Buried in a handful of soil"
      },
      "transliteration": {
        "lang": "ko-Latn",
        "text": "han  jum  heuk  so ge  jam gyeo",
        "syllabus": [
          { "time": 28728, "duration": 363, "text": "han  " },
          { "time": 29091, "duration": 424, "text": "jum  " },
          { "time": 30171, "duration": 595, "text": "heuk  so ge  " },
          { "time": 30766, "duration": 877, "text": "jam gyeo" }
        ]
      }
    }
  ],
  "cached": null,
  "processingTime": {
    "timeElapsed": 942,
    "lastProcessed": 1775246580605
  }
}
```

### Line-Level Sync (lyricsplus source)

```json
{
  "KpoeTools": "1.0-LPlusBcknd",
  "type": "Line",
  "metadata": {
    "source": "lyricsplus",
    "songWriters": ["Miguel Jontel Pimentel", "Jhené Aiko"],
    "title": "HAPPINESS OVER EVERYTHING (H.O.E.)",
    "language": "en",
    "artist": "Jhené Aiko",
    "album": "Chilombo (Explicit)",
    "leadingSilence": "0.000",
    "agents": {},
    "songParts": [
      {
        "name": "Verse",
        "time": 590,
        "duration": 3219
      }
    ],
    "totalDuration": "3:06.017"
  },
  "lyrics": [
    {
      "time": 590,
      "duration": 3219,
      "text": "These red roses damn near turn to ashes",
      "syllabus": [],
      "element": {
        "key": "L39608",
        "songPartIndex": 0,
        "singer": ""
      }
    },
    {
      "time": 3950,
      "duration": 3299,
      "text": "If I keep it real, you won't understand it",
      "syllabus": [],
      "element": {
        "key": "L39609",
        "songPartIndex": 0,
        "singer": ""
      }
    }
  ],
  "cached": null,
  "processingTime": {
    "timeElapsed": 6614,
    "lastProcessed": 1775246546881
  }
}
```

---

## Code Examples

### JavaScript/TypeScript — Fetching Lyrics

```typescript
async function getLyrics(
  title: string,
  artist: string,
  options?: {
    album?: string;
    duration?: number;
    sources?: string[];
  }
): Promise<LyricsResponse> {
  const params = new URLSearchParams({ title, artist });

  if (options?.album) params.append("album", options.album);
  if (options?.duration) params.append("duration", options.duration.toString());
  if (options?.sources) params.append("source", options.sources.join(","));

  const response = await fetch(
    `https://lyricsplus.prjktla.workers.dev/v2/lyrics/get?${params}`
  );

  if (!response.ok) {
    throw new Error(`Failed to fetch lyrics: ${response.statusText}`);
  }

  return response.json();
}

// Example usage
const lyrics = await getLyrics("NOT CUTE ANYMORE", "ILLIT", {
  album: "NOT CUTE ANYMORE",
  duration: 132,
  sources: ["apple", "lyricsplus", "musixmatch", "spotify"],
});

console.log(lyrics.type);                    // "Word" or "Line"
console.log(lyrics.metadata.totalDuration); // "2:12.000"
console.log(lyrics.metadata.songParts);     // [{name, time, duration}, ...]
console.log(lyrics.cached);                 // "GDrive" or null
```

### JavaScript — Resolving a Line's Song Section

```javascript
function getSongPart(line, metadata) {
  return metadata.songParts[line.element.songPartIndex] ?? null;
}

// Example
const part = getSongPart(lyrics.lyrics[0], lyrics.metadata);
console.log(part.name); // "Chorus"
console.log(part.time); // 12856
```

### JavaScript — Displaying Synchronized Lyrics

```javascript
function displayLyrics(lyrics, currentTime) {
  const currentLine = lyrics.lyrics.find((line) => {
    const endTime = line.time + line.duration;
    return currentTime >= line.time && currentTime < endTime;
  });

  if (!currentLine) return;

  // For word-level sync, highlight individual words
  if (lyrics.type === "Word" && currentLine.syllabus.length > 0) {
    currentLine.syllabus.forEach((word) => {
      const wordEndTime = word.time + word.duration;
      const isActive = currentTime >= word.time && currentTime < wordEndTime;
      const isBackground = word.isBackground === true;

      console.log({ text: word.text, isActive, isBackground });
    });
  } else {
    // For line-level sync, just display the line
    console.log(currentLine.text);
  }
}
```

### JavaScript — Handling Translation and Transliteration

```javascript
function getDisplayText(line, mode = "original") {
  if (mode === "translation" && line.translation) {
    return line.translation.text;
  }
  if (mode === "transliteration" && line.transliteration) {
    return line.transliteration.text;
  }
  return line.text;
}

function getCurrentWord(line, currentTime, useTransliteration = false) {
  const syllabus =
    useTransliteration && line.transliteration
      ? line.transliteration.syllabus
      : line.syllabus;

  return syllabus.find((word) => {
    const endTime = word.time + word.duration;
    return currentTime >= word.time && currentTime < endTime;
  });
}
```

### Python — Fetching Lyrics

```python
import requests
from typing import Optional, List, Dict, Any
from urllib.parse import urlencode

def get_lyrics(
    title: str,
    artist: str,
    album: Optional[str] = None,
    duration: Optional[int] = None,
    sources: Optional[List[str]] = None
) -> Dict[str, Any]:
    params = {"title": title, "artist": artist}

    if album:
        params["album"] = album
    if duration:
        params["duration"] = str(duration)
    if sources:
        params["source"] = ",".join(sources)

    url = f"https://lyricsplus.prjktla.workers.dev/v2/lyrics/get?{urlencode(params)}"
    response = requests.get(url)
    response.raise_for_status()
    return response.json()

# Example usage
lyrics = get_lyrics(
    title="NOT CUTE ANYMORE",
    artist="ILLIT",
    album="NOT CUTE ANYMORE",
    duration=132,
    sources=["apple", "lyricsplus", "musixmatch", "spotify"]
)

print(f"Type: {lyrics['type']}")
print(f"Duration: {lyrics['metadata']['totalDuration']}")
print(f"Song parts: {lyrics['metadata']['songParts']}")
print(f"Cached: {lyrics['cached']}")
```

### Error Handling with Fallback

```javascript
async function getLyricsWithFallback(title, artist, options) {
  const servers = [
    "https://lyricsplus.prjktla.workers.dev",
    "https://lyrics-plus-backend.vercel.app",
  ];

  for (const server of servers) {
    try {
      const params = new URLSearchParams({ title, artist });
      if (options?.album) params.append("album", options.album);
      if (options?.duration) params.append("duration", options.duration.toString());
      if (options?.sources) params.append("source", options.sources.join(","));

      const response = await fetch(`${server}/v2/lyrics/get?${params}`);

      if (response.ok) {
        return await response.json();
      }
    } catch (error) {
      console.warn(`Failed to fetch from ${server}, trying next...`);
      continue;
    }
  }

  throw new Error("All servers failed to respond");
}
```

---

## Time Format

All timing values are provided in **milliseconds** from the start of the song.

* **Line timing**: `time` and `duration` fields in the lyrics array
* **Word timing**: `time` and `duration` fields in the syllabus array
* **Song parts**: `time` and `duration` in `metadata.songParts`
* **Total duration**: Formatted as `MM:SS.mmm` (minutes:seconds.milliseconds)

### Converting Time Values

```javascript
// Milliseconds to seconds
const seconds = milliseconds / 1000;

// Milliseconds to MM:SS.mmm format
function formatTime(ms) {
  const minutes = Math.floor(ms / 60000);
  const seconds = Math.floor((ms % 60000) / 1000);
  const milliseconds = ms % 1000;
  return `${minutes}:${seconds.toString().padStart(2, "0")}.${milliseconds}`;
}
```

---

## Use Cases

### Karaoke Applications

* **Word-level sync**: Use the word-level timing data in the `syllabus` array to highlight lyrics in real-time as the song plays
* **Background vocals**: Filter out or style differently lyrics with `isBackground: true` for cleaner karaoke display
* **Transliteration karaoke**: Use `transliteration.syllabus` for word-level karaoke display in romanized form

### Lyrics Display

* **Word-level**: Display line-by-line lyrics synchronized with music playback using the `time` and `duration` fields
* **Line-level**: Show complete lines when word-level precision isn't needed
* **Translation**: Show `line.translation.text` alongside original lyrics for bilingual display

### Multi-Language Support

* **Transliteration**: Display romanized text alongside original lyrics for non-Latin script songs (Korean, Japanese, etc.)
* **Translation**: Display human-translated lines using `line.translation.text` (e.g., `lang: "en-US"`)
* Both `translation` and `transliteration` can be present on the same line simultaneously

### Song Structure Navigation

* Use `metadata.songParts` to build a section navigator (jump to Chorus, Verse, Bridge, etc.)
* Resolve a lyric line's section with `metadata.songParts[line.element.songPartIndex]`
* Sections include timing so you can highlight or seek to them directly

### Multi-Artist Tracking

* Track which artist is singing at any given time using the `singer` field and cross-referencing with `metadata.agents`
* Useful for duets or songs with multiple featured artists

### Performance Monitoring

* Use `processingTime.timeElapsed` to monitor API response times
* Use `cached` to distinguish between fresh and cached responses (`"GDrive"` vs `null`)

---

## Error Handling

The API may return errors in the following scenarios:

* **Missing required parameters**: Title or artist not provided
* **No lyrics found**: No matching lyrics in any of the specified sources
* **Invalid source**: Specified source is not supported

Common HTTP status codes:

* `200 OK`: Successful request
* `400 Bad Request`: Missing or invalid parameters
* `404 Not Found`: No lyrics found for the specified song
* `500 Internal Server Error`: Server-side error

---

## Rate Limiting

Please check with the API provider for specific rate limiting policies.

---

## Notes

* URL encode all query parameters
* Multiple artists should be comma-separated
* The API returns either word-level (`type: "Word"`) or line-level (`type: "Line"`) synchronization
* For word-level sync, the `syllabus` array provides word/syllable-level granularity
* For line-level sync, the `syllabus` array will be empty `[]`
* Background vocals are marked with `isBackground: true` in the syllabus array
* Non-Latin script songs may include both `transliteration` (romanized + timed syllabus) and `translation` (human-translated text, no timing) on each line
* `translation` and `transliteration` are independent optional fields and can appear together or separately
* Censored words may appear as `"****"` in the lyrics
* Some agent names and singer fields may be empty strings if not provided by the source
* The `duration` parameter helps improve matching accuracy when searching for songs
* The `metadata.songParts` array is always ordered by `time` ascending
* Resolving a line's song section: `metadata.songParts[line.element.songPartIndex]`
* When source is `lyricsplus`, extra fields `artist`, `album`, and `leadingSilence` are present in metadata
* The `cached` field indicates whether the response was served from cache (`"GDrive"`) or freshly fetched (`null`)
* **Server availability**: If one server is unavailable, try the alternative base URL
* **Best practice**: Implement automatic failover logic to switch between servers if needed

---

## Quick Integration Checklist

1. **Check response type**: Look at `response.type` to determine if you have `"Word"` or `"Line"` sync
2. **Word-level sync**: If `type === "Word"`, iterate through `line.syllabus` for word-by-word timing
3. **Line-level sync**: If `type === "Line"`, `line.syllabus` will be empty — use `line.text` directly
4. **Song section**: Resolve `metadata.songParts[line.element.songPartIndex]` to get the section name and timing
5. **Background vocals**: Check `word.isBackground === true` to style backing vocals differently
6. **Translation**: Check `line.translation` for a human-translated version of the line (no timing data)
7. **Transliteration**: Check `line.transliteration` for romanized text with word-level timing
8. **Timing calculation**: All times are in milliseconds — convert to seconds by dividing by 1000
9. **Cache status**: Check `cached` to know if the response was freshly fetched or served from cache
10. **Error handling**: Implement fallback to alternative server if primary fails
