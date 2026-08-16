# Video Processing Architecture V2

## Overview

The Video Processing Architecture V2 introduces two optimizations to reduce storage usage and processing time.

### 1. CRF-Based Encoding

- Anurag suggested that **we should compress it to reduce storage usage**.

-the new approach uses **CRF (Constant Rate Factor)** for the initial encoding.

**CRF is a quality-based encoding method where a lower CRF generally provides higher quality and a larger file, while a higher CRF provides lower quality and a smaller file.**

* Default CRF: **22**
* CRF is configurable.

```text
Original Video
      ↓
CRF Encoding / Compression
      ↓
Optimized Video
      ↓
Split / Transcoding
```

### 2. Resolution Optimization

While researching the compression approach, I identified another optimization: **reducing the video resolution when the uploaded video is higher than the maximum quality required by the system**.

For example, if the user uploads a **4K video** but the maximum required quality is **720p**:

```text
4K Video
   ↓
CRF 22 + Scale to 720p
   ↓
720p Intermediate Video
   ↓
720p / 480p / 360p
```

There is no need to keep a 4K intermediate video when the final output only requires a maximum of 720p.

The system should also **never upscale** the video. For example, a 480p input should remain 480p even when the maximum configured quality is 720p.

## CRF Encoding Example

For testing, I used a **1 minute 12 second video** with an original file size of **138.9 MiB**.

After encoding the same video with different CRF values, the resulting file sizes were:

|      CRF | Output Size | Size Reduction |
| -------: | ----------: | -------------: |
| Original |   138.9 MiB |              — |
|       18 |    108.7 MB |      **21.7%** |
|       20 |     82.8 MB |      **40.4%** |
|       22 |     66.2 MB |      **52.3%** |
|       23 |     53.6 MB |      **61.4%** |

### Observation

The test shows that increasing the CRF value reduces the output file size.

For example, with the default **CRF 22**, the video size was reduced from **138.9 MiB to 66.2 MB**, which is approximately a **52.3% reduction** in file size.

This demonstrates that CRF encoding can significantly reduce the storage required for the intermediate video before further processing, while providing a configurable balance between video quality and compression.


Overall, V2 first **compresses/optimizes the video using CRF** and also **reduces its resolution when a lower maximum quality is required**, resulting in less storage usage and faster processing.
